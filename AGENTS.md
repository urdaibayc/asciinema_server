# AGENTS.md

Deployment-only repo: a Compose stack (`compose.yml`) for the official asciinema-server image (`ghcr.io/asciinema/asciinema-server:20260626`) plus PostgreSQL 14. There is no application source code here. The git root is this `asciinema_server/` directory; its parent is not a repo.

## Runtime environment (this host)

- This machine IS the deployment host. The stack runs under **rootless podman** with `podman-compose` — use `podman compose ...` (delegates to podman-compose) or `podman-compose ...` directly.
- The docker CLI is installed but has **no compose plugin**: `docker compose ...` fails here. Do not use it, and do not "fix" the docs back to docker.
- `.env` is already populated (`SECRET_KEY_BASE`, `URL_HOST`, `URL_PORT`) — first-run setup is DONE. Never regenerate or change `SECRET_KEY_BASE`; that invalidates sessions and encrypted data.
- Containers are named `asciinema_server_asciinema_1` / `asciinema_server_postgres_1` (project name comes from the directory name). `podman logs <name>` works even when the stack is down.

## Commands

- Validate: `podman compose config`
- Start: `podman compose up -d`
- Logs: `podman compose logs -f asciinema`
- Stop (data persists): `podman compose down`
- Login links (no SMTP configured, so they are printed to logs instead of emailed): `podman compose logs asciinema | grep http`

## Setup (fresh host only)

`.env.example` was intentionally removed — create `.env` (gitignored, never commit) by hand with exactly these keys:

- `SECRET_KEY_BASE=` — 64 alphanumeric chars: `tr -dc A-Za-z0-9 </dev/urandom | head -c 64 ; echo`
- `URL_HOST=` — IP/DNS clients will use to reach the server
- `URL_PORT=8080`

## Gotchas

- Data lives in host bind mounts `/srv/asciinema/asciinema` (recordings) and `/srv/asciinema/postgres` (DB). Never delete or reset them; back them up to preserve recordings.
- Those dirs were created by an earlier **rootful docker** run and are root/system-owned, which rootless podman cannot `chown`: the postgres container crash-loops with `chown: Operation not permitted`, and asciinema then fails migrations (DB unreachable). `podman unshare chown` can NOT fix this (no power over files owned by unmapped real root) — it needs sudo. Both dirs are currently EMPTY (the stack never initialized), so the fix is: `sudo chown -R $USER:$USER /srv/asciinema`, then `podman compose up -d`.
- podman-compose quirk: `up` does not recreate existing containers. If crashed/stale containers exist, `up` fails with `Error: ... name is already in use` (exit 125), then silently `podman start`s the old broken ones and prints exit 0 — a hollow success. Run `podman compose down` first whenever containers already exist.
- No `restart:` policy in `compose.yml` — crashed or stopped containers stay down until `podman compose up -d`. (Linger is enabled for the user, so rootless containers can run without an active login session.)
- Host port `8080` maps to container port `4000`; `URL_PORT` must stay `8080`. `URL_SCHEME` is hardcoded to `http` — HTTPS would need a reverse proxy plus `URL_SCHEME=https`.
- Postgres uses `POSTGRES_HOST_AUTH_METHOD=trust` (no password); it is only reachable inside the compose network — never publish its port.
- Bump the pinned image tag (`20260626`) deliberately, only after checking [asciinema-server releases](https://github.com/asciinema/asciinema-server/releases).
- `TZ=UTC` is hardcoded in `compose.yml` (not overridable via `.env`); timestamps are in UTC.
