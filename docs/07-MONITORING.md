# Monitoring Setup Guide

Panduan lengkap untuk setup monitoring stack (Prometheus, Grafana, Loki, Uptime Kuma) untuk sistem Affiliate Marketing Automation.

## 🏗️ Arsitektur Monitoring

```
Applications (API, Workers, n8n)
    │
    ├─→ Metrics → Prometheus → Grafana
    ├─→ Logs → Promtail → Loki → Grafana
    └─→ Health Checks → Uptime Kuma
```

## 📊 Komponen Monitoring

### 1. Prometheus
Metrics collection dan storage untuk time-series data.

### 2. Grafana
Visualization dashboard untuk metrics dan logs.

### 3. Loki
Log aggregation system untuk centralized logging.

### 4. Promtail
Log agent untuk mengirim logs ke Loki.

### 5. Uptime Kuma
Uptime monitoring untuk service availability.

## 🔧 Setup Prometheus

### 1. Buat Konfigurasi Prometheus

```bash
# Create config directory
mkdir -p config/prometheus
```

```yaml
# config/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'affiliate-automation'
    environment: 'production'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: []

# Load rules once and periodically evaluate them
rule_files:
  - "alerts/*.yml"

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          service: 'prometheus'

  # FastAPI Service
  - job_name: 'api'
    static_configs:
      - targets: ['api:8000']
        labels:
          service: 'api'
          environment: 'production'
    metrics_path: '/metrics'
    scrape_interval: 30s

  # n8n
  - job_name: 'n8n'
    static_configs:
      - targets: ['n8n:5678']
        labels:
          service: 'n8n'
          environment: 'production'
    metrics_path: '/metrics'
    scrape_interval: 30s

  # PostgreSQL Exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
        labels:
          service: 'postgres'
          environment: 'production'
    scrape_interval: 30s

  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
        labels:
          service: 'redis'
          environment: 'production'
    scrape_interval: 30s

  # Node Exporter (System metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          service: 'node'
          environment: 'production'
    scrape_interval: 30s

  # cAdvisor (Container metrics)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
        labels:
          service: 'cadvisor'
          environment: 'production'
    scrape_interval: 30s
```

### 2. Buat Alert Rules

```bash
# Create alerts directory
mkdir -p config/prometheus/alerts
```

```yaml
# config/prometheus/alerts/alerts.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      - alert: HighAPILatency
        expr: histogram_quantile(0.95, http_request_latency_seconds_bucket) > 1
        for: 5m
        labels:
          severity: warning
          service: api
        annotations:
          summary: "High API latency detected"
          description: "API latency is above 1 second for 5 minutes"

      - alert: HighAPIErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
          service: api
        annotations:
          summary: "High API error rate detected"
          description: "API error rate is above 5% for 5 minutes"

      - alert: APIDown
        expr: up{job="api"} == 0
        for: 1m
        labels:
          severity: critical
          service: api
        annotations:
          summary: "API service is down"
          description: "API service has been down for 1 minute"

  - name: database_alerts
    interval: 30s
    rules:
      - alert: PostgreSQLDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
          service: postgres
        annotations:
          summary: "PostgreSQL is down"
          description: "PostgreSQL has been down for 1 minute"

      - alert: PostgreSQLHighConnections
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
          service: postgres
        annotations:
          summary: "PostgreSQL high connection count"
          description: "PostgreSQL connection usage is above 80%"

      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 1m
        labels:
          severity: critical
          service: redis
        annotations:
          summary: "Redis is down"
          description: "Redis has been down for 1 minute"

  - name: worker_alerts
    interval: 30s
    rules:
      - alert: WorkerQueueBacklog
        expr: celery_queue_length > 100
        for: 10m
        labels:
          severity: warning
          service: workers
        annotations:
          summary: "Worker queue backlog detected"
          description: "Worker queue has more than 100 pending jobs"

      - alert: WorkerHighFailureRate
        expr: rate(celery_task_failures_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
          service: workers
        annotations:
          summary: "High worker failure rate"
          description: "Worker failure rate is above 10%"

  - name: system_alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
          service: system
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for 5 minutes"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
        for: 5m
        labels:
          severity: warning
          service: system
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 80% for 5 minutes"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
        for: 5m
        labels:
          severity: critical
          service: system
        annotations:
          summary: "Disk space low"
          description: "Disk space is below 20%"

  - name: container_alerts
    interval: 30s
    rules:
      - alert: ContainerDown
        expr: up{job="cadvisor"} == 0
        for: 1m
        labels:
          severity: critical
          service: containers
        annotations:
          summary: "Container monitoring down"
          description: "cAdvisor has been down for 1 minute"

      - alert: ContainerHighMemoryUsage
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
          service: containers
        annotations:
          summary: "Container high memory usage"
          description: "Container memory usage is above 90%"
```

### 3. Update docker-compose.yml

Tambahkan service berikut ke docker-compose.yml:

```yaml
# PostgreSQL Exporter
postgres-exporter:
  image: prometheuscommunity/postgres-exporter:latest
  container_name: affiliate-postgres-exporter
  restart: unless-stopped
  environment:
    - DATA_SOURCE_NAME=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable
  depends_on:
    - postgres
  networks:
    - affiliate-network

# Redis Exporter
redis-exporter:
  image: oliver006/redis_exporter:latest
  container_name: affiliate-redis-exporter
  restart: unless-stopped
  environment:
    - REDIS_ADDR=redis://redis:6379
    - REDIS_PASSWORD=${REDIS_PASSWORD}
  depends_on:
    - redis
  networks:
    - affiliate-network

# Node Exporter
node-exporter:
  image: prom/node-exporter:latest
  container_name: affiliate-node-exporter
  restart: unless-stopped
  command:
    - '--path.procfs=/host/proc'
    - '--path.rootfs=/rootfs'
    - '--path.sysfs=/host/sys'
    - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  networks:
    - affiliate-network

# cAdvisor
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  container_name: affiliate-cadvisor
  restart: unless-stopped
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
    - /dev/disk/:/dev/disk:ro
  privileged: true
  networks:
    - affiliate-network
```

## 📈 Setup Grafana

### 1. Buat Konfigurasi Grafana

```bash
# Create Grafana config directories
mkdir -p config/grafana/provisioning/datasources
mkdir -p config/grafana/provisioning/dashboards
mkdir -p config/grafana/dashboards
```

### 2. Konfigurasi Datasource

```yaml
# config/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: "15s"
```

```yaml
# config/grafana/provisioning/datasources/loki.yml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
    jsonData:
      maxLines: 1000
```

### 3. Konfigurasi Dashboard Provider

```yaml
# config/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'Affiliate Automation'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

### 4. Buat Dashboard JSON

```json
// config/grafana/dashboards/overview.json
{
  "dashboard": {
    "title": "Affiliate Automation Overview",
    "uid": "affiliate-overview",
    "panels": [
      {
        "title": "API Requests",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ],
        "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 }
      },
      {
        "title": "API Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, http_request_latency_seconds_bucket)",
            "legendFormat": "95th percentile"
          }
        ],
        "gridPos": { "x": 12, "y": 0, "w": 12, "h": 8 }
      },
      {
        "title": "Worker Queue Length",
        "type": "graph",
        "targets": [
          {
            "expr": "celery_queue_length",
            "legendFormat": "{{queue}}"
          }
        ],
        "gridPos": { "x": 0, "y": 8, "w": 12, "h": 8 }
      },
      {
        "title": "Worker Task Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(celery_tasks_total[5m])",
            "legendFormat": "{{worker}}"
          }
        ],
        "gridPos": { "x": 12, "y": 8, "w": 12, "h": 8 }
      },
      {
        "title": "System CPU",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}"
          }
        ],
        "gridPos": { "x": 0, "y": 16, "w": 8, "h": 8 }
      },
      {
        "title": "System Memory",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
            "legendFormat": "{{instance}}"
          }
        ],
        "gridPos": { "x": 8, "y": 16, "w": 8, "h": 8 }
      },
      {
        "title": "Disk Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - (node_filesystem_avail_bytes{mountpoint=\"/\"} / node_filesystem_size_bytes{mountpoint=\"/\"})) * 100",
            "legendFormat": "{{instance}}"
          }
        ],
        "gridPos": { "x": 16, "y": 16, "w": 8, "h": 8 }
      }
    ]
  }
}
```

## 📝 Setup Loki

### 1. Buat Konfigurasi Loki

```bash
# Create Loki config directory
mkdir -p config/loki
```

```yaml
# config/loki/loki.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 30
  ingestion_burst_size_mb: 60

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
```

## 🔍 Setup Promtail

### 1. Buat Konfigurasi Promtail

```bash
# Create Promtail config directory
mkdir -p config/promtail
```

```yaml
# config/promtail/promtail.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Application logs
  - job_name: api-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: api
          __path__: /var/log/affiliate/api/*.log

  - job_name: worker-ai-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: worker-ai
          __path__: /var/log/affiliate/worker-ai/*.log

  - job_name: worker-render-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: worker-render
          __path__: /var/log/affiliate/worker-render/*.log

  - job_name: worker-upload-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: worker-upload
          __path__: /var/log/affiliate/worker-upload/*.log

  # Docker container logs
  - job_name: docker-logs
    docker:
      max_log_age: 168h
      auto_discover:
        filters:
          - name: label
            values:
              - "com.docker.compose.service"
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: container
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: stream
      - source_labels: ['__meta_docker_compose_service']
        target_label: job
```

## ⏰ Setup Uptime Kuma

### 1. Akses Uptime Kuma

Buka http://your-server-ip:3003

### 2. Setup Initial Admin

1. Create admin account
2. Set username and password

### 3. Add Monitors

Tambahkan monitor untuk setiap service:

**API Service**
- Type: HTTP
- URL: http://api:8000/health
- Interval: 60 seconds
- Notification: Telegram/Email

**n8n**
- Type: HTTP
- URL: http://n8n:5678/healthz
- Interval: 60 seconds

**PostgreSQL**
- Type: TCP
- Hostname: postgres
- Port: 5432
- Interval: 60 seconds

**Redis**
- Type: TCP
- Hostname: redis
- Port: 6379
- Interval: 60 seconds

**Grafana**
- Type: HTTP
- URL: http://grafana:3000/api/health
- Interval: 60 seconds

**Prometheus**
- Type: HTTP
- URL: http://prometheus:9090/-/healthy
- Interval: 60 seconds

### 4. Setup Notifications

**Telegram Notification**
1. Settings → Notifications
2. Add Telegram
3. Configure bot token and chat ID

**Email Notification**
1. Settings → Notifications
2. Add Email
3. Configure SMTP settings

## 🚀 Start Monitoring Stack

```bash
# Start all monitoring services
docker compose up -d prometheus grafana loki promtail

# Start exporters
docker compose up -d postgres-exporter redis-exporter node-exporter cadvisor

# Start Uptime Kuma
docker compose up -d uptime-kuma

# Check status
docker compose ps prometheus grafana loki promtail uptime-kuma
```

## 📊 Akses Monitoring Tools

### Grafana
- URL: http://your-server-ip:3001
- Default credentials: admin/admin
- Change password on first login

### Prometheus
- URL: http://your-server-ip:9090
- View targets: http://your-server-ip:9090/targets
- View alerts: http://your-server-ip:9090/alerts

### Loki
- URL: http://your-server-ip:3100
- API: http://your-server-ip:3100/loki/api/v1

### Uptime Kuma
- URL: http://your-server-ip:3003
- Login with admin credentials

## 🎨 Import Grafana Dashboards

### 1. Download Dashboard JSON

Download dashboard JSON dari:
- https://grafana.com/grafana/dashboards/
- Search for: "Docker", "PostgreSQL", "Redis", "Celery"

### 2. Import Dashboard

1. Buka Grafana → Dashboards → Import
2. Upload JSON file
3. Select datasource
4. Import dashboard

### 3. Recommended Dashboards

- Docker Monitoring (ID: 15141)
- PostgreSQL (ID: 9628)
- Redis (ID: 11835)
- Node Exporter Full (ID: 1860)
- cAdvisor (ID: 14207)

## 🔔 Setup Alerts

### 1. Configure Alertmanager (Optional)

Jika ingin menggunakan Alertmanager untuk alerting:

```yaml
# Add to docker-compose.yml
alertmanager:
  image: prom/alertmanager:latest
  container_name: affiliate-alertmanager
  restart: unless-stopped
  ports:
    - "9093:9093"
  volumes:
    - ./config/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
  networks:
    - affiliate-network
```

```yaml
# config/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'telegram'

receivers:
  - name: 'telegram'
    telegram_configs:
      - bot_token: ${TELEGRAM_BOT_TOKEN}
        chat_id: ${TELEGRAM_CHAT_ID}
        parse_mode: 'HTML'
```

### 2. Configure Grafana Alerts

1. Buka Grafana dashboard
2. Click panel → Edit
3. Tab "Alert"
4. Configure alert conditions
5. Set notification channel

## 📈 Monitoring Best Practices

### 1. Metrics to Monitor

- **API Health**: Response time, error rate, request rate
- **Database**: Connection count, query performance, replication lag
- **Cache**: Hit rate, memory usage, eviction rate
- **Workers**: Queue length, task rate, failure rate
- **System**: CPU, memory, disk, network
- **Containers**: Resource usage, restart count

### 2. Log Levels

- **ERROR**: Critical issues requiring immediate attention
- **WARNING**: Potential issues that should be investigated
- **INFO**: Normal operational events
- **DEBUG**: Detailed information for troubleshooting

### 3. Alert Thresholds

Set appropriate thresholds based on your requirements:

- API latency: < 500ms (warning), < 1s (critical)
- Error rate: < 1% (warning), < 5% (critical)
- CPU usage: < 70% (warning), < 90% (critical)
- Memory usage: < 80% (warning), < 95% (critical)
- Disk usage: < 80% (warning), < 90% (critical)

### 4. Retention Policies

- Metrics: 15 days retention
- Logs: 7 days retention
- Alerts: 30 days retention

## 🧪 Testing Monitoring

### Test Prometheus

```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Query metrics
curl http://localhost:9090/api/v1/query?query=up

# Check alerts
curl http://localhost:9090/api/v1/alerts
```

### Test Grafana

```bash
# Check Grafana health
curl http://localhost:3001/api/health

# Test datasource connection
# Via Grafana UI → Configuration → Datasources
```

### Test Loki

```bash
# Query logs
curl -G "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="api"}' \
  --data-urlencode 'start=2024-01-01T00:00:00Z' \
  --data-urlencode 'end=2024-01-01T01:00:00Z' \
  --data-urlencode 'limit=100'
```

### Test Uptime Kuma

1. Buka Uptime Kuma dashboard
2. Check all monitors status
3. Trigger test notification
4. Verify notification received

## 📚 Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Uptime Kuma Documentation](https://github.com/louislam/uptime-kuma/wiki)
