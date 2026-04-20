# RabbitMQ

## What is RabbitMQ

**RabbitMQ** is a message broker. Its job is to receive a message from one service and deliver it to another.

---

## Problem it solves

Without a broker, services communicate directly:

```
Service A  →  HTTP request  →  Service B
```

If Service B is down — the request is lost. If Service B is slow — Service A waits.

With a broker:

```
Service A  →  [RabbitMQ]  →  Service B
```

Service A sends and forgets. RabbitMQ holds the message until Service B is ready.

---

## Key concepts

| Concept | What it is |
|---|---|
| **Producer** | Service that sends messages |
| **Consumer** | Service that receives messages |
| **Queue** | Storage where messages wait |
| **Exchange** | Router — decides which queue to send to |
| **Vhost** | Isolated namespace (like a database in PostgreSQL) |

---

## When to use

✅ **Good fit when:**
- Async processing is needed (emails, SMS, notifications)
- One service produces tasks, multiple services process them
- Services are written in different languages
- Message delivery guarantee is required

❌ **Not a good fit when:**
- You need an immediate response → use HTTP/gRPC
- You need to store and replay large event streams → use Kafka
- Simple communication between two services with low load → overkill
