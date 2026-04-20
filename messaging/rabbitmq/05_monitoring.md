# RabbitMQ — Monitoring

## What to monitor

| Metric | Why it matters |
|---|---|
| **Queue depth** | Queue is growing → consumer can't keep up |
| **Message rate** | Messages published/consumed per second |
| **Unacked messages** | Messages taken but not acknowledged — potential leak |
| **Consumer count** | Zero consumers → messages are piling up |
| **Memory usage** | RabbitMQ will block publishers at 40% RAM |
| **Disk space** | RabbitMQ stops writing when disk is full |
| **Connection count** | Sudden spike — reconnect logic issue |

---

## Built-in Management UI

Available on port `15672`. Shows everything in real time.

### Where to look

**Overview** — general health, memory/disk alerts, total message rate

**Queues** — most important:
- `Ready` — messages waiting for a consumer
- `Unacked` — taken by consumer but not acknowledged
- `Total` — sum of both

**Connections / Channels** — how many services are connected

> ⚠️ If `Ready` keeps growing — consumer cannot handle the load

---

## Prometheus + Grafana

### Step 1 — Enable Prometheus plugin

**Linux:**
```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

**Docker:**
```bash
docker exec rabbitmq rabbitmq-plugins enable rabbitmq_prometheus
```

**docker-compose** (automatically on start):
```yaml
services:
  rabbitmq:
    image: rabbitmq:4.1-management
    volumes:
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins
```

`enabled_plugins`:
```
[rabbitmq_management,rabbitmq_prometheus].
```

After enabling, metrics are available at:
```
http://<host>:15692/metrics
```

---

### Step 2 — Configure scrape in Prometheus

`prometheus.yml`:
```yaml
scrape_configs:
  - job_name: rabbitmq
    static_configs:
      - targets:
          - rabbitmq:15692
```

---

### Step 3 — Grafana Dashboard

Official dashboard from the RabbitMQ team:

```
Grafana → Import → ID: 10991
```

Shows:
- Queue depth per queue
- Message rates (publish / deliver / ack)
- Memory and disk usage
- Connection and channel count

---

## Alerts — what to set up

```yaml
# prometheus alerting rules
groups:
  - name: rabbitmq
    rules:

      # Queue growing for more than 5 minutes
      - alert: RabbitMQQueueGrowing
        expr: rabbitmq_queue_messages_ready > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} has too many messages"

      # No consumers on a queue
      - alert: RabbitMQNoConsumers
        expr: rabbitmq_queue_consumers == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Queue {{ $labels.queue }} has no consumers"

      # Memory above 80%
      - alert: RabbitMQHighMemory
        expr: rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes > 0.8
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "RabbitMQ memory usage is high"
```

---

## Useful commands for quick diagnostics

```bash
# General status
docker exec rabbitmq rabbitmq-diagnostics status

# List queues with message counts
docker exec rabbitmq rabbitmqctl list_queues name messages messages_ready messages_unacknowledged consumers

# Memory breakdown
docker exec rabbitmq rabbitmq-diagnostics memory_breakdown

# Check node is alive
docker exec rabbitmq rabbitmq-diagnostics ping
```
