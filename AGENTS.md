# apmua

Application Performance Monitoring stack using Prometheus, Loki, and Grafana.

## Services

| Service | Port | Purpose |
|---------|------|---------|
| **Prometheus** | 9009 | Metrics collection and alerting |
| **Loki** | 3100 | Log aggregation |
| **Promtail** | - | Log shipper (collects container logs) |
| **Grafana** | 3003 | Visualization and dashboards |

## Architecture

```
┌─────────────┐
│   Backend   │───┐
│  (Axum App) │   │
└─────────────┘   │
                  │ metrics
                  ▼
            ┌──────────┐
            │Prometheus│
            └──────────┘
                  │
                  │ query
                  ▼
            ┌──────────┐
            │ Grafana  │───┐
            └──────────┘   │
                           │ logs
            ┌──────────┐   │
            │  Loki    │◄──┘
            └──────────┘
                  ▲
                  │ scrape
            ┌──────────┐
            │Promtail  │
            └──────────┘
                  │ collect
            ┌──────────┐
│Docker Logs│───┤/var/log │
└──────────┘   └──────────┘
```

## Quick Start

### Start Monitoring Stack

```bash
cd apmua
docker compose up -d
```

### Access Services

- **Grafana:** http://localhost:3003 (admin/admin)
- **Prometheus:** http://localhost:9009
- **Loki:** http://localhost:3100

### Prerequisites

Monitoring stack connects to the `monitoring` network. Ensure it exists:

```bash
docker network create monitoring
```

## Configuration Files

```
apmua/
├── prometheus/
│   └── prometheus.yml    # Prometheus scrape config
├── loki/
│   ├── loki-config.yml   # Loki storage config
│   └── promtail-config.yml # Promtail scrape config
├── grafana/
│   └── provisioning/      # Auto-provisioned dashboards/datasources
├── docker-compose.yml    # Service definitions
└── AGENTS.md             # This file
```

## Prometheus Configuration

`prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'backend'
    static_configs:
      - targets: ['mesx-erp_backend:8888']
```

**Add custom scrape targets:**
1. Edit `prometheus/prometheus.yml`
2. Add new job under `scrape_configs`
3. Restart: `docker compose restart prometheus`

## Loki Configuration

`loki/loki-config.yml`:

- Configures log storage
- Sets retention policies
- Defines index schema

`loki/promtail-config.yml`:

- Scrapes logs from `/var/log`
- Collects Docker container logs
- Labels logs by service

## Grafana Setup

### Default Credentials

- Username: `admin`
- Password: `admin`

**Change password:**
1. Login to Grafana
2. Profile → Change Password

### Data Sources

Configured in `grafana/provisioning/datasources/`:

- **Prometheus:** http://prometheus:9090
- **Loki:** http://loki:3100

### Dashboards

Auto-provisioned dashboards in `grafana/provisioning/dashboards/`:

- Backend metrics
- System resources
- Request latency
- Error rates

## Viewing Metrics

### Backend Metrics

Backend exposes metrics at `/metrics` endpoint:

```bash
curl http://localhost:8888/metrics
```

Metrics include:
- HTTP request counts
- Response times
- Active connections
- Custom business metrics

### Log Queries (Loki)

In Grafana, use LogQL:

```logql
# All logs from backend
{container="mesx-erp_backend"}

# Error logs
{container="mesx-erp_backend"} |= "error"

# Logs with JSON parsing
{container="mesx-erp_backend"} | json | level = "error"
```

### Metric Queries (PromQL)

```promql
# Request rate (requests per second)
rate(http_requests_total[5m])

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

## Common Tasks

### View Backend Logs

```bash
docker compose logs -f backend
```

Or in Grafana:
1. Explore → Select Loki datasource
2. Query: `{container="mesx-erp_backend"}`

### Check Service Health

```bash
# Prometheus
curl http://localhost:9009/-/healthy

# Loki
curl http://localhost:3100/ready

# Grafana
curl http://localhost:3003/api/health
```

### Restart Services

```bash
docker compose restart prometheus
docker compose restart loki
docker compose restart grafana
```

### Reload Configuration

**Prometheus:**
```bash
curl -X POST http://localhost:9009/-/reload
# Or restart
docker compose restart prometheus
```

**Loki:**
```bash
docker compose restart loki
```

### Access Container Shells

```bash
docker compose exec prometheus sh
docker compose exec loki sh
```

## Creating Dashboards

1. Open Grafana: http://localhost:3003
2. + → Create Dashboard
3. Add Panel → Select metric
4. Save Dashboard

**Export dashboard for provisioning:**
1. Dashboard settings → JSON Model
2. Save to `grafana/provisioning/dashboards/`
3. Restart Grafana: `docker compose restart grafana`

## Alerting

Prometheus alerting rules in `prometheus/alerts.yml` (if configured):

```yaml
groups:
  - name: backend
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High error rate detected"
```

## Persistence

Data stored in Docker volumes:

- `prometheus_data` - Metrics storage
- `loki_data` - Log storage
- `grafana_data` - Dashboards and settings

**Backup volumes:**
```bash
docker run --rm -v apmua_prometheus_data:/data -v $(pwd):/backup alpine tar czf /backup/prometheus_backup.tar.gz -C /data .
```

**Clear all data:**
```bash
docker compose down -v  # Warning: Deletes all monitoring data!
```

## Troubleshooting

### No Logs in Grafana

1. Check Promtail is running: `docker compose ps`
2. Verify Promtail config: `loki/promtail-config.yml`
3. Check log path: `/var/lib/docker/containers`

### No Metrics Showing

1. Verify backend exposes metrics: `curl http://localhost:8888/metrics`
2. Check Prometheus targets: http://localhost:9009/targets
3. Verify network connectivity: `docker network ls`

### Grafana Can't Connect to Datasource

1. Check service is healthy: `docker compose ps`
2. Verify network: Services must be on `monitoring` network
3. Check logs: `docker compose logs grafana`

### High Memory Usage

Reduce retention:
1. Edit `loki/loki-config.yml`
2. Set `retention_period: 168h` (7 days)
3. Restart: `docker compose restart loki`

## Performance Tuning

### Prometheus

- Reduce `scrape_interval` for less frequent collection
- Use `scrape_timeout` to prevent hanging
- Configure `retention` time (default 15d)

### Loki

- Adjust `chunk_size` for log volume
- Configure `retention_period`
- Limit labels with `label_allowlist`

### Grafana

- Set appropriate query timeout
- Use dashboard refresh intervals
- Limit concurrent queries

## Integration with Backend

Backend must:
1. Connect to `monitoring` network
2. Expose metrics endpoint
3. Configure Prometheus exporter

**Backend docker-compose.yml:**
```yaml
services:
  backend:
    networks:
      - default
      - monitoring  # Connect to monitoring network
```

**Backend code (Rust/Axum):**
```rust
use metrics_exporter_prometheus::PrometheusBuilder;

PrometheusBuilder::new()
    .with_registry(registry)
    .install()?;

// Metrics endpoint automatically added at /metrics
```