# RabbitMQ — Policies

## What is a Policy

A Policy is a rule applied to queues or exchanges. It allows you to configure queue behavior without changing application code.

Policies are applied by **queue name pattern** — you can configure all queues at once or only specific ones.

---

## TTL — auto-delete old messages

A message lives in the queue no longer than the specified time. If the consumer didn't pick it up in time — the message is deleted (or sent to DLQ).

**Via CLI:**
```bash
# Messages live for a maximum of 1 hour (3600000 ms)
docker exec rabbitmq rabbitmqctl set_policy ttl-policy ".*" \
  '{"message-ttl": 3600000}' \
  --apply-to queues
```

**Via UI:**
```
Admin → Policies → Add policy
Name:       ttl-policy
Pattern:    .*
Apply to:   Queues
Definition: message-ttl = 3600000
```

---

## Max Length — limit queue size

Limits the maximum number of messages in a queue. When the limit is reached — old messages are dropped (or sent to DLQ).

**Via CLI:**
```bash
# Maximum 10000 messages in a queue
docker exec rabbitmq rabbitmqctl set_policy max-length-policy ".*" \
  '{"max-length": 10000}' \
  --apply-to queues
```

**Combine TTL and Max Length:**
```bash
docker exec rabbitmq rabbitmqctl set_policy combined-policy ".*" \
  '{"message-ttl": 3600000, "max-length": 10000}' \
  --apply-to queues
```

---

## Dead Letter Queue (DLQ)

A queue where messages end up when they could not be processed:
- Message rejected by consumer (`basic.reject` / `basic.nack`)
- TTL expired
- Queue reached Max Length

```
Producer → [main-queue] → Consumer (crashed/rejected)
                ↓
          [main-queue.dlq]  ← message ends up here
```

### Setting up DLQ

**Step 1 — Create the DLQ queue:**

Via UI: `Queues → Add queue → Name: main-queue.dlq`

**Step 2 — Apply DLQ policy to the main queue:**
```bash
docker exec rabbitmq rabbitmqctl set_policy dlq-policy "^main-queue$" \
  '{"dead-letter-exchange": "", "dead-letter-routing-key": "main-queue.dlq"}' \
  --apply-to queues
```

**All together — TTL + Max Length + DLQ:**
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

## Pattern reference

| Pattern | Which queues it applies to |
|---|---|
| `.*` | All queues |
| `^bnpl-` | Queues starting with `bnpl-` |
| `^main-queue$` | Only the queue named `main-queue` |

---

## Managing policies

```bash
# List all policies
docker exec rabbitmq rabbitmqctl list_policies

# List policies in a specific vhost
docker exec rabbitmq rabbitmqctl list_policies -p /bakai-bnpl

# Delete a policy
docker exec rabbitmq rabbitmqctl clear_policy ttl-policy
```
