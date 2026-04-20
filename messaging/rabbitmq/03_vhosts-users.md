# RabbitMQ — Vhosts и пользователи

## Что такое Vhost

Vhost (Virtual Host) — изолированное пространство внутри RabbitMQ. Аналогия: как отдельная база данных в PostgreSQL.

Каждый vhost имеет свои:
- Очереди (Queues)
- Exchange'ы
- Bindings

Пользователи из одного vhost **не видят** другой vhost.

---

## Права — что означают три `".*"`

При выдаче прав указываются три параметра:

| Параметр | Что разрешает |
|---|---|
| **configure** `".*"` | Создавать/удалять очереди и exchange'ы |
| **write** `".*"` | Публиковать сообщения |
| **read** `".*"` | Читать сообщения из очередей |

---

## Теги для Management UI

По умолчанию новый пользователь **не может войти** в UI — нужен тег.

| Тег | Что видит в UI |
|---|---|
| `management` | Только свой vhost — для команд |
| `monitoring` | Все vhost'ы, но без изменений |
| `administrator` | Полный доступ — только для админа |

---

## Вариант 1 — Linux (bare metal)

RabbitMQ установлен напрямую на сервер, `rabbitmqctl` доступен глобально.

```bash
# Создать vhost
rabbitmqctl add_vhost /bakai-bnpl

# Создать пользователя
rabbitmqctl add_user bakai_bnpl_user StrongPassword123

# Выдать права
rabbitmqctl set_permissions -p /bakai-bnpl bakai_bnpl_user ".*" ".*" ".*"

# Дать доступ в UI
rabbitmqctl set_user_tags bakai_bnpl_user management

# Проверка
rabbitmqctl list_vhosts
rabbitmqctl list_users
rabbitmqctl list_permissions -p /bakai-bnpl
```

---

## Вариант 2 — Docker

RabbitMQ запущен в контейнере, команды выполняются через `docker exec`.

```bash
# Создать vhost
docker exec rabbitmq rabbitmqctl add_vhost /bakai-bnpl

# Создать пользователя
docker exec rabbitmq rabbitmqctl add_user bakai_bnpl_user StrongPassword123

# Выдать права
docker exec rabbitmq rabbitmqctl set_permissions -p /bakai-bnpl bakai_bnpl_user ".*" ".*" ".*"

# Дать доступ в UI
docker exec rabbitmq rabbitmqctl set_user_tags bakai_bnpl_user management

# Проверка
docker exec rabbitmq rabbitmqctl list_vhosts
docker exec rabbitmq rabbitmqctl list_users
docker exec rabbitmq rabbitmqctl list_permissions -p /bakai-bnpl
```

> `rabbitmq` — имя контейнера. Проверить: `docker ps`

---

## Вариант 3 — Kubernetes

RabbitMQ запущен в поде, команды выполняются через `kubectl exec`.

```bash
# Узнать имя пода
kubectl get pods -n <namespace> | grep rabbitmq

# Создать vhost
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl add_vhost /bakai-bnpl

# Создать пользователя
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl add_user bakai_bnpl_user StrongPassword123

# Выдать права
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl set_permissions -p /bakai-bnpl bakai_bnpl_user ".*" ".*" ".*"

# Дать доступ в UI
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl set_user_tags bakai_bnpl_user management

# Проверка
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl list_vhosts
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl list_users
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl list_permissions -p /bakai-bnpl
```

> Если используется RabbitMQ Cluster Operator — команды выполняются на любом поде кластера.

---

## Генерация надёжного пароля

```bash
openssl rand -base64 16
```
