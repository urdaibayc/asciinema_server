# asciinema-server with Monitoring Stack

Self-hosted [asciinema-server](https://github.com/asciinema/asciinema-server) running as a rootless Podman Compose stack:
- Official asciinema-server image (`ghcr.io/asciinema/asciinema-server:20260626`) 
- Local PostgreSQL 14
- **NEW**: Prometheus + Grafana monitoring stack
- **NEW**: OctoPrint 3D printer support with USB integration

## Requirements

- Podman with `podman-compose` (`podman compose` delegates to it); the stack runs rootless
- A host with space under `/srv/asciinema` for the bind mounts (see "Data & backups")
- **NEW**: At least 2GB RAM recommended for monitoring stack
- **NEW**: USB printer with Tailscale connectivity

## Quick start

1. Create `.env` with the keys `SECRET_KEY_BASE`, `URL_HOST` and `URL_PORT=8080`. Generate the secret key:

   ```sh
   tr -dc A-Za-z0-9 </dev/urandom | head -c 64 ; echo
   ```

2. Set `URL_HOST` to the IP/DNS clients will use to reach the server.
3. Set `GRAFANA_PASSWORD` for admin access:
   ```sh
   export GRAFANA_PASSWORD=your-secure-password
   ```
4. Start the stack:

   ```sh
   podman compose up -d
   ```

The UI is served at:
- Asciinema: `http://<URL_HOST>:8080`
- Grafana: `http://<URL_HOST>:3000/grafana` (admin:admin_password)

## Commands

| Task | Command |
| --- | --- |
| Validate config | `podman compose config` |
| Start | `podman compose up -d` |
| Stream logs | `podman compose logs -f asciinema` |
| Grafana logs | `podman compose logs -f grafana` |
| Prometheus logs | `podman compose logs -f prometheus` |
| Stop (data persists) | `podman compose down` |

## Login

- **Asciinema**: Sign-in links are printed to the container logs:
  ```sh
  podman compose logs asciinema | grep http
  ```

- **Grafana**: Default credentials - change immediately:
  ```sh
  # Login with admin/admin_password
  # Change via Grafana UI or environment variable
  podman exec -it asciinema_server_grafana bash -c "gftool admin reset-password admin"
  ```

## Data & backups

- Recordings: `/srv/asciinema/asciinema`
- Database: `/srv/asciinema/postgres`
- **NEW**: Prometheus data: `./prometheus_data`
- **NEW**: Grafana data: `./grafana_data`

Both are host bind mounts. Never delete or reset them; back them up to preserve recordings.

Note: these dirs were created by an earlier rootful Docker run and are root/system-owned, which rootless Podman cannot `chown` — the postgres container fails with `chown: Operation not permitted`. `podman unshare chown` cannot fix this (no power over files owned by unmapped real root); it needs sudo. The dirs are currently empty (the stack never initialized), so: `sudo chown -R $USER:$USER /srv/asciinema`, then `podman compose up -d`.

## Monitoring

### Overview
The stack includes a comprehensive monitoring solution with:

- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboarding
- **Alertmanager**: Alert routing and notification

### Metrics Collected

#### System Metrics
- CPU usage (per core and overall)
- Memory usage and allocation
- Disk I/O and space utilization
- Network traffic and bandwidth
- System load and process counts

#### Application Metrics
- Asciinema HTTP request rates
- PostgreSQL query performance
- Connection counts and timeouts
- Database size and growth

#### Container Metrics
- Container CPU and memory usage
- Docker container counts
- Network I/O between containers

### Grafana Dashboards

Available dashboards:
1. **System Overview**: Host system metrics
2. **Application Metrics**: Asciinema and database performance
3. **Database Performance**: PostgreSQL-specific metrics
4. **Container Performance**: Docker container metrics

### Alerting

Critical alerts include:
- **High CPU/Memory Usage**: System resource alerts
- **Disk Space Low**: Storage capacity warnings
- **Service Down**: Application and database availability
- **High Network Load**: Network performance alerts

Alerts are routed through Alertmanager with support for:
- Email notifications
- Slack integration
- PagerDuty integration
- Grafana webhook notifications

## Printer (OctoPrint)

**NEW**: Integrated 3D printer support:

- USB printer connection via device passthrough
- Tailscale integration for remote access
- OctoPrint web interface at `http://<URL_HOST>:8081`
- Print job monitoring and logging

### Printer Configuration

**Login**: Default credentials - change immediately:
```sh
# Login with admin/admin or octoprint/octoprint
# Change via OctoPrint UI
```

**Access**: Printer is accessible via Tailscale at port 8081

**Monitoring**: Printer metrics are collected and displayed in the Container Performance dashboard

## Upgrade

`SECRET_KEY_BASE` must never change after first boot — it invalidates sessions and encrypted data.

The image tags are pinned in `compose.yml`. To upgrade:

1. Back up `/srv/asciinema`
2. `podman compose down`
3. Bump the tags in `compose.yml` (check releases first), then `podman compose up -d`

## HTTPS

Not configured. The stack serves plain HTTP on ports:
- 8080 for asciinema
- 3000 for Grafana
- 8081 for OctoPrint

For TLS, put reverse proxies in front and set `URL_SCHEME=https`.

## Notes

- There is no `restart:` policy in `compose.yml` — after a crash, `podman compose down`, or a host reboot, start the stack again with `podman compose up -d`.
- podman-compose does not recreate existing containers: if crashed or stale ones exist, `up` fails with "name already in use" and then just starts the old ones. Run `podman compose down` first whenever containers already exist.
- Postgres uses trust auth (no password) and is only reachable inside the compose network — never publish its port.
- Host ports: `8080` (asciinema) → container `4000`, `3000` (Grafana), `8081` (OctoPrint) → container `8080`
- `TZ=UTC` is fixed; timestamps are in UTC.
- **NEW**: Monitoring stack uses separate `monitoring` network for security isolation
- **NEW**: All containers are configured with health checks and restart policies
- **NEW**: Rootless operation maintained with proper device permissions

## Troubleshooting

### Common Issues

**1. Port conflicts**
```bash
# Check what's using ports
netstat -tlnp | grep -E ':(3000|8080|8081)'
# Change ports in compose.yml if needed
```

**2. Monitoring services not starting**
```bash
# Check logs
podman compose logs prometheus
grafana
podman compose logs -f node-exporter
```

**3. Grafana login issues**
```bash
# Reset admin password
podman exec -it asciinema_server_grafana bash -c "gftool admin reset-password admin"
```

**4. Database connection issues**
```bash
# Check PostgreSQL health
podman exec asciinema_server_postgres pg_isready
# Check logs
podman logs asciinema_server_postgres
```

### Monitoring Maintenance

**Regular checks**:
- Monitor container health: `podman ps`
- Check logs for errors: `podman compose logs -f`
- Alert status: `curl http://localhost:9093`
- Data retention: Review Prometheus storage configuration

**Backup procedures**:
- Database: `/srv/asciinema/postgres`
- Asciinema recordings: `/srv/asciinema/asciinema`
- Grafana: `/home/urdaibayc/asciinema_server/grafana_data`
- Prometheus: `/home/urdaibayc/asciinema_server/prometheus_data`

**Security best practices**:
- Change default passwords immediately
- Configure firewall rules
- Use Tailscale for secure remote access
- Regular updates and patches
- Monitor for unusual activity in dashboards

## Service Dependencies

The monitoring stack adds these service dependencies:

- Prometheus requires: node-exporter, cadvisor, pushgateway
- Grafana requires: Prometheus datasource
- Alertmanager requires: Grafana integration

All services are designed to start independently and communicate through the `monitoring` network.

