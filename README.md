# asciinema-server

Self-hosted [asciinema-server](https://github.com/asciinema/asciinema-server) running as a Docker Compose stack: the official image `ghcr.io/asciinema/asciinema-server:20260626` plus a local PostgreSQL 14.

## Requirements

- Docker with the Compose plugin (`docker compose`)
- A host with space under `/srv/asciinema` (created automatically on first boot, root-owned)

## Quick start

1. `cp .env.example .env`
2. Generate a secret key and set it as `SECRET_KEY_BASE` in `.env`:

   ```sh
   tr -dc A-Za-z0-9 </dev/urandom | head -c 64 ; echo
   ```

3. Set `URL_HOST` to the IP/DNS clients will use to reach the server (the placeholder `192.168.1.10` must be changed).
4. Start the stack:

   ```sh
   docker compose up -d
   ```

The UI is served at `http://<URL_HOST>:8080`.

## Commands

| Task | Command |
| --- | --- |
| Validate config | `docker compose config` |
| Start | `docker compose up -d` |
| Stream logs | `docker compose logs -f asciinema` |
| Stop (data persists) | `docker compose down` |

## Login

No SMTP is configured, so sign-in links are printed to the container logs instead of emailed:

```sh
docker compose logs asciinema | grep http
```

## Data & backups

- Recordings: `/srv/asciinema/asciinema`
- Database: `/srv/asciinema/postgres`

Both are host bind mounts created on first boot and are root-owned. Never delete or reset them; back them up to preserve recordings.

## Upgrade

`SECRET_KEY_BASE` must never change after first boot — it invalidates sessions and encrypted data.

The image tag is pinned in `compose.yml`. To upgrade:

1. Back up `/srv/asciinema`.
2. `docker compose down`.
3. Bump the tag in `compose.yml` (check [asciinema-server releases](https://github.com/asciinema/asciinema-server/releases) first), then `docker compose up -d`.

## HTTPS

Not configured. The stack serves plain HTTP on port 8080 (`URL_SCHEME` is hardcoded to `http`). For TLS, put a reverse proxy in front and set `URL_SCHEME=https`.

## Notes

- Postgres uses trust auth (no password) and is only reachable inside the compose network — never publish its port.
- Host port `8080` maps to the container's port `4000`; `URL_PORT` must stay `8080`.
- `TZ=UTC` is fixed; timestamps are in UTC.
