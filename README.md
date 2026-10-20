# Nyuppsättning Deploy

Docker Compose setup for running Nyuppsättning (server, web, MySQL) on a single host, published through a Cloudflare
Tunnel. It follows [spexregister-deploy][1]'s layout and runs next to it. The host needs no open inbound ports and
no reverse proxy: `cloudflared` dials out to Cloudflare, which terminates TLS and forwards each public hostname to a
container.

Keycloak runs from [fgv-infra-deploy][2] on the same host, and Nyuppsättning reads spex from Spexregister. Set
up both first.

```
Browser ──https──▶ Cloudflare ──tunnel──▶ cloudflared ─┬─▶ revival-web:3000   (WEB_HOST)
                                                       └─▶ revival-api:8080   (API_HOST, MCP)

revival-web ──http──▶ revival-api:8080 ──▶ revival-mysql
                      revival-api ──http──▶ keycloak-server:8080    (fgv-infra-deploy)
                      revival-api ──http──▶ spexregister-api:8080   (spexregister-deploy)
```

The web talks to the API over this project's default network. The API also joins the network `fgv-<DEPLOY_ENV>`,
which `fgv-infra-deploy` creates. There it reaches Keycloak's signing keys as `keycloak-server:8080` and
Spexregister's GraphQL API as `spexregister-api:8080`, with the `fgv-revival-service` client and its `REVIVAL` role.
Browsers and the web reach Keycloak at its public `AUTH_HOST`. The API hostname is needed only for MCP clients.

Every service is named after its application and role, as in `fgv-infra-deploy` and `spexregister-deploy`, so
names on the shared network say where they come from.

## Files

| File                          | Purpose                                                                    |
|-------------------------------|----------------------------------------------------------------------------|
| `compose.yaml`                | `revival-mysql`, `revival-api`, `revival-web`, configured entirely from `.env` |
| `compose.tunnel.yaml`         | `cloudflared` connector, for hosts published through a Cloudflare Tunnel   |
| `staging.env.example`         | Template for `.env` on the staging host                                    |
| `production.env.example`      | Template for `.env` on the production host                                 |
| `bin/init-secrets.sh`         | Generates random secrets into `secrets/` and copies the client secrets from `fgv-infra-deploy` (never overwrites existing ones) |
| `bin/backup-db.sh`            | Dumps the database into `backups/`, pruning dumps older than 14 days       |

`COMPOSE_FILE` in `.env` selects the overlays, so plain `docker compose ...` commands work on each host.

| `.env` variable  | Image                          | Source repo            |
|------------------|--------------------------------|------------------------|
| `API_IMAGE`      | `bloomper/fgv-revival-server`  | `fgv-revival-server`   |
| `WEB_IMAGE`      | `bloomper/fgv-revival-web`     | `fgv-revival-web`      |

## 1. Prerequisites

On the host:

- `fgv-infra-deploy` running, with a realm created from an `fgv-keycloak` image that has the Nyuppsättning
  clients. Its `.env` names this deployment's `WEB_HOST` and `API_HOST` as `FGV_REVIVAL_WEB_HOST` and
  `FGV_REVIVAL_API_HOST`, because Keycloak only sends users back to that address.
- `spexregister-deploy` running a `spexregister-server` release that knows the `REVIVAL` role (feature
  `002-spexregister-access-policy`). Without it, Nyuppsättning shows spex as unavailable.

## 2. Cloudflare

1. Networking → Tunnels → *Create a tunnel* → Cloudflared. Name it e.g. `fgv-revival-staging`.
2. Copy the token from the install command (the long string after `--token`).
3. On the tunnel's **Routes** tab, select *Add route* → *Published application* once for each row below.

   | Hostname     | Service URL                |
   |--------------|----------------------------|
   | `WEB_HOST`   | `http://revival-web:3000`  |
   | `API_HOST`   | `http://revival-api:8080`  |

4. On staging, restrict `WEB_HOST` to testers with Cloudflare Access, as `spexregister-deploy`'s README describes.
   Leave `API_HOST` unprotected: MCP clients can't complete an Access login, and the API already requires a
   Keycloak token.

## 3. Checkout and configuration

Check this repository out next to `fgv-infra-deploy`:

```shell
git clone https://github.com/bloomper/fgv-revival-deploy.git
cd fgv-revival-deploy

cp staging.env.example .env               # set the hostnames
bin/init-secrets.sh                       # copies the client secrets from ../fgv-infra-deploy
nano secrets/cloudflared.env              # TUNNEL_TOKEN=<token from step 2>
mkdir -p data/api/log_data
sudo chown -R 1000:1000 data/api
```

`bin/init-secrets.sh` copies the `fgv-revival` and `fgv-revival-service` client secrets from
`../fgv-infra-deploy/secrets/clients/`. Set `INFRA_DEPLOY` to its path if it is checked out elsewhere. If
`fgv-infra-deploy` got those secrets only now, run its `bin/configure-keycloak.sh` too, so that the realm has them.

| Variable                   | Purpose                                                              |
|----------------------------|----------------------------------------------------------------------|
| `WEB_HOST`                 | Public address of the web                                            |
| `API_HOST`                 | Public address of the API, for MCP clients                           |
| `AUTH_HOST`, `AUTH_REALM`  | Keycloak's public address and realm, as in `fgv-infra-deploy`        |
| `SPEXREGISTER_GRAPHQL_URL` | Spexregister's GraphQL endpoint, `http://spexregister-api:8080/api/graphql` on the shared network |
| `MYSQL_DATABASE`, `MYSQL_USER` | The application database                                         |
| `SUPPORTED_LOCALES`, `DEFAULT_LOCALE` | `sv,en` and `sv`                                          |

## 4. First start

```shell
docker compose up -d
docker compose ps                         # wait for "healthy"
```

The API creates its tables on first start.

### The first super admin

Access is granted in Keycloak by role alone, never by FGV membership. In the admin console of
`fgv-infra-deploy` (`https://AUTH_HOST/auth/admin`), open *Users*, pick the person and, under *Role mapping →
Assign role → Client roles*, give them `fgv-revival` `USER` (may log in) and `SUPER_ADMIN`. They get an email that
they now have access to Nyuppsättning, with its address. From then on super admins appoint directeurs and steering
groups in the application; only `USER`, `SUPER_ADMIN` and `OBSERVER` are given in Keycloak.

## Updating

```shell
git pull
docker compose pull
docker compose up -d --remove-orphans
```

Containers whose image changed are recreated. Database migrations run when the API starts. When an update adds a
secret or a variable, compare `.env` with the example file, then run `bin/init-secrets.sh` again. It doesn't
overwrite existing secrets.

## Data and backups

| Location                    | Contents                                   | Backup                                  |
|-----------------------------|--------------------------------------------|-----------------------------------------|
| volume `mysql_data`         | Application database                       | `bin/backup-db.sh`                      |
| `data/api/log_data`         | Application logs                           | not needed                              |

`bin/backup-db.sh` uses `mysqldump --single-transaction`, which is consistent without stopping anything. Run it
nightly from the host user's crontab (`crontab -e`):

```
40 2 * * * /home/<user>/fgv-revival-deploy/bin/backup-db.sh >> /home/<user>/fgv-revival-deploy/backups/backup.log 2>&1
```

`BACKUP_KEEP_DAYS` sets how many days of dumps to keep (default 14). Copy `backups/` off the host as well, e.g.
with `rclone` or `rsync`; a backup on the same disk doesn't survive losing the disk.

To restore a dump:

```shell
gunzip -c backups/<timestamp>/revival-mysql.sql.gz | docker compose exec -T revival-mysql \
  sh -c 'MYSQL_PWD=$(cat /run/secrets/MYSQL_ROOT_PASSWORD) mysql -uroot "$MYSQL_DATABASE"'
```

Apart from the API's own files in `data/api/log_data`, every service logs to the console only; read it with
`docker compose logs -f <service>`. Each service keeps at most 5 × 10 MB of console output.

## Resetting staging data

```shell
docker compose down -v                    # removes the database volume
rm -rf data/api                           # logs
```

Then repeat *First start*. Users and their `fgv-revival` roles live in Keycloak and are kept.

## Production

Staging and production use the same `compose.yaml`; they differ only in `.env`:

- `.env` copied from `production.env.example`, with `DEPLOY_ENV=production`
- `bin/backup-db.sh` scheduled in cron, with `backups/` copied to off-host storage
- `API_IMAGE` and `WEB_IMAGE` pinned to release versions, e.g. `bloomper/fgv-revival-server:0.1.0`, instead of
  `develop`, and `CLOUDFLARED_VERSION` pinned instead of `latest`, so that an update happens only when `.env` changes

Add `compose.tunnel.yaml` to `COMPOSE_FILE` if production goes through a Cloudflare Tunnel, using its own tunnel and
token. Leave it out if production sits behind its own reverse proxy instead.

## Troubleshooting

| Symptom                                           | Cause                                                       |
|---------------------------------------------------|-------------------------------------------------------------|
| `network fgv-<DEPLOY_ENV> declared as external, but could not be found` | `fgv-infra-deploy` isn't running, or uses another `DEPLOY_ENV` |
| Login ends with "Invalid parameter: redirect_uri" | `FGV_REVIVAL_WEB_HOST` in `fgv-infra-deploy` doesn't match `WEB_HOST`, or the realm lacks the clients |
| Login fails with `invalid_client` in the web's log | `secrets/AUTH_CLIENT_SECRET` differs from the realm; run `fgv-infra-deploy/bin/configure-keycloak.sh` |
| Spexes show as unavailable                        | Spexregister is down, too old for `REVIVAL`, or `secrets/AUTH_SERVICE_CLIENT_SECRET` differs from the realm |
| A user gets "no access"                           | They lack `fgv-revival` `USER` in Keycloak                  |

## Image tags

CI in `fgv-revival-server` and `fgv-revival-web` publishes each image with these tags:

| Trigger              | Tags                                   |
|----------------------|----------------------------------------|
| push to `develop`    | `develop`, `sha-<commit>`              |
| push of tag `vX.Y.Z` | `X.Y.Z`, `latest`, `sha-<commit>`      |

To release, set the version in that repo (`build.gradle` in `fgv-revival-server`, `package.json` in
`fgv-revival-web`) to `X.Y.Z`, merge to `main`, then tag and push `vX.Y.Z`. The build fails if the tag doesn't match
the version or if `X.Y.Z` is already on Docker Hub.

[1]: https://github.com/bloomper/spexregister-deploy
[2]: https://github.com/bloomper/fgv-infra-deploy
