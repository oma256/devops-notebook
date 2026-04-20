# RabbitMQ — Clustering

## How a cluster works

Unlike PostgreSQL + Patroni where there is one primary and the rest are replicas — in RabbitMQ **all nodes are equal**. Any node accepts connections and processes messages.

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
| Type | Active/Passive | Active/Active |
| Failover | Via etcd/consul | Built-in |
| Shared secret | — | Erlang Cookie |
| Minimum nodes | 2 | 3 (for quorum) |

> **Erlang Cookie** — a shared secret on all nodes. Without the same cookie nodes will not join the cluster.

---

## Quorum Queues — why it matters

By default a queue lives on only one node. If that node goes down — the queue is unavailable.

**Quorum Queues** solve this — the queue is replicated to a majority of nodes (quorum). They use Raft consensus, which is the closest analogy to Patroni.

```python
# Specify queue type as quorum when declaring
channel.queue_declare(
    queue='my-queue',
    arguments={'x-queue-type': 'quorum'}
)
```

| | Classic Queue | Quorum Queue |
|---|---|---|
| Replication | No (by default) | Yes (Raft) |
| Delivery guarantee | Weak | Strong |
| Performance | Higher | Slightly lower |
| Recommended for | Dev/Test | Production |

---

## Option 1 — Linux (bare metal)

RabbitMQ is installed on each node. The same Erlang Cookie must be set on all servers.

```bash
# Set the same cookie on all nodes
echo "secret-erlang-cookie" > /var/lib/rabbitmq/.erlang.cookie
chmod 400 /var/lib/rabbitmq/.erlang.cookie
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

# Restart RabbitMQ on all nodes
systemctl restart rabbitmq-server

# On node2 and node3 — join the cluster
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# Verify cluster
rabbitmqctl cluster_status
```

---

## Option 2 — Docker Compose

Three containers with the same `RABBITMQ_ERLANG_COOKIE`.

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

Join node2 and node3 to the cluster:
```bash
# node2
docker exec rabbitmq2 rabbitmqctl stop_app
docker exec rabbitmq2 rabbitmqctl join_cluster rabbit@rabbitmq1
docker exec rabbitmq2 rabbitmqctl start_app

# node3
docker exec rabbitmq3 rabbitmqctl stop_app
docker exec rabbitmq3 rabbitmqctl join_cluster rabbit@rabbitmq1
docker exec rabbitmq3 rabbitmqctl start_app

# Verify
docker exec rabbitmq1 rabbitmqctl cluster_status
```

---

## Option 3 — Kubernetes (Cluster Operator)

The easiest approach — RabbitMQ Cluster Operator manages the cluster and failover automatically.

Install the operator:
```bash
kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml
```

Create a cluster (`rabbitmq-cluster.yaml`):
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

# Verify
kubectl get pods -n messaging
kubectl get rabbitmqcluster -n messaging
```

Get credentials:
```bash
kubectl get secret rabbitmq-cluster-default-user -n messaging -o jsonpath='{.data.username}' | base64 --decode
kubectl get secret rabbitmq-cluster-default-user -n messaging -o jsonpath='{.data.password}' | base64 --decode
```

---

## Useful cluster commands

```bash
# Cluster status
rabbitmqctl cluster_status

# Remove a node from cluster (if node is already unreachable)
rabbitmqctl forget_cluster_node rabbit@node2
```
