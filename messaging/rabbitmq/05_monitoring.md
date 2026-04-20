# RabbitMQ — Мониторинг

## Что мониторить

| Метрика | Почему важно |
|---|---|
| **Queue depth** | Очередь растёт → consumer не справляется |
| **Message rate** | Сколько сообщений в секунду publish/consume |
| **Unacked messages** | Сообщения взяты но не подтверждены — утечка? |
| **Consumer count** | Ноль consumer'ов → сообщения копятся |
| **Memory usage** | RabbitMQ начнёт блокировать publisher'ов при 40% RAM |
| **Disk space** | При нехватке диска RabbitMQ останавливает запись |
| **Connection count** | Резкий рост — проблема с reconnect логикой |

---

## Встроенный Management UI

Доступен на порту `15672`. Показывает всё в реальном времени.

### Где смотреть

**Overview** — общее состояние, memory/disk алерты, общий message rate

**Queues** — самое важное:
- `Ready` — сообщения ждут consumer'а
- `Unacked` — взяты consumer'ом но не подтверждены
- `Total` — сумма

**Connections / Channels** — сколько сервисов подключено

> ⚠️ Если `Ready` постоянно растёт — consumer не справляется с нагрузкой

---

## Prometheus + Grafana

### Шаг 1 — Включить Prometheus plugin

**Linux:**
```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

**Docker:**
```bash
docker exec rabbitmq rabbitmq-plugins enable rabbitmq_prometheus
```

**docker-compose** (автоматически при старте):
```yaml
services:
  rabbitmq:
    image: rabbitmq:4.1-management
    environment:
      RABBITMQ_ENABLED_PLUGINS_FILE: /etc/rabbitmq/enabled_plugins
    volumes:
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins
```

`enabled_plugins`:
```
[rabbitmq_management,rabbitmq_prometheus].
```

После включения метрики доступны на:
```
http://<host>:15692/metrics
```

---

### Шаг 2 — Настроить scrape в Prometheus

`prometheus.yml`:
```yaml
scrape_configs:
  - job_name: rabbitmq
    static_configs:
      - targets:
          - rabbitmq:15692
```

---

### Шаг 3 — Дашборд в Grafana

Готовый официальный дашборд от RabbitMQ team:

```
Grafana → Import → ID: 10991
```

Показывает:
- Queue depth по каждой очереди
- Message rates (publish / deliver / ack)
- Memory и disk usage
- Connection и channel count

---

## Алерты — на что ставить

```yaml
# prometheus alerting rules
groups:
  - name: rabbitmq
    rules:

      # Очередь растёт дольше 5 минут
      - alert: RabbitMQQueueGrowing
        expr: rabbitmq_queue_messages_ready > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Queue {{ $labels.queue }} has too many messages"

      # Нет consumer'ов на очереди
      - alert: RabbitMQNoConsumers
        expr: rabbitmq_queue_consumers == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Queue {{ $labels.queue }} has no consumers"

      # Память выше 80%
      - alert: RabbitMQHighMemory
        expr: rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes > 0.8
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "RabbitMQ memory usage is high"
```

---

## Полезные команды для быстрой диагностики

```bash
# Общий статус
docker exec rabbitmq rabbitmq-diagnostics status

# Список очередей с количеством сообщений
docker exec rabbitmq rabbitmqctl list_queues name messages messages_ready messages_unacknowledged consumers

# Использование памяти
docker exec rabbitmq rabbitmq-diagnostics memory_breakdown

# Проверить что нода живая
docker exec rabbitmq rabbitmq-diagnostics ping
```
