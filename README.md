# apmua

Application Performance Monitoring, Usage Analytics stack using Prometheus, Loki, and Grafana.

## Services

| Service | Port | Purpose |
|---------|------|---------|
| **Prometheus** | 9009 | Metrics collection and alerting |
| **Loki** | 3100 | Log aggregation |
| **Promtail** | - | Log shipper that collects container logs |
| **Grafana** | 3003 | Visualization and dashboards |

## Quick Start

### Start Monitoring Stack

```bash
cd apmua
docker compose up -d
```

### Access Services

- Grafana: http://localhost:3003 (`admin` / `admin`)
- Prometheus: http://localhost:9009
- Loki: http://localhost:3100

### Prerequisites

The monitoring stack connects to the `monitoring` network. Create it if it does not already exist:

```bash
docker network create monitoring
```

## Architecture

```text
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

## Configuration Files

```text
apmua/
├── prometheus/
│   └── prometheus.yml
├── loki/
│   ├── loki-config.yml
│   └── promtail-config.yml
├── grafana/
│   └── provisioning/
├── docker-compose.yml
└── AGENTS.md
```

## Common Tasks

### View Logs

```bash
docker compose logs -f backend
```

### Check Service Health

```bash
curl http://localhost:9009/-/healthy
curl http://localhost:3100/ready
curl http://localhost:3003/api/health
```

### Restart Services

```bash
docker compose restart
```

## Metrics and Logs

Backend exposes metrics at `/metrics`:

```bash
curl http://localhost:8888/metrics
```

Prometheus and Grafana can be used to inspect:

- Request rate
- Response latency
- Error rate
- Usage and log patterns

