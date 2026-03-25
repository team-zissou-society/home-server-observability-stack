# Observability Stack

A complete monitoring, logging, and alerting solution for Docker-based homelabs. Provides metrics collection, log aggregation, visualization, and automatic container recovery in a single `docker-compose` deployment.

**This is a template for your own deployment.** You will need to configure it with your own credentials, domain, and alert targets before using.

## Quick Links

- **Blog post**: [Building a Production-Grade Observability Stack for My Homelab](https://example.com/blog) — detailed walkthrough of the architecture and lessons learned
- **GitHub**: [home-server-observability-stack](https://github.com/your-org/home-server-observability-stack)
- **Issues**: [Report bugs or ask questions](https://github.com/your-org/home-server-observability-stack/issues)

## What's Included

- **Prometheus** — time-series metrics collection and short-term storage (30 days)
- **Grafana** — dashboards and visualization (uses Prometheus + Loki)
- **Loki + Promtail** — log aggregation and searching
- **Alertmanager** — alert routing and delivery (email, webhooks, etc.)
- **Node Exporter** — host-level metrics (CPU, memory, disk, network)
- **cAdvisor** — container-level metrics (memory, CPU, I/O)
- **DCGM Exporter** — NVIDIA GPU metrics (utilization, VRAM, temperature)
- **Uptime Kuma** — HTTP/TCP health checks for external services
- **Autoheal** — automatic container restart on failed health checks

## Prerequisites

- Docker Engine 24+ with Compose v2
- NVIDIA Container Toolkit (optional, only if you have GPUs)
- Python 3 or similar for running the compose-refresh script

## Quick Start

**IMPORTANT**: Before starting, follow these security steps:

1. **Never commit `.env` to git** — it's in `.gitignore` to prevent accidental secret exposure
2. **Set strong passwords** — change all placeholder values in `.env`
3. **Rotate credentials if this repo was exposed** — regenerate webhooks, passwords, SMTP credentials

### Setup Steps

1. **Clone or navigate to the observability stack directory:**
   ```bash
   cd observabillity_stack
   ```

2. **Create `.env` from the template:**
   ```bash
   cp .env.example .env
   ```

3. **Edit `.env` and set required values:**
   ```bash
   GRAFANA_ADMIN_PASSWORD=your_secure_password
   GRAFANA_DOMAIN=localhost  # or your actual domain
   ```

4. **Start the stack:**
   ```bash
   docker compose up -d
   ```

5. **Verify services are healthy:**
   ```bash
   docker compose ps
   ```

## Accessing Services

| Service | URL | Default Credentials |
|---------|-----|-------------------|
| **Grafana** | http://localhost:3000 | admin / (see .env) |
| **Prometheus** | http://localhost:9090 | none |
| **Alertmanager** | http://localhost:9093 | none |
| **Uptime Kuma** | http://localhost:3001 | setup on first visit |
| **Node Exporter** | http://localhost:9100/metrics | metrics only |
| **cAdvisor** | http://localhost:8081 | none |
| **Loki** | http://localhost:3100 | metrics/api only |

## Directory Structure

```
observabillity_stack/
├── docker-compose.yml              # Main service definitions
├── .env.example                    # Environment variables template
├── config/
│   ├── prometheus/
│   │   ├── prometheus.yml          # Prometheus scrape jobs
│   │   └── alerts.yml              # Alert rules
│   ├── alertmanager/
│   │   └── alertmanager.yml        # Alert routing and receivers
│   ├── loki/
│   │   └── loki.yml                # Loki retention and storage
│   └── promtail/
│       └── promtail.yml            # Log collection sources
└── README.md                       # This file
```

## Configuration

### Environment Variables (.env)

```bash
# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=your_secure_password  # REQUIRED
GRAFANA_DOMAIN=localhost
GRAFANA_SMTP_ENABLED=false
GRAFANA_SMTP_HOST=smtp.example.com
GRAFANA_SMTP_USER=alerts@example.com
GRAFANA_SMTP_PASSWORD=smtp_password
GRAFANA_SMTP_FROM=alerts@example.com
```

### Prometheus Configuration (config/prometheus/prometheus.yml)

Defines which services to scrape metrics from. By default includes:
- Node Exporter (9100) — host metrics
- cAdvisor (8081) — container metrics
- DCGM Exporter (9400) — GPU metrics
- Prometheus (9090) — self-monitoring

Add additional scrape targets here.

### Alert Rules (config/prometheus/alerts.yml)

Contains alert rules for:
- **System alerts** — high CPU, high memory, low disk space
- **Container alerts** — restart loops, high CPU, high memory (with memory limit guard)
- **GPU alerts** — NVENC overload, high temperature, high memory
- **Service alerts** — targets down, Prometheus scrape failures

Edit threshold values (e.g., CPU > 90%, memory > 90%) here.

### Alertmanager Routes (config/alertmanager/alertmanager.yml)

Defines where alerts go (email, Slack, PagerDuty, webhooks, etc.). The base configuration sends to a null receiver; configure your actual notification channels here.

### Loki Config (config/loki/loki.yml)

Retention settings, compactor config, and storage backend. Default uses local filesystem storage at `/loki`.

### Promtail Config (config/promtail/promtail.yml)

Defines which logs to collect. By default collects Docker container logs and system logs.

## Grafana Dashboards

Recommended community dashboards to import:

1. **Node Exporter Full** (ID: 1860)
   - Host CPU, memory, disk, network
   - Most important dashboard for overall system health

2. **Docker Container Monitoring** (ID: 893)
   - Per-container CPU, memory, restarts
   - Network I/O and filesystem usage

3. **NVIDIA GPU Metrics** (ID: 12239)
   - GPU utilization, VRAM, temperature, power draw
   - Only needed if you have NVIDIA GPUs

### Importing Dashboards

1. Go to Grafana → **Dashboards** (or **+** icon)
2. Click **Import**
3. Enter dashboard ID above
4. Select **Prometheus** datasource
5. Click **Import**

## Known Issues & Gotchas

### Loki Requires Explicit Compactor Config

Loki 3.x requires `delete_request_store` when retention is enabled:

```yaml
compactor:
  retention_enabled: true
  delete_request_store: filesystem
```

If Loki crashes with "compactor.delete-request-store should be configured," check the config.

### Promtail Cannot Replay Years of Logs

On first startup, Promtail reads Docker logs from the beginning. Loki rejects logs older than the retention period, generating many error lines. This is harmless — Promtail advances through the backlog and eventually catches up.

### Promtail is Distroless (No Health Check)

The Grafana Promtail image doesn't include shell, wget, or curl. There's no reliable health check. Using `service_started` instead of `service_healthy` in dependents is the recommended approach; Autoheal handles genuine failures.

### cAdvisor CPU Usage on ZFS/Many Filesystems

cAdvisor can use 100%+ CPU if the host has many filesystems (ZFS datasets, snap mounts, overlay networks). Symptoms: cAdvisor process eating CPU continuously.

See `docker-compose.yml` for disabled cAdvisor flags that reduce scope. Uncomment to optimize for your setup.

### ContainerHighMemory Alert Fires for Unlimited Containers

The `ContainerHighMemory` alert requires an explicit `mem_limit` on containers to work correctly. Containers without limits cause division-by-zero in the calculation. The alert rule includes a guard (`container_spec_memory_limit_bytes > 0`) to prevent false positives.

### Community Dashboards Use Wrong Datasource UID

Imported dashboards reference datasources by UID. If the UID doesn't match, panels show "datasource not found" (magenta panels). Fix by provisioning datasources via config with a fixed UID rather than through the UI.

## Troubleshooting

### Services won't start

1. Check logs:
   ```bash
   docker compose logs -f prometheus
   docker compose logs -f grafana
   ```

2. Verify config files exist:
   ```bash
   ls -la config/prometheus/*.yml
   ls -la config/loki/*.yml
   ls -la config/alertmanager/*.yml
   ls -la config/promtail/*.yml
   ```

3. Check `.env`:
   ```bash
   cat .env | grep GRAFANA_ADMIN_PASSWORD
   # Should have a value set
   ```

### Prometheus shows "No Targets Found"

- Verify scrape targets are reachable (test ports like 9100, 8081, 9400)
- Check `config/prometheus/prometheus.yml` for typos
- Prometheus must be able to reach target hosts/ports on the `observability` bridge network

### Grafana dashboards show no data

1. Check datasource configuration — go to **Configuration → Data Sources**
2. Verify datasource UID matches what dashboards expect (usually `DS_PROMETHEUS` or `prometheus`)
3. Test the connection (red/green indicator on datasource page)

### Alerts aren't firing

1. Check alert rules in Prometheus: http://localhost:9090/alerts
2. Look for alert syntax errors in `config/prometheus/alerts.yml`
3. Ensure Alertmanager is running and healthy:
   ```bash
   docker compose ps alertmanager
   docker compose logs alertmanager
   ```

### Loki crashes on startup

Check for the compactor error mentioned above. Verify `config/loki/loki.yml` includes:

```yaml
compactor:
  retention_enabled: true
  delete_request_store: filesystem
```

### High memory usage

Prometheus defaults to 30 days of metrics retention. Reduce with:

```yaml
storage.tsdb.retention.time=14d
storage.tsdb.retention.size=5GB
```

in `docker-compose.yml`.

## Maintenance

### Update images

```bash
docker compose pull
docker compose up -d
```

### Backup Grafana dashboards and datasources

```bash
docker cp grafana:/var/lib/grafana ./backups/grafana-data-$(date +%s)
```

### Clean old data

Docker volumes persist data. To reset a service:

```bash
docker compose down prometheus
docker volume rm observabillity_stack_prometheus_data
docker compose up -d prometheus
```

## Resource Usage

At steady state with 20+ other Docker services running:

- Prometheus: ~150-200 MB RAM, <1% CPU
- Grafana: ~100-150 MB RAM, <1% CPU
- Loki: ~50-100 MB RAM, <1% CPU
- cAdvisor: ~5-10% CPU (heavily depends on number of containers/filesystems)
- Node Exporter + DCGM Exporter: negligible

## Security Notes

### For Your Deployment

- **Change all `.env` values** — the `.env.example` file contains placeholders, not real credentials
- **Grafana admin password** — Set a strong password (16+ characters) in `.env` before first run
- **Disable Grafana sign-ups** — `GF_USERS_ALLOW_SIGN_UP=false` (enabled by default) if exposed to network
- **Keep Prometheus/Alertmanager behind firewall** — These should not be directly internet-exposed; use a reverse proxy (nginx, Traefik) with authentication
- **Use HTTPS for Grafana** — Configure reverse proxy with TLS for production deployments
- **Rotate credentials regularly** — Change Grafana password, regenerate Discord webhooks, rotate SMTP credentials

### Secrets Management

- `.env` file is in `.gitignore` — it will never be committed to git
- Never commit actual credentials, API keys, or webhook URLs to the repository
- For CI/CD, use Docker secrets or a secrets manager instead of `.env`
- If you ever expose credentials accidentally:
  1. Immediately rotate the credential
  2. Regenerate API keys/webhooks
  3. Force password change on affected services

### External Services

- **Discord webhooks** — Treat like passwords; regenerate immediately if leaked
- **SMTP credentials** — Use dedicated service account; never use personal/admin passwords
- **Grafana sessions** — Run behind reverse proxy with rate limiting to prevent brute force
- **Prometheus metrics** — May expose sensitive information; keep internal-only or behind authentication

## See Also

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [CONTRIBUTING](CONTRIBUTING.md) — Guidelines for contributing to this project
