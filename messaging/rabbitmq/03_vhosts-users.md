# RabbitMQ — Vhosts and Users

## What is a Vhost

Vhost (Virtual Host) — an isolated namespace inside RabbitMQ. Think of it like a separate database in PostgreSQL.

Each vhost has its own:
- Queues
- Exchanges
- Bindings

Users from one vhost **cannot see** another vhost.

---

## Permissions — what the three `".*"` mean

When granting permissions, three parameters are specified:

| Parameter | What it allows |
|---|---|
| **configure** `".*"` | Create/delete queues and exchanges |
| **write** `".*"` | Publish messages |
| **read** `".*"` | Consume messages from queues |

---

## Tags for Management UI

By default a new user **cannot log in** to the UI — a tag is required.

| Tag | What they see in UI |
|---|---|
| `management` | Their vhost only — for teams |
| `monitoring` | All vhosts, read-only |
| `administrator` | Full access — admins only |

---

## Option 1 — Linux (bare metal)

RabbitMQ is installed directly on the server, `rabbitmqctl` is available globally.

```bash
# Create vhost
rabbitmqctl add_vhost /bakai-bnpl

# Create user
rabbitmqctl add_user bakai_bnpl_user StrongPassword123

# Grant permissions
rabbitmqctl set_permissions -p /bakai-bnpl bakai_bnpl_user ".*" ".*" ".*"

# Grant UI access
rabbitmqctl set_user_tags bakai_bnpl_user management

# Verify
rabbitmqctl list_vhosts
rabbitmqctl list_users
rabbitmqctl list_permissions -p /bakai-bnpl
```

---

## Option 2 — Docker

RabbitMQ is running in a container, commands are executed via `docker exec`.

```bash
# Create vhost
docker exec rabbitmq rabbitmqctl add_vhost /bakai-bnpl

# Create user
docker exec rabbitmq rabbitmqctl add_user bakai_bnpl_user StrongPassword123

# Grant permissions
docker exec rabbitmq rabbitmqctl set_permissions -p /bakai-bnpl bakai_bnpl_user ".*" ".*" ".*"

# Grant UI access
docker exec rabbitmq rabbitmqctl set_user_tags bakai_bnpl_user management

# Verify
docker exec rabbitmq rabbitmqctl list_vhosts
docker exec rabbitmq rabbitmqctl list_users
docker exec rabbitmq rabbitmqctl list_permissions -p /bakai-bnpl
```

> `rabbitmq` is the container name. Check with: `docker ps`

---

## Option 3 — Kubernetes

RabbitMQ is running in a pod, commands are executed via `kubectl exec`.

```bash
# Get pod name
kubectl get pods -n <namespace> | grep rabbitmq

# Create vhost
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl add_vhost /bakai-bnpl

# Create user
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl add_user bakai_bnpl_user StrongPassword123

# Grant permissions
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl set_permissions -p /bakai-bnpl bakai_bnpl_user ".*" ".*" ".*"

# Grant UI access
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl set_user_tags bakai_bnpl_user management

# Verify
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl list_vhosts
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl list_users
kubectl exec -n <namespace> <pod-name> -- rabbitmqctl list_permissions -p /bakai-bnpl
```

> If using RabbitMQ Cluster Operator — commands can be run on any pod in the cluster.

---

## Generate a strong password

```bash
openssl rand -base64 16
```
