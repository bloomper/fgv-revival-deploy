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
revival-web ──http──▶ umami-server:3000                             (fgv-infra-deploy)
```

The web talks to the API over this project's default network. The API also joins the network `fgv-<DEPLOY_ENV>`,
which `fgv-infra-deploy` creates. There it reaches Keycloak's signing keys as `keycloak-server:8080` and
Spexregister's GraphQL API as `spexregister-api:8080`, with the `fgv-revival-service` client and its `REVIVAL` role.
The web joins it too and forwards its visitor statistics to Umami as `umami-server:3000`.
Browsers and the web reach Keycloak at its public `KEYCLOAK_HOST`. The API hostname is needed only for MCP clients and `/docs`.

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

Every service runs from a published image, so the host needs only this repo:

| `.env` variable  | Image                          | Source repo            |
|------------------|--------------------------------|------------------------|
| `API_IMAGE`      | `bloomper/fgv-revival-server`  | `fgv-revival-server`   |
| `WEB_IMAGE`      | `bloomper/fgv-revival-web`     | `fgv-revival-web`      |

## 1. Host

Set up the host and Keycloak as [fgv-infra-deploy][2] describes, and Spexregister as [spexregister-deploy][1]
describes. Both must be running before the first start below:

- `fgv-infra-deploy` running, with a realm created from an `fgv-keycloak` image that has the Nyuppsättning
  clients. Its `.env` names this deployment's `WEB_HOST` and `API_HOST` as `FGV_REVIVAL_WEB_HOST` and
  `FGV_REVIVAL_API_HOST`, because Keycloak only sends users back to that address.
- `spexregister-deploy` running a `spexregister-server` release that knows the `REVIVAL` role. Without it,
  Nyuppsättning shows spex as unavailable.

## 2. Cloudflare

The domain's DNS must be hosted on Cloudflare. Use single-level subdomains such as
`revival-staging.example.org`. A name like `api.revival.example.org` is two levels deep, and the free
Universal SSL certificate doesn't cover it.

1. Networking → Tunnels → *Create a tunnel* → Cloudflared. Name it e.g. `fgv-revival-staging`.
   Keycloak has its own tunnel.
2. Copy the token from the install command (the long string after `--token`).
3. On the tunnel's **Routes** tab, select *Add route* → *Published application* once for each row below.
   Enter the hostname as subdomain plus domain, and the service in **Service URL**. Cloudflare creates the
   DNS records itself.

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
`../fgv-infra-deploy/secrets/clients/`. Only when `.env` has `API_IMPORT_SAMPLE_DATA=true`, it also copies the `fgv-admin`
client secret, and `SAMPLE_USER_PASSWORD` from `../spexregister-deploy/secrets/`. It creates an empty
`secrets/SMTP_PASSWORD`; write the relay's password into it in production. Set `INFRA_DEPLOY` and
`SPEXREGISTER_DEPLOY` to their paths if they are checked out elsewhere. If `fgv-infra-deploy` got those secrets only
now, run its `bin/configure-keycloak.sh` too, so that the realm has them.

| Variable                              | Purpose                                                                                           |
|---------------------------------------|---------------------------------------------------------------------------------------------------|
| `WEB_HOST`                            | Public address of the web                                                                         |
| `API_HOST`                            | Public address of the API, for MCP clients and `/docs`                                            |
| `KEYCLOAK_HOST`, `KEYCLOAK_REALM`     | Keycloak's public address and realm, as in `fgv-infra-deploy`                                     |
| `SPEXREGISTER_GRAPHQL_URL`            | Spexregister's GraphQL endpoint, `http://spexregister-api:8080/api/graphql` on the shared network |
| `API_IMPORT_SAMPLE_DATA`              | `true` on staging: replace all data with sample data at every start, see *First start*             |
| `MYSQL_DATABASE`, `MYSQL_USER`        | The application database                                                                          |
| `SUPPORTED_LOCALES`, `DEFAULT_LOCALE` | `sv,en` and `sv`                                                                                  |
| `SUPPORT_EMAIL`, `FEEDBACK_EMAIL`     | Addresses shown on the Support and About pages                                                    |
| `SMTP_*`, `MAIL_FROM`                 | The mail server, see *Mail*                                                                       |
| `UMAMI_WEBSITE_ID`                    | Umami website for `WEB_HOST`; without it the web sends no statistics                              |

For visitor statistics, run `bin/configure-umami.sh` in `fgv-infra-deploy` (see *Umami* in its README). It adds a
website `FGV_REVIVAL_NAME` for `FGV_REVIVAL_WEB_HOST` and prints its ID; put that ID in `UMAMI_WEBSITE_ID`. The web
serves Umami's tracker as `/stats/script.js` and forwards events through `/stats/api/send`, so browsers only talk to
`WEB_HOST`. Leave the ID empty to send nothing.

The `revival-api` container runs as user and group 1000, so it needs to own `data/api`. Create the directory before
the first start: Docker would otherwise create it owned by root, and the container couldn't write to it.

## 4. First start

```shell
docker compose up -d
docker compose ps                         # wait for "healthy"
```

The API creates its tables on first start.

With `API_IMPORT_SAMPLE_DATA=true` the API replaces all data with sample revivals on every start. They are built on
Spexregister's sample data, so `spexregister-deploy` must run with `API_IMPORT_SAMPLE_DATA=true`: the spex come from
Spexregister, and its sample user `admin@fgv.nu` becomes super admin. The import waits for Spexregister and refuses to
run without that user. With the `fgv-admin` client it also creates users with one purpose each, such as
`observer@fgv.nu`, `directeur@fgv.nu` and `steering-group-finance@fgv.nu` (all are listed under
*Test credentials* in `fgv-revival-server`'s getting started guide), with the same password as Spexregister's
sample users. Spexregister deletes all Keycloak users on every start, so restart the API after Spexregister:
`docker compose restart revival-api`.

### The first super admin

Access is granted in Keycloak by role alone, never by FGV membership. In the admin console of
`fgv-infra-deploy` (`https://KEYCLOAK_HOST/auth/admin`), open *Users*, pick the person and, under *Role mapping →
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

Keycloak and its realm are updated with `fgv-infra-deploy`.

## Mail

The `revival-api` service sends meeting minutes, mailings and their reminders, invitations to participants and
helpers, and tells the super admins about members waiting for access. Staging sends it to Mailpit, run by
`fgv-infra-deploy` as `mailpit` on the shared network. For a real relay, set these in `.env` and restart
`revival-api`. Keycloak's own mail settings are in `fgv-infra-deploy`:

| Variable        | Purpose                                                                  |
|-----------------|--------------------------------------------------------------------------|
| `SMTP_HOST`     | SMTP server                                                              |
| `SMTP_PORT`     | Usually `587` with STARTTLS, `465` with implicit TLS, `25` without       |
| `SMTP_AUTH`     | `true` to log in with `SMTP_USER` and the password in `secrets/SMTP_PASSWORD` |
| `SMTP_USER`     | SMTP user                                                                |
| `SMTP_STARTTLS` | `true` to upgrade the connection with STARTTLS                           |
| `SMTP_SSL`      | `true` to connect with implicit TLS                                      |
| `MAIL_FROM`     | Sender address                                                           |

`bin/init-secrets.sh` creates `secrets/SMTP_PASSWORD` empty. Write the password into it without a trailing
newline, e.g. `printf '%s' '<password>' > secrets/SMTP_PASSWORD`.

## Data and backups

| Location                    | Contents                                   | Backup                                  |
|-----------------------------|--------------------------------------------|-----------------------------------------|
| volume `mysql_data`         | Application database                       | `bin/backup-db.sh`                      |
| `data/api/log_data`         | Application logs                           | not needed                              |

The database stays in a named volume: copying the files of a running MySQL server doesn't give a consistent
backup. `bin/backup-db.sh` uses `mysqldump --single-transaction` instead, which is consistent without stopping
anything. Run it nightly from the host user's crontab (`crontab -e`):

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
`docker compose logs -f <service>`. Each service keeps at most 5 × 10 MB of console output (`x-logging` in the
compose files), so logs can't fill the disk.

## Resetting staging data

```shell
docker compose down -v                    # removes the database volume
rm -rf data/api                           # logs
```

Then repeat *First start*. Users and their `fgv-revival` roles live in Keycloak and are kept. The secrets in
`secrets/` and the dumps in `backups/` are kept as well.

## Production

Staging and production use the same `compose.yaml`; they differ only in `.env`:

- `.env` copied from `production.env.example`: `DEPLOY_ENV=production`, `API_IMPORT_SAMPLE_DATA=false` and a real SMTP
  relay instead of Mailpit (see *Mail*; the values in the example are placeholders)
- no sample users: only the sample data import creates them
- `bin/backup-db.sh` scheduled in cron, with `backups/` copied to off-host storage
- `API_IMAGE` and `WEB_IMAGE` pinned to release versions, e.g. `bloomper/fgv-revival-server:1.0.0`, instead of
  `develop`, and `CLOUDFLARED_VERSION` pinned instead of `latest`, so that an update happens only when `.env` changes

Add `compose.tunnel.yaml` to `COMPOSE_FILE` if production goes through a Cloudflare Tunnel, using its own tunnel and
token. Leave it out if production sits behind its own reverse proxy instead.

## Troubleshooting

| Symptom                                           | Cause                                                       |
|---------------------------------------------------|-------------------------------------------------------------|
| `network fgv-<DEPLOY_ENV> declared as external, but could not be found` | `fgv-infra-deploy` isn't running, or uses another `DEPLOY_ENV` |
| Login ends with "Invalid parameter: redirect_uri" | `FGV_REVIVAL_WEB_HOST` in `fgv-infra-deploy` doesn't match `WEB_HOST`, or the realm lacks the clients |
| Login fails with `invalid_client` in the web's log | `secrets/KEYCLOAK_CLIENT_SECRET` differs from the realm; run `fgv-infra-deploy/bin/configure-keycloak.sh` |
| Spex show as unavailable                          | Spexregister is down, too old for `REVIVAL`, or `secrets/KEYCLOAK_SERVICE_CLIENT_SECRET` differs from the realm |
| A user gets "no access"                           | They lack `fgv-revival` `USER` in Keycloak                  |
| Sample users get "no access" on staging           | Spexregister restarted after the API and replaced the users; restart `revival-api` |

## Image tags

CI in `fgv-revival-server` and `fgv-revival-web` publishes each image with these tags:

| Trigger              | Tags                                   |
|----------------------|----------------------------------------|
| push to `develop`    | `develop`, `sha-<commit>`              |
| push of tag `vX.Y.Z` | `X.Y.Z`, `latest`, `sha-<commit>`      |

To release, set the version in that repo (`build.gradle` in `fgv-revival-server`, `package.json` in
`fgv-revival-web`) to `X.Y.Z`, merge to `main`, then tag and push:

```shell
git tag vX.Y.Z && git push origin vX.Y.Z
```

The build fails if the tag doesn't match the version or if `X.Y.Z` is already on Docker Hub, so release tags
are never overwritten.

[1]: https://github.com/bloomper/spexregister-deploy
[2]: https://github.com/bloomper/fgv-infra-deploy
