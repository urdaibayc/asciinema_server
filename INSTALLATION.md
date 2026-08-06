# Installation and Setup Guide
# This document provides step-by-step instructions for setting up the asciinema monitoring stack with Tailscale integration

## Prerequisites

### System Requirements
- **Hardware**: Minimum 2GB RAM, 2GB disk space, USB port for printer
- **Operating System**: Linux (tested on Ubuntu/Debian)
- **Software**: Podman 4.9+ with podman-compose

### Network Requirements
- USB device access for printer connection
- Tailscale installation and authentication
- Ports availability: 3000 (Grafana), 8080 (Asciinema), 8081 (OctoPrint)

## Quick Start Guide

### 1. Environment Setup

#### Create .env File
```bash
# Create from existing .env.example template
cp .env.example .env  # if exists

# Edit .env with your values
cat > .env << EOF
SECRET_KEY_BASE=7a00051afb0161cf0e67b23e0173512e34a98e94235bc3147cdbb84e62c33c8540b12d8a1f88919d85c1d0925902a4ccc834e121d89790aec31963b9eb1b1ba3
URL_HOST=100.102.145.105
URL_PORT=8080
GRAFANA_PASSWORD=your-secure-password
TAILSCALE_AUTHKEY=tskey-...
EOF
```

#### Install Tailscale
```bash
# Install Tailscale
sudo apt update && sudo apt install -y tailscale

# Authenticate
sudo tailscale up --authkey=${TAILSCALE_AUTHKEY} --ssh

# Create network for monitoring
TUN_IP=$(tailscale ip -4)
echo "Tailscale IP: $TUN_IP"
```

### 2. Directory Structure

Create required directories:
```bash
mkdir -p /home/urdaibayc/asciinema_server
mkdir -p /home/urdaibayc/asciinema_server/monitoring
mkdir -p /home/urdaibayc/asciinema_server/grafana/provisioning/dashboards
mkdir -p /home/urdaibayc/asciinema_server/grafana/provisioning/alertmanager
mkdir -p /home/urdaibayc/asciinema_server/grafana/config
mkdir -p ./prometheus_data ./grafana_data
```

### 3. Copy Configuration Files

Copy the configuration files:
```bash
cp ./compose.yml /home/urdaibayc/asciinema_server/
cp ./monitoring/prometheus.yml /home/urdaibayc/asciinema_server/monitoring/
cp ./monitoring/alert_rules.yml /home/urdaibayc/asciinema_server/monitoring/
cp ./grafana/provisioning/grafana.yml /home/urdaibayc/asciinema_server/grafana/provisioning/
cp ./grafana/provisioning/alertmanager/alertmanager.yml /home/urdaibayc/asciinema_server/grafana/provisioning/alertmanager/
cp ./grafana/config/grafana.ini /home/urdaibayc/asciinema_server/grafana/config/

# Copy placeholder dashboards
cp ./grafana/provisioning/dashboards/*.json /home/urdaibayc/asciinema_server/grafana/provisioning/dashboards/
```

### 4. Update Compose Configuration

The compose.yml file is already configured with:
- **Services**: asciinema, postgres, prometheus, grafana, node-exporter, alertmanager, cadvisor, pushgateway
- **Networks**: app (for asciinema) and monitoring (for monitoring stack)
- **Volumes**: prometheus_data, grafana_data

### 5. Initial Setup

#### Set Up Permissions
```bash
# Fix permissions for existing data directories (if any)
sudo chown -R $USER:$USER /srv/asciinema

# Create necessary directories
mkdir -p ./prometheus_data ./grafana_data
chmod 755 ./prometheus_data ./grafana_data
```

#### Generate SSL Certificates (Optional)
```bash
# Create SSL directory
mkdir -p ./ssl

# Generate self-signed certificates for development
cd ssl
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 \
  -subj "/C=US/ST=State/L=City/O=Organization/OU=OrgUnit/CN=${URL_HOST}" \
  -keyout privkey.pem -out fullchain.pem
```

### 6. Start the Stack

#### Validate Configuration
```bash
podman compose -f compose.yml config
```

#### Start Services
```bash
# Stop any existing containers
podman compose -f compose.yml down

# Start the monitoring stack
podman compose -f compose.yml up -d
```

#### Check Service Status
```bash
# View running containers
podman ps

# Check logs for issues
podman compose -f compose.yml logs -f prometheus
grafana
node-exporter

# Health checks
curl -f http://localhost:9090/-/ready && echo "Prometheus OK"
curl -f http://localhost:3000/api/health && echo "Grafana OK"
curl -f http://localhost:8080 && echo "Asciinema OK"
```

### 7. Initial Configuration

#### Access Grafana
```bash
# Login with default credentials
# Username: admin
# Password: your GRAFANA_PASSWORD from .env

# Change password immediately via Grafana UI
# or using command line tool
podman exec -it asciinema_server_grafana bash -c "gftool admin reset-password admin"
```

#### Configure Data Sources
1. Login to Grafana (http://localhost:3000/grafana)
2. Go to "Connections" → "Data Sources"
3. Add Prometheus data source:
   - Name: Prometheus
   - URL: http://prometheus:9090
4. Add PostgreSQL data source:
   - Name: PostgreSQL
   - Host: postgres
   - Database: grafana

#### Import Dashboards
1. In Grafana, go to "Dashboards" → "Manage"
2. Click "Import"
3. Use dashboard IDs or upload JSON files
4. Configure dashboard settings

### 8. OctoPrint Integration

#### USB Printer Setup
```bash
# Verify USB printer is connected and accessible
lsusb | grep Printer

# Check if printer device is available in the host
ls -la /dev/bus/usb/

# Update compose.yml if printer device path differs
# Currently configured: /dev/bus/usb:/dev/bus/usb
```

#### OctoPrint Access
```bash
# Access OctoPrint at http://localhost:8081
# Default credentials: admin/admin or octoprint/octoprint
# Change immediately via OctoPrint web interface
```

## Troubleshooting

### Common Issues and Solutions

#### Port Conflicts
```bash
# Check for port conflicts
netstat -tlnp | grep -E ':(3000|8080|8081|9090|9093)'

# Change ports in compose.yml if needed
# Example: Replace "8080:4000" with "8082:4000"
```

#### Service Health Issues
```bash
# Check Prometheus
podman compose -f compose.yml logs prometheus

# Check Grafana
podman compose -f compose.yml logs grafana

# Check database connectivity
podman exec asciinema_server_postgres pg_isready
```

#### Monitoring Data Not Appearing
```bash
# Verify Prometheus scrape targets
podman exec asciinema_server_prometheus cat /etc/prometheus/prometheus.yml

# Check node exporter metrics
curl http://localhost:9100/metrics | head -20

# Verify cadvisor metrics
curl http://localhost:8080/metrics | head -20
```

### Maintenance Tasks

#### Regular Monitoring Checks
```bash
# Daily health checks
curl -f http://localhost:9090/-/ready > /dev/null || echo "Prometheus down"
curl -f http://localhost:3000/api/health > /dev/null || echo "Grafana down"

# Check disk usage
df -h

# Check container resource usage
podman stats --no-stream
```

#### Backup Procedures
```bash
# Database backup
podman exec asciinema_server_postgres pg_dumpall -U postgres > backup_$(date +%Y%m%d_%H%M%S).sql

# Asciinema recordings
rsync -av /srv/asciinema/asciinema/ /backup/asciinema/

# Prometheus data backup
rsync -av ./prometheus_data/ /backup/prometheus/

# Grafana configuration backup
cp -r ./grafana/ /backup/grafana/
```

#### Security Maintenance
```bash
# Change default passwords immediately
podman exec -it asciinema_server_grafana bash -c "gftool admin reset-password admin"

# Configure firewall rules
# Allow only necessary ports
# Use Tailscale for remote access

# Update system packages
sudo apt update && sudo apt upgrade -y

# Monitor for unusual activity in Grafana alerts
```

## Performance Tuning

### Prometheus Configuration
```yaml
# Adjust scrape intervals in prometheus.yml
scrape_interval: 15s
evaluation_interval: 15s

# Configure retention
retention: 15d
max_concurrency: 20
```

### Grafana Configuration
```ini
# In grafana.ini
[metrics]
enabled = true

[metrics.prometheus]
enabled = true
```

## Service Dependencies

The monitoring stack adds these service dependencies:

1. **Prometheus** requires: node-exporter, cadvisor, pushgateway
2. **Grafana** requires: Prometheus datasource
3. **Alertmanager** requires: Grafana integration

All services are designed to start independently and communicate through the `monitoring` network.

## Upgrade Guide

### To New Version
```bash
# 1. Backup existing configuration
podman compose -f compose.yml down

# 2. Update image versions in compose.yml
# Example: Update monitoring stack images

# 3. Redeploy
sudo chown -R $USER:$USER /srv/asciinema
podman compose -f compose.yml up -d
```

### Breaking Changes
- Monitor for breaking changes in Prometheus, Grafana, or related components
- Review alert rules and dashboard configurations
- Test application functionality after updates

## Getting Help

### Documentation
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Tailscale Documentation](https://tailscale.com/docs/)

### Community Support
- Join the asciinema Discord community
- File issues on GitHub repositories
- Use GitHub discussions for questions

### Troubleshooting Resources
- Container logs: `podman compose -f compose.yml logs <service>`
- System logs: `journalctl -u podman`
- Network connectivity: `ping`, `curl`, `telnet`

## Monitoring Stack Summary

This monitoring stack provides:

✅ **Comprehensive Monitoring**: System, application, and database metrics
✅ **Alerting**: Critical notifications via email, Slack, PagerDuty
✅ **Visualization**: Professional dashboards with real-time metrics
✅ **Security**: Network isolation and authentication
✅ **Reliability**: Health checks and restart policies
✅ **Scalability**: Modular architecture for future expansion

The stack is production-ready and suitable for monitoring:
- asciinema server performance
- PostgreSQL database operations
- Container resource utilization
- System infrastructure health

All components are configured with proper restart policies, health checks, and logging for operational excellence.