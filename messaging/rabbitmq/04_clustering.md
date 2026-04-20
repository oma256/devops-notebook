# RabbitMQ — Кластеризация

## Как устроен кластер

В отличие от PostgreSQL + Patroni где есть один primary и остальные replica — в RabbitMQ **все ноды равноправны**. Любая нода принимает подключения и обрабатывает сообщения.

```
        [Load Balancer / HAProxy]
               /        \
        [Node 1]      [Node 2]      [Node 3]
        rabbit@n1     rabbit@n2     rabbit@n3
              \           |           /
               [Shared Erlang Cookie]
```

| | PostgreSQL + Patroni | RabbitMQ Cluster |
|---|---|---|
| Тип | Active/Passive | Active/Active |
| Failover | Через etcd/consul | Встроенный |
| Общий секрет | — | Erlang Cookie |
| Минимум нод | 2 | 3 (для кворума) |

> **Erlang Cookie** — общий секрет на всех нодах. Без одинакового cookie ноды не объединятся в кластер.

---

## Quorum Queues — почему важно

По умолчанию очередь живёт только на одной ноде. Если нода упала — очередь недоступна.

**Quorum Queues** решают это — очередь реплицируется на большинство нод (кворум). Используют Raft-консенсус, аналогия с Patroni ближе всего именно здесь.

```python
# При создании очереди указать тип quorum
channel.queue_declare(
    queue='my-queue',
    arguments={'x-queue-type': 'quorum'}
)
```

| | Classic Queue | Quorum Queue |
|---|---|---|
| Репликация | Нет (по умолчанию) | Да (Raft) |
| Гарантия доставки | Слабая | Сильная |
| Производительность | Выше | Немного ниже |
| Рекомендуется | Для dev/test | Для production |

---

## Вариант 1 — Linux (bare metal)

На каждой ноде установлен RabbitMQ. Одинаковый Erlang Cookie на всех серверах.

```bash
# На всех нодах — задать одинаковый cookie
echo "secret-erlang-cookie" > /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

# Перезапустить RabbitMQ на всех нодах
systemctl restart rabbitmq-server

# На node2 и node3 — присоединиться к node1
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# Проверить кластер
rabbitmqctl cluster_status
```

---

## Вариант 2 — Docker Compose

Три контейнера с одинаковым `RABBITMQ_ERLANG_COOKIE`.

`.env`:
```env
RABBITMQ_USER=admin
RABBITMQ_PASS=StrongPassword123
RABBITMQ_ERLANG_COOKIE=secret-erlang-cookie
```

`docker-compose.yml`:
```yaml
services:
  rabbitmq1:
    image: rabbitmq:4.1-management
    container_name: rabbitmq1
    hostname: rabbitmq1
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS}
      RABBITMQ_ERLANG_COOKIE: ${RABBITMQ_ERLANG_COOKIE}
      RABBITMQ_NODENAME: rabbit@rabbitmq1
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq1_data:/var/lib/rabbitmq

  rabbitmq2:
    image: rabbitmq:4.1-management
    container_name: rabbitmq2
    hostname: rabbitmq2
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS}
      RABBITMQ_ERLANG_COOKIE: ${RABBITMQ_ERLANG_COOKIE}
      RABBITMQ_NODENAME: rabbit@rabbitmq2
    volumes:
      - rabbitmq2_data:/var/lib/rabbitmq
    depends_on:
      - rabbitmq1

  rabbitmq3:
    image: rabbitmq:4.1-management
    container_name: rabbitmq3
    hostname: rabbitmq3
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS}
      RABBITMQ_ERLANG_COOKIE: ${RABBITMQ_ERLANG_COOKIE}
      RABBITMQ_NODENAME: rabbit@rabbitmq3
    volumes:
      - rabbitmq3_data:/var/lib/rabbitmq
    depends_on:
      - rabbitmq1

volumes:
  rabbitmq1_data:
  rabbitmq2_data:
  rabbitmq3_data:
```

Присоединить node2 и node3 к кластеру:
```bash
# node2
docker exec rabbitmq2 rabbitmqctl stop_app
docker exec rabbitmq2 rabbitmqctl join_cluster rabbit@rabbitmq1
docker exec rabbitmq2 rabbitmqctl start_app

# node3
docker exec rabbitmq3 rabbitmqctl stop_app
docker exec rabbitmq3 rabbitmqctl join_cluster rabbit@rabbitmq1
docker exec rabbitmq3 rabbitmqctl start_app

# Проверить
docker exec rabbitmq1 rabbitmqctl cluster_status
```

---

## Вариант 3 — Kubernetes (Cluster Operator)

Самый простой способ — RabbitMQ Cluster Operator сам управляет кластером и failover.

Установить оператор:
```bash
kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml
```

Создать кластер (`rabbitmq-cluster.yaml`):
```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-cluster
  namespace: messaging
spec:
  replicas: 3
  image: rabbitmq:4.1-management
  persistence:
    storageClassName: standard
    storage: 10Gi
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1
      memory: 2Gi
```

```bash
kubectl apply -f rabbitmq-cluster.yaml

# Проверить
kubectl get pods -n messaging
kubectl get rabbitmqcluster -n messaging
```

Получить креды:
```bash
kubectl get secret rabbitmq-cluster-default-user -n messaging -o jsonpath='{.data.username}' | base64 --decode
kubectl get secret rabbitmq-cluster-default-user -n messaging -o jsonpath='{.data.password}' | base64 --decode
```

---

## Полезные команды для кластера

```bash
# Статус кластера
rabbitmqctl cluster_status

# Список нод
rabbitmqctl cluster_status | grep nodes

# Вывести ноду из кластера (если нода уже недоступна)
rabbitmqctl forget_cluster_node rabbit@node2
```
