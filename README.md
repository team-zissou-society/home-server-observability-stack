# Observability Stack

A complete monitoring, logging, and alerting solution for Docker-based homelabs. Provides metrics collection, log aggregation, visualization, and automatic container recovery in a single `docker-compose` deployment.

This is a template for your own deployment. You will need to configure it with your own credentials, domain, and alert targets before using.

## Quick Links

- Blog post: [Building a Production-Grade Observability Stack for My Homelab](https://toddco.xyz/post/observability-stack--for-my-homelab/) — detailed walkthrough of the architecture and lessons learned
- GitHub: [home-server-observability-stack](https://github.com/team-zissou-society/home-server-observability-stack)
- Issues: [Report bugs or ask questions](https://github.com/team-zissou-society/home-server-observability-stack/issues)

## What's Included

- Prometheus — time-series metrics collection and short-term storage (30 days)
- Grafana — dashboards and visualization (uses Prometheus + Loki)
- Loki + Promtail — log aggregation and searching
- Alertmanager — alert routing and delivery (email, webhooks, etc.)
- Node Exporter — host-level metrics (CPU, memory, disk, network)
- cAdvisor — container-level metrics (memory, CPU, I/O)
- DCGM Exporter — NVIDIA GPU metrics (utilization, VRAM, temperature)
- Uptime Kuma — HTTP/TCP health checks for external services
- Autoheal — automatic container restart on failed health checks

## Prerequisites

- Docker Engine 24+ with Compose v2
- NVIDIA Container Toolkit (optional, only if you have GPUs)

### Setup Steps

1. Clone or navigate to the observability stack directory:
   ```bash
   cd home-server-observability_stack
   ```

2. Create `.env` from the template:
   ```bash
   cp .env.example .env
   ```

3. Edit `.env` and set required values:
   ```bash
   GRAFANA_ADMIN_PASSWORD=your_secure_password
   GRAFANA_DOMAIN=localhost  # or your actual domain
   ```

4. Start the stack:
   ```bash
   docker compose up -d
   ```

5. Verify services are healthy:
   ```bash
   docker compose ps
   ```

## Accessing Services

| Service | URL | Default Credentials |
|---------|-----|-------------------|
| Grafana | http://localhost:3000 | admin / (see .env) |
| Prometheus | http://localhost:9090 | none |
| Alertmanager | http://localhost:9093 | none |
| Uptime Kuma | http://localhost:3001 | setup on first visit |
| Node Exporter | http://localhost:9100/metrics | metrics only |
| cAdvisor | http://localhost:8081 | none |
| Loki | http://localhost:3100 | metrics/api only |

## Configuration

### Alertmanager Routes (config/alertmanager/alertmanager.yml)

Defines where alerts go (email, Slack, PagerDuty, webhooks, etc.). The base configuration sends to a null receiver; configure your actual notification channels here.


## Grafana Dashboards

Recommended community dashboards to import:

1. Node Exporter Full (ID: 1860)
   - Host CPU, memory, disk, network
   - Most important dashboard for overall system health

2. Docker Container Monitoring (ID: 893)
   - Per-container CPU, memory, restarts
   - Network I/O and filesystem usage

3. NVIDIA GPU Metrics (ID: 12239)
   - GPU utilization, VRAM, temperature, power draw
   - Only needed if you have NVIDIA GPUs

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

1. Check datasource configuration — go to Configuration → Data Sources
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
docker volume rm observability_stack_prometheus_data
docker compose up -d prometheus
```

## Resource Usage

At steady state with 20+ other Docker services running:

- Prometheus: ~150-200 MB RAM, <1% CPU
- Grafana: ~100-150 MB RAM, <1% CPU
- Loki: ~50-100 MB RAM, <1% CPU
- cAdvisor: ~5-10% CPU (heavily depends on number of containers/filesystems)
- Node Exporter + DCGM Exporter: negligible


## For Your Deployment

- Change all `.env` values — the `.env.example` file contains placeholders, not real credentials
- Prometheus metrics — May expose sensitive information; keep internal-only or behind authentication
