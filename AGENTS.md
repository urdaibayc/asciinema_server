# AGENTS.md

Deployment-only repo: Docker Compose stack for the official asciinema-server container image (`ghcr.io/asciinema/asciinema-server:20260626`) plus a local PostgreSQL 14. There is no application source code here.

## Setup (first run)

1. `cp .env.example .env` — `.env` is gitignored but holds secrets; never share or commit it.
2. Generate `SECRET_KEY_BASE` (64 alphanumeric chars, per official docs):
   `tr -dc A-Za-z0-9 </dev/urandom | head -c 64 ; echo`
3. Set `URL_HOST` (IP/DNS the server is reachable at) and `URL_PORT` (host port, `8080`).

## Commands

- Validate: `docker compose config`
- Start: `docker compose up -d` (postgres waits for its healthcheck first)
- Logs: `docker compose logs -f asciinema`
- Stop (data persists): `docker compose down`
- Note: docker is not installed in the dev environment; run these on the deployment host.

## Gotchas

- `URL_SCHEME` is hardcoded to `http` in `compose.yml` — matches the `:8080` port mapping. HTTPS is not configured; would need a reverse proxy and `URL_SCHEME=https`.
- No SMTP configured, so login links are printed to the asciinema container logs, e.g. `docker compose logs asciinema | grep http`.
- Data lives in host bind mounts `/srv/asciinema/asciinema` (recordings) and `/srv/asciinema/postgres` (DB), created automatically (root-owned). Never delete or reset them; back up to preserve recordings.
- Never change `SECRET_KEY_BASE` after first boot — it invalidates sessions and encrypted data.
- Postgres uses `POSTGRES_HOST_AUTH_METHOD=trust` (no password); only reachable inside the compose network, do not publish its port.
- Bump the pinned image tag (`20260626`) deliberately, only after checking asciinema-server releases.
- Host port `8080` maps to container port `4000`; `URL_PORT` must be `8080` to match.
- `TZ=UTC` is hardcoded in `compose.yml` (not overridable via `.env`); timestamps are in UTC.
- `.env.example` ships a placeholder `URL_HOST=192.168.1.10`; replace it with the real IP/DNS before first boot.
