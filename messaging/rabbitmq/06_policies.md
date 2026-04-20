# RabbitMQ — Policies

## Что такое Policy

Policy — правило которое применяется к очередям или exchange'ам. Позволяет настроить поведение очереди без изменения кода приложения.

Применяется по **паттерну имени очереди** — можно настроить сразу для всех очередей или только для конкретных.

---

## TTL — автоудаление старых сообщений

Сообщение живёт в очереди не дольше указанного времени. Если consumer не успел забрать — сообщение удаляется (или уходит в DLQ).

**Через CLI:**
```bash
# Сообщения живут максимум 1 час (3600000 мс)
docker exec rabbitmq rabbitmqctl set_policy ttl-policy ".*" \
  '{"message-ttl": 3600000}' \
  --apply-to queues
```

**Через UI:**
```
Admin → Policies → Add policy
Name:       ttl-policy
Pattern:    .*
Apply to:   Queues
Definition: message-ttl = 3600000
```

---

## Max Length — ограничение размера очереди

Ограничивает максимальное количество сообщений в очереди. Когда лимит достигнут — старые сообщения удаляются (или уходят в DLQ).

**Через CLI:**
```bash
# Максимум 10000 сообщений в очереди
docker exec rabbitmq rabbitmqctl set_policy max-length-policy ".*" \
  '{"max-length": 10000}' \
  --apply-to queues
```

**Совместить TTL и Max Length:**
```bash
docker exec rabbitmq rabbitmqctl set_policy combined-policy ".*" \
  '{"message-ttl": 3600000, "max-length": 10000}' \
  --apply-to queues
```

---

## Dead Letter Queue (DLQ)

Очередь куда падают сообщения которые не удалось обработать:
- Сообщение отклонено consumer'ом (`basic.reject` / `basic.nack`)
- Истёк TTL
- Очередь достигла Max Length

```
Producer → [main-queue] → Consumer (упал/отклонил)
                ↓
          [main-queue.dlq]  ← сюда попадает сообщение
```

### Настройка DLQ

**Шаг 1 — Создать DLQ очередь:**
```bash
docker exec rabbitmq rabbitmqctl eval '
  rabbit_amqqueue:declare(
    rabbit_misc:r(<<"/">>, queue, <<"main-queue.dlq">>),
    true, false, [], none, <<"cli">>
  ).
'
```

Или через UI: `Queues → Add queue → Name: main-queue.dlq`

**Шаг 2 — Применить политику DLQ на основную очередь:**
```bash
docker exec rabbitmq rabbitmqctl set_policy dlq-policy "^main-queue$" \
  '{"dead-letter-exchange": "", "dead-letter-routing-key": "main-queue.dlq"}' \
  --apply-to queues
```

**Всё вместе — TTL + Max Length + DLQ:**
```bash
docker exec rabbitmq rabbitmqctl set_policy full-policy "^main-queue$" \
  '{
    "message-ttl": 3600000,
    "max-length": 10000,
    "dead-letter-exchange": "",
    "dead-letter-routing-key": "main-queue.dlq"
  }' \
  --apply-to queues
```

---

## Паттерны применения

| Паттерн | К каким очередям применится |
|---|---|
| `.*` | Все очереди |
| `^bnpl-` | Очереди начинающиеся с `bnpl-` |
| `^main-queue$` | Только очередь `main-queue` |

---

## Управление политиками

```bash
# Список всех политик
docker exec rabbitmq rabbitmqctl list_policies

# Список политик в конкретном vhost
docker exec rabbitmq rabbitmqctl list_policies -p /bakai-bnpl

# Удалить политику
docker exec rabbitmq rabbitmqctl clear_policy ttl-policy
```
