# RabbitMQ — Docker

## docker run — quick start for testing

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:4.1-management
```

| Port | Purpose |
|---|---|
| `5672` | AMQP — application connections |
| `15672` | Management UI |

> After starting, UI is available at `http://localhost:15672`  
> Default credentials: `guest` / `guest` (only work from localhost)

---

## docker-compose — persistent setup

### Minimal version

```yaml
services:
  rabbitmq:
    image: rabbitmq:4.1-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

### With environment variables via .env

`.env` file:
```env
RABBITMQ_USER=admin
RABBITMQ_PASS=StrongPassword123
RABBITMQ_VHOST=/myapp
RABBITMQ_ERLANG_COOKIE=secret-erlang-cookie
```

`docker-compose.yml`:
```yaml
services:
  rabbitmq:
    image: rabbitmq:4.1-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS}
      RABBITMQ_DEFAULT_VHOST: ${RABBITMQ_VHOST}
      RABBITMQ_ERLANG_COOKIE: ${RABBITMQ_ERLANG_COOKIE}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  rabbitmq_data:
```

---

## Volumes — how to persist data

Without a volume, all queues and messages are lost on `docker rm`.

```yaml
volumes:
  - rabbitmq_data:/var/lib/rabbitmq  # queues, messages, users
```

Verify volume was created:
```bash
docker volume ls | grep rabbitmq
```

---

## Environment variables

| Variable | What it does | Default |
|---|---|---|
| `RABBITMQ_DEFAULT_USER` | Username on first start | `guest` |
| `RABBITMQ_DEFAULT_PASS` | Password on first start | `guest` |
| `RABBITMQ_DEFAULT_VHOST` | Default vhost | `/` |
| `RABBITMQ_ERLANG_COOKIE` | Clustering secret — must match on all nodes | random |

> `RABBITMQ_DEFAULT_USER` and `RABBITMQ_DEFAULT_PASS` are applied **on first start only**.  
> If a volume already exists — these variables are ignored.

---

## Healthcheck

Check that RabbitMQ is alive:

```bash
docker exec rabbitmq rabbitmq-diagnostics ping
```

Expected response:
```
Ping succeeded if node is up
```

Check status of all components:
```bash
docker exec rabbitmq rabbitmq-diagnostics status
```
