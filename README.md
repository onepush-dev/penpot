# Penpot — open-source Figma alternative for design and prototyping (v2.18.3, frontend + backend + exporter + Postgres + Valkey). Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/iUQla_)

[Penpot](https://penpot.app) is the open-source design and prototyping platform from [penpot/penpot](https://github.com/penpot/penpot) (MPL-2.0). It runs in the browser and is built on open standards (SVG, CSS, HTML). You get vector design, components and variants, design tokens, flex and grid layout, interactive prototypes, real-time multiplayer editing, plugins, and an inspect mode that gives developers CSS, SVG and HTML. This template runs the **official** `penpotapp/frontend`, `penpotapp/backend` and `penpotapp/exporter` images at **2.18.3**, each pinned by tag and digest, laid out like upstream's `docker/images/docker-compose.yaml`. It needs **no deploy-time input**: open the URL and create your account.

## Why this template
- **Pinned, current release.** `2.18.3` (released 2026-10-06) is the latest stable release, pinned by tag **and** digest on all three Penpot images, Valkey and Postgres. A redeploy never jumps versions or runs migrations you didn't plan for.
- **No bucket, no MinIO, one volume.** Upstream's compose file shares one `assets` disk between nginx (frontend) and the backend. Railway volumes attach to a single service, so most Penpot templates switch to S3 (MinIO you have to set up by hand, or a bucket). This one keeps upstream's filesystem storage: the backend owns the volume at `/opt/data/assets` and serves it read-only over the private network, and the frontend's nginx fetches files from there instead of a local disk. Uploads, fonts, thumbnails and exports persist with no extra service.
- **Survives redeploys.** nginx upstreams use `server <private-domain> resolve`, so the web app re-resolves the backend and exporter while it runs. Redeploying the backend never leaves the frontend pointing at a stale private IP.
- **Exports work out of the box.** The exporter's headless Chromium renders PNG, JPEG, SVG and PDF from the web app over the private network (`PENPOT_INTERNAL_URI`).
- **Generated secrets.** `PENPOT_SECRET_KEY` (sessions, invitations, the exporter's shared key) and the Valkey and Postgres passwords are generated per deploy. Upstream's compose file ships a publicly known `change-this-insecure-key`.
- **Private by default.** Only `penpot-frontend` has a public domain. The backend, exporter, Postgres and Valkey talk over Railway's private network, and Valkey requires a password.

## Services
| Service | Source | Port | Volume | Notes |
|---|---|---|---|---|
| penpot-frontend | `penpotapp/frontend:2.18.3@sha256:bb8abe27d53de84c95597f2c02c0e702b2779971fb0703e543f9ecf183e999f6` | `8080` (public) | none | nginx: static web app, proxies `/api`, `/ws/notifications`, `/assets`, `/readyz` to the backend and `/api/export` to the exporter. Health `/` |
| penpot-backend | `penpotapp/backend:2.18.3@sha256:2df1b3440d2a82cc3571db211b4ffdfa2b89ccc910759e8d5e9387fb62971b5c` | `6060` (private), `6070` (private, assets) | `/opt/data/assets` | API, migrations on boot, background workers. Health `/readyz` (timeout 600 s) |
| penpot-exporter | `penpotapp/exporter:2.18.3@sha256:418232d6ca3120b1c2bfde298a56a05a1f41f567cd8494deac3fe7fbc186cfbd` | `6061` (private) | none | headless Chromium exports. Health `/readyz` |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl:18.6@sha256:fcc6686a3f4f1962fe4ed77a15884553b7bd611ae1cea3204687968c5c09d517` | `5432` (private) | `/var/lib/postgresql/data` | Railway's Postgres template. Holds every design file |
| Valkey | `valkey/valkey:8.1.10@sha256:640c5e62cea04b6d6f2084232651d0cc70362d31f4f805e7be94dbed6855e8f2` | `6379` (private) | none | real-time notifications (pub/sub). Same `8.1` line and memory flags as upstream |

Upstream's optional `penpot-admin-console`, `penpot-mcp` and `mailcatch` containers are not included, and the `enable-admin-console` / `enable-mcp` flags stay off.

### Start commands
Railway replaces an image's ENTRYPOINT when a start command is set, so the commands call the upstream entrypoint themselves. They are also in `railway/*-start-command.txt`.

**penpot-frontend.** It writes two nginx override files (Penpot's image includes `/etc/nginx/overrides/{http,server}.d/*.conf`), then runs upstream's `/entrypoint.sh`:
```
/bin/bash -c 'printf "upstream penpot-backend { zone penpot_backend 64k; server %s:6060 resolve; }\nupstream penpot-exporter { zone penpot_exporter 64k; server %s:6061 resolve; }\nupstream penpot-assets { zone penpot_assets 64k; server %s:6070 resolve; }\n" "$PENPOT_BACKEND_HOST" "$PENPOT_EXPORTER_HOST" "$PENPOT_BACKEND_HOST" > /etc/nginx/overrides/http.d/railway-upstreams.conf && printf "location ~ ^/internal/assets/(.*)\$ { internal; proxy_pass http://penpot-assets/\$1; proxy_hide_header Content-Type; include /etc/nginx/nginx-security-headers.conf; }\n" > /etc/nginx/overrides/server.d/railway-assets.conf && exec /bin/bash /entrypoint.sh nginx -g "daemon off;"'
```
**penpot-backend.** It starts a read-only static file server for the assets volume on private port 6070 (`python3` ships in the image; 6063 is Penpot's prepl port), then runs the normal backend:
```
/bin/bash -c 'python3 -m http.server 6070 --bind :: --directory /opt/data/assets >/dev/null 2>&1 & exec /bin/bash run.sh'
```
**penpot-exporter**, **Valkey** and **Postgres:** none (image defaults). Valkey's entrypoint appends `$VALKEY_EXTRA_FLAGS`, which carries `--requirepass`.

### How assets are served
1. The browser requests `/assets/by-id/<id>` from the frontend. nginx proxies it to the backend.
2. The backend checks permissions (non-public buckets such as profile photos need a session) and answers `204` with `X-Accel-Redirect: /internal/assets/<path>`.
3. Upstream nginx would read that path from a shared disk. Here, the override location fetches `http://penpot-assets/<path>` (the backend's port 6070) over the private network instead, and keeps the backend's `Content-Type`.

`/internal/assets` is `internal` in nginx, so it can't be requested from outside. Port 6070 is reachable only inside the project's private network.

## Variables
Every value is also in `env.example` (and `railway/template-variables.env`).

### penpot-frontend
| Variable | Template value | Notes |
|---|---|---|
| PENPOT_PUBLIC_URI | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | public URL. The backend and exporter reference it, so a custom domain is a one-place change |
| PENPOT_FLAGS | `${{penpot-backend.PENPOT_FLAGS}}` | written into `js/config.js` at start |
| PENPOT_BACKEND_URI | `http://penpot-backend` | nginx upstream defined by the start command; leave as is |
| PENPOT_EXPORTER_URI | `http://penpot-exporter` | nginx upstream defined by the start command; leave as is |
| PENPOT_BACKEND_HOST | `${{penpot-backend.RAILWAY_PRIVATE_DOMAIN}}` | backend private hostname (ports 6060 and 6070) |
| PENPOT_EXPORTER_HOST | `${{penpot-exporter.RAILWAY_PRIVATE_DOMAIN}}` | exporter private hostname (port 6061) |
| PENPOT_HTTP_SERVER_MAX_BODY_SIZE | `367001600` | nginx `client_max_body_size` (350 MiB, upstream default) |
| PORT | `8080` | port the public domain routes to |

### penpot-backend
| Variable | Template value | Notes |
|---|---|---|
| PENPOT_FLAGS | `enable-registration enable-login-with-password disable-email-verification enable-prepl-server` | sign-up with email + password, no SMTP needed. `prepl-server` enables `manage.py` inside the container |
| PENPOT_PUBLIC_URI | `${{penpot-frontend.PENPOT_PUBLIC_URI}}` | |
| PENPOT_SECRET_KEY | `${{secret(64, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | master key; changing it logs everyone out |
| PENPOT_DATABASE_URI | `postgresql://${{Postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/${{Postgres.PGDATABASE}}` | |
| PENPOT_DATABASE_USERNAME | `${{Postgres.PGUSER}}` | |
| PENPOT_DATABASE_PASSWORD | `${{Postgres.PGPASSWORD}}` | |
| PENPOT_REDIS_URI | `redis://default:${{Valkey.VALKEY_PASSWORD}}@${{Valkey.RAILWAY_PRIVATE_DOMAIN}}:6379/0` | |
| PENPOT_OBJECTS_STORAGE_BACKEND | `fs` | filesystem storage on the volume |
| PENPOT_OBJECTS_STORAGE_FS_DIRECTORY | `/opt/data/assets` | volume mount path |
| PENPOT_HTTP_SERVER_HOST | `::` | listen on IPv6 + IPv4 for Railway's private network |
| PENPOT_HTTP_SERVER_PORT | `6060` | |
| PENPOT_HTTP_SERVER_MAX_BODY_SIZE | `367001600` | |
| PENPOT_HTTP_SERVER_MAX_MULTIPART_BODY_SIZE | `367001600` | deprecated upstream alias, kept as in compose |
| PORT | `6060` | used by the `/readyz` healthcheck |
| JAVA_OPTS | `-Xms256m -Xmx1g` | JVM heap cap. Without it the JVM sizes its heap from the host's memory |
| RAILWAY_RUN_UID | `0` | Railway volumes mount root-owned, and the image runs as uid 1001 |

### penpot-exporter
| Variable | Template value |
|---|---|
| PENPOT_FLAGS | `${{penpot-backend.PENPOT_FLAGS}}` |
| PENPOT_PUBLIC_URI | `${{penpot-frontend.PENPOT_PUBLIC_URI}}` |
| PENPOT_SECRET_KEY | `${{penpot-backend.PENPOT_SECRET_KEY}}` |
| PENPOT_REDIS_URI | `${{penpot-backend.PENPOT_REDIS_URI}}` |
| PENPOT_INTERNAL_URI | `http://${{penpot-frontend.RAILWAY_PRIVATE_DOMAIN}}:8080` |
| PENPOT_HTTP_SERVER_PORT | `6061` |
| PORT | `6061` |

### Valkey
| Variable | Template value |
|---|---|
| VALKEY_PASSWORD | `${{secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` |
| VALKEY_EXTRA_FLAGS | `--requirepass ${{VALKEY_PASSWORD}} --maxmemory 128mb --maxmemory-policy volatile-lfu` |
| REDISHOST / REDISPORT / REDISUSER | `${{RAILWAY_PRIVATE_DOMAIN}}` / `6379` / `default` |
| REDIS_URL | `redis://default:${{VALKEY_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:6379` |

### Postgres
These are Railway's Postgres template defaults: `POSTGRES_USER=postgres`, `POSTGRES_PASSWORD=${{secret(32, ...)}}`, `POSTGRES_DB=railway`, `PGDATA=/var/lib/postgresql/data/pgdata`, `PGHOST/PGPORT/PGUSER/PGPASSWORD/PGDATABASE`, `DATABASE_URL`, `SSL_CERT_DAYS=820` and `RAILWAY_DEPLOYMENT_DRAINING_SECONDS=60`.

## After deploy
1. Wait for `penpot-backend` to go healthy. The first boot runs the database migrations, which takes about a minute after the image pull.
2. Open `https://<your-domain>` → **Create an account** → name, email, password. There's no email verification, so you're in right away.
3. **Close registration** once your team has signed up. Add `disable-registration` to `PENPOT_FLAGS` on **penpot-backend** (the frontend and exporter reference it), then redeploy all three Penpot services. Without SMTP, invitation emails aren't sent. Add `enable-log-invitation-tokens` to `PENPOT_FLAGS` and copy invite links from the `penpot-backend` logs, or set up SMTP.

Optional, on `penpot-backend`:
- **Email:** `PENPOT_SMTP_HOST`, `PENPOT_SMTP_PORT`, `PENPOT_SMTP_USERNAME`, `PENPOT_SMTP_PASSWORD`, `PENPOT_SMTP_TLS`/`PENPOT_SMTP_SSL`, `PENPOT_SMTP_DEFAULT_FROM`, then add `enable-smtp` to `PENPOT_FLAGS` (and drop `disable-email-verification` if you want verification). Railway allows outbound SMTP only on the Pro plan and above.
- **SSO:** `enable-login-with-google|github|gitlab|oidc|ldap` plus the matching `PENPOT_GOOGLE_*`, `PENPOT_GITHUB_*`, `PENPOT_OIDC_*` and `PENPOT_LDAP_*` variables ([configuration guide](https://help.penpot.app/technical-guide/configuration/)).
- **Custom domain:** add it to `penpot-frontend` and set `PENPOT_PUBLIC_URI=https://your.domain` there.

## Notes
- **Back up both stores.** Postgres holds every design file. The `penpot-backend` volume holds uploaded images, fonts, thumbnails and exports.
- **Upgrades:** change the tag **and** digest on `penpot-frontend`, `penpot-backend` and `penpot-exporter` together. The backend migrates on boot. Read the [release notes](https://github.com/penpot/penpot/releases) and back up Postgres first.
- **One backend replica.** The assets volume ties the backend to one replica in one region. That's fine for teams. For horizontal scaling, switch to S3 storage (`PENPOT_OBJECTS_STORAGE_BACKEND=s3`, for example with a Railway Bucket).
- **RAM:** expect about 1.5 GB at idle for the stack. The JVM backend (heap capped at 1 GB) uses most of it, and the exporter grows by a few hundred MB while Chromium renders. That needs at least the Hobby plan.
- **Logs:** the backend and exporter print the Valkey URI, password included, at startup (upstream behavior). Keep project log access restricted.
- Upstream: https://github.com/penpot/penpot · compose at 2.18.3: https://github.com/penpot/penpot/blob/2.18.3/docker/images/docker-compose.yaml · Docs: https://help.penpot.app/technical-guide/
