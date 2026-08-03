# asciinema-server

Self-hosted [asciinema-server](https://github.com/asciinema/asciinema-server) running as a rootless Podman Compose stack: the official image `ghcr.io/asciinema/asciinema-server:20260626` plus a local PostgreSQL 14.

## Requirements

- Podman with `podman-compose` (`podman compose` delegates to it); the stack runs rootless
- A host with space under `/srv/asciinema` for the bind mounts (see "Data & backups")

## Quick start

1. Create `.env` with the keys `SECRET_KEY_BASE`, `URL_HOST` and `URL_PORT=8080`. Generate the secret key:

   ```sh
   tr -dc A-Za-z0-9 </dev/urandom | head -c 64 ; echo
   ```

2. Set `URL_HOST` to the IP/DNS clients will use to reach the server.
3. Start the stack:

   ```sh
   podman compose up -d
   ```

The UI is served at `http://<URL_HOST>:8080`.

## Commands

| Task | Command |
| --- | --- |
| Validate config | `podman compose config` |
| Start | `podman compose up -d` |
| Stream logs | `podman compose logs -f asciinema` |
| Stop (data persists) | `podman compose down` |

## Login

No SMTP is configured, so sign-in links are printed to the container logs instead of emailed:

```sh
podman compose logs asciinema | grep http
```

## Data & backups

- Recordings: `/srv/asciinema/asciinema`
- Database: `/srv/asciinema/postgres`

Both are host bind mounts. Never delete or reset them; back them up to preserve recordings.

Note: these dirs were created by an earlier rootful Docker run and are root/system-owned, which rootless Podman cannot `chown` — the postgres container fails with `chown: Operation not permitted`. `podman unshare chown` cannot fix this (no power over files owned by unmapped real root); it needs sudo. The dirs are currently empty (the stack never initialized), so: `sudo chown -R $USER:$USER /srv/asciinema`, then `podman compose up -d`.

## Upgrade

`SECRET_KEY_BASE` must never change after first boot — it invalidates sessions and encrypted data.

The image tag is pinned in `compose.yml`. To upgrade:

1. Back up `/srv/asciinema`.
2. `podman compose down`.
3. Bump the tag in `compose.yml` (check [asciinema-server releases](https://github.com/asciinema/asciinema-server/releases) first), then `podman compose up -d`.

## HTTPS

Not configured. The stack serves plain HTTP on port 8080 (`URL_SCHEME` is hardcoded to `http`). For TLS, put a reverse proxy in front and set `URL_SCHEME=https`.

## Notes

- There is no `restart:` policy in `compose.yml` — after a crash, `podman compose down`, or a host reboot, start the stack again with `podman compose up -d`.
- podman-compose does not recreate existing containers: if crashed or stale ones exist, `up` fails with "name already in use" and then just starts the old ones. Run `podman compose down` first whenever containers already exist.
- Postgres uses trust auth (no password) and is only reachable inside the compose network — never publish its port.
- Host port `8080` maps to the container's port `4000`; `URL_PORT` must stay `8080`.
- `TZ=UTC` is fixed; timestamps are in UTC.
