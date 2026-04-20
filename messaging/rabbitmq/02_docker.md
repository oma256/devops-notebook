# RabbitMQ — Docker

## docker run — быстро поднять для теста

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:4.1-management
```

| Порт | Назначение |
|---|---|
| `5672` | AMQP — подключение сервисов |
| `15672` | Management UI |

> После запуска UI доступен на `http://localhost:15672`  
> Дефолтные креды: `guest` / `guest` (работают только с localhost)

---

## docker-compose — постоянный запуск

### Минимальный вариант

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

### С переменными окружения через .env

`.env` файл:
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

## Volumes — как не потерять данные

Без volume при `docker rm` все очереди и сообщения удалятся.

```yaml
volumes:
  - rabbitmq_data:/var/lib/rabbitmq  # очереди, сообщения, пользователи
```

Проверить что volume создан:
```bash
docker volume ls | grep rabbitmq
```

---

## Переменные окружения

| Переменная | Что делает | Дефолт |
|---|---|---|
| `RABBITMQ_DEFAULT_USER` | Имя пользователя при старте | `guest` |
| `RABBITMQ_DEFAULT_PASS` | Пароль пользователя при старте | `guest` |
| `RABBITMQ_DEFAULT_VHOST` | Дефолтный vhost | `/` |
| `RABBITMQ_ERLANG_COOKIE` | Секрет для кластеризации — должен совпадать на всех нодах | случайный |

> `RABBITMQ_DEFAULT_USER` и `RABBITMQ_DEFAULT_PASS` применяются **только при первом запуске**.  
> Если volume уже существует — переменные игнорируются.

---

## Healthcheck

Проверить что RabbitMQ живой:

```bash
docker exec rabbitmq rabbitmq-diagnostics ping
```

Ожидаемый ответ:
```
Ping succeeded if node is up
```

Проверить статус всех компонентов:
```bash
docker exec rabbitmq rabbitmq-diagnostics status
```