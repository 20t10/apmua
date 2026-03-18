# Grafana Dashboard Tutorial

## สร้าง Dashboard สำหรับ MESX-ERP Monitoring

### Overview
Dashboard นี้แสดงข้อมูล 3 ส่วนหลัก:
1. **Prometheus Metrics** - HTTP Request Rate, Latency, Requests by Method/Status
2. **Loki Logs** - Backend Log Rate และ Live Logs
3. **Automatic Provisioning** - Dashboard ถูกโหลดอัตโนมัติเมื่อ start container

---

## Step 1: สร้างโครงสร้าง Directory

```bash
mkdir -p apmua/grafana/provisioning/dashboards
```

**โครงสร้างที่ต้องการ:**
```
apmua/
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasources.yml
        └── dashboards/
            ├── dashboard.yml
            └── mesx-erp-backend.json
```

---

## Step 2: สร้าง Dashboard Provider Config

สร้างไฟล์ `apmua/grafana/provisioning/dashboards/dashboard.yml`:

```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

**คำอธิบาย:**
- `type: file` - อ่าน dashboard จากไฟล์ .json
- `orgId: 1` - ใช้ organization แรก
- `path` - path ที่ container จะอ่านไฟล์

---

## Step 3: สร้าง Dashboard JSON

สร้างไฟล์ `apmua/grafana/provisioning/dashboards/mesx-erp-backend.json`

### 3.1 กำหนด Dashboard Metadata

```json
{
  "annotations": { "list": [] },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "liveNow": false,
  "refresh": "5s",
  "schemaVersion": 38,
  "style": "dark",
  "tags": ["mesx-erp", "monitoring"],
  "templating": { "list": [] },
  "time": {
    "from": "now-15m",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "MESX-ERP Backend Monitoring",
  "uid": "mesx-erp-backend",
  "version": 1,
  "weekStart": ""
}
```

### 3.2 เพิ่ม Prometheus Panels

**Panel 1: HTTP Request Rate (Stat Panel)**

```json
{
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "reqps"
    }
  },
  "targets": [
    {
      "expr": "sum(rate(http_requests_total[5m]))",
      "legendFormat": "Requests/sec"
    }
  ],
  "title": "HTTP Request Rate",
  "type": "stat"
}
```

**Panel 2: HTTP Request Latency (Time Series)**

```json
{
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"
  },
  "targets": [
    {
      "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
      "legendFormat": "p50"
    },
    {
      "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
      "legendFormat": "p95"
    },
    {
      "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
      "legendFormat": "p99"
    }
  ],
  "title": "HTTP Request Latency",
  "type": "timeseries"
}
```

**Panel 3: Requests by Method & Status (Stacked Bar)**

```json
{
  "targets": [
    {
      "expr": "sum by (method, status) (rate(http_requests_total[5m]))",
      "legendFormat": "{{method}} {{status}}"
    }
  ],
  "title": "Requests by Method & Status",
  "type": "timeseries"
}
```

### 3.3 เพิ่ม Loki Panels

**Panel 4: Backend Log Rate**

```json
{
  "datasource": {
    "type": "loki",
    "uid": "loki"
  },
  "targets": [
    {
      "expr": "count_over_time({job=\"mesx-erp_backend\"}[5m])",
      "legendFormat": "Log Rate"
    }
  ],
  "title": "Backend Log Rate",
  "type": "timeseries"
}
```

**Panel 5: Live Logs (Logs Panel)**

```json
{
  "datasource": {
    "type": "loki",
    "uid": "loki"
  },
  "options": {
    "dedupStrategy": "none",
    "enableLogDetails": true,
    "prettifyLogMessage": true,
    "showCommonLabels": false,
    "showLabels": false,
    "showTime": true,
    "sortOrder": "Descending",
    "wrapLogMessage": true
  },
  "targets": [
    {
      "expr": "{job=\"mesx-erp_backend\"}"
    }
  ],
  "title": "Backend Logs (Live)",
  "type": "logs"
}
```

---

## Step 4: Mount Volume ใน docker-compose.yml

เพิ่ม volume mount สำหรับ provisioning:

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3003:3003"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning  # เพิ่มบรรทัดนี้
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SERVER_HTTP_PORT=3003
```

---

## Step 5: Restart Grafana Container

```bash
cd apmua
docker compose restart grafana
```

---

## Step 6: ตรวจสอบว่า Dashboard ถูกสร้าง

### ผ่าน Terminal
```bash
curl -u admin:admin "http://localhost:3003/api/search?type=dash-db" | jq '.[] | {uid, title}'
```

**Expected Output:**
```json
{
  "uid": "mesx-erp-backend",
  "title": "MESX-ERP Backend Monitoring"
}
```

### ผ่าน Browser
1. เปิด http://localhost:3003
2. Login: admin / admin
3. ไปที่ Dashboards > Browse
4. จะเห็น "MESX-ERP Backend Monitoring"

---

## คำสั่งที่เป็นประโยชน์

### ดู Dashboard JSON
```bash
curl -u admin:admin "http://localhost:3003/api/dashboards/uid/mesx-erp-backend"
```

### Export Dashboard จาก Grafana UI
1. เปิด Dashboard
2. คลิก Share > Export > Save to file

### Reset Admin Password
```bash
docker exec grafana grafana cli admin reset-admin-password admin
```

---

## Dashboard Panel Types ที่ใช้บ่อย

| Type | Use Case |
|------|----------|
| `stat` | แสดงค่าเดียว (Total, Current) |
| `timeseries` | แสดงกราฟเส้น/bar ตามเวลา |
| `logs` | แสดง log entries |
| `table` | แสดงข้อมูลแบบตาราง |
| `gauge` | แสดงค่าแบบ gauge |
| `piechart` | แสดงสัดส่วน |

---

## การ Query Prometheus

### Basic Metrics
```promql
# Counter (นับรวม)
http_requests_total

# Rate (ค่าต่อวินาที)
rate(http_requests_total[5m])

# Histogram Quantiles
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### ด้วย Labels
```promql
# Filter by method
rate(http_requests_total{method="GET"}[5m])

# Group by label
sum by (status) (rate(http_requests_total[5m]))
```

---

## การ Query Loki

### Basic Log Query
```logql
{job="mesx-erp_backend"}
```

### Filter by Level
```logql
{job="mesx-erp_backend"} |= "ERROR"
```

### Count Logs Over Time
```logql
count_over_time({job="mesx-erp_backend"}[5m])
```

### Parse JSON Logs
```logql
{job="mesx-erp_backend"} | json | status = 500
```
