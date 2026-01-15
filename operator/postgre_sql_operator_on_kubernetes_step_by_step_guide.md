# PostgreSQL on Kubernetes using an Operator (CloudNativePG)

This document explains **how to deploy, operate, and govern a PostgreSQL database on Kubernetes using an operator-based approach**, suitable for **large databases (2TB+)** and **enterprise production systems** such as SaaS platforms and internal business applications.

---

## 1. Why Use an Operator?

Running PostgreSQL directly with a StatefulSet works for small or non-critical workloads, but it becomes risky at scale.

A **PostgreSQL Operator**:
- Understands PostgreSQL internals
- Manages replication and failover
- Automates backups and PITR
- Handles upgrades safely
- Creates and manages Kubernetes resources automatically

**Key idea:**
> You declare *what you want* (a Postgres cluster), the operator decides *how to run it safely*.

---

## 2. High-Level Architecture

```
Application Pods
   |
   |  (Service: read/write)
   v
Postgres Primary Pod  <---- Replicas
   |
   |  (PVC per pod)
   v
Persistent Storage (EBS / Cinder / Ceph / SSD)

Backups -> Object Storage (S3 or S3-compatible)
```

---

## 3. Prerequisites

### 3.1 Kubernetes Cluster
- Kubernetes v1.25+
- CSI storage driver installed (EBS, Cinder, Ceph, etc.)
- StorageClass available (e.g., `fast-ssd`)

### 3.2 Install CloudNativePG Operator (One Time)

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

helm install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace
```

This installs:
- Operator controller
- CRDs (Custom Resource Definitions)
- Webhooks

---

## 4. Namespace Setup

```bash
kubectl create namespace database
```

---

## 5. Secrets Management

### 5.1 PostgreSQL Credentials

Create a Kubernetes Secret for database credentials. In enterprise environments, **source these from a vault** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, etc.) and **sync into Kubernetes** via a secrets operator or CSI driver.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: database
type: Opaque
stringData:
  username: postgres
  password: strong-password
```

**Notes:**
- Secrets are injected securely by the operator
- Never commit plaintext passwords to git
- You can also create this Secret manually when needed, but prefer automated sync from a vault

### 5.2 Vault-Backed Secrets (Recommended)

Common patterns:
- **External Secrets Operator**: syncs vault secrets to Kubernetes `Secret` objects
- **Secrets Store CSI Driver**: mounts secrets into pods; then you can mirror to Kubernetes Secrets if required by the operator

Regardless of the method, ensure the resulting Secret name matches `superuserSecret.name` and contains `username` and `password` keys.

---

## 6. PostgreSQL Operator Cluster YAML

This single YAML defines the entire database cluster.

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: customer-postgres
  namespace: database
spec:
  instances: 3   # 1 primary + 2 replicas

  imageName: ghcr.io/cloudnative-pg/postgresql:15

  storage:
    size: 3Ti
    storageClass: fast-ssd

  bootstrap:
    initdb:
      database: customer_db
      owner: postgres

  superuserSecret:
    name: postgres-secret

  resources:
    requests:
      cpu: "4"
      memory: "16Gi"
    limits:
      cpu: "8"
      memory: "32Gi"

  postgresql:
    parameters:
      shared_buffers: "8GB"
      max_connections: "500"
      wal_keep_size: "4GB"

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: cnpg.io/cluster
                operator: In
                values:
                  - customer-postgres
          topologyKey: kubernetes.io/hostname
```

---

## 7. How Storage Works (Important)

You **do not mount volumes manually**.

What happens:
1. Operator creates PVCs automatically
2. PVCs reference the specified StorageClass
3. CSI driver provisions real disks
4. Volumes are mounted at `/var/lib/postgresql/data`

Each instance (primary + replicas) gets its **own volume**.

---

## 8. Networking & Service Discovery

The operator creates Services automatically:

- `customer-postgres-rw` → primary (read/write)
- `customer-postgres-ro` → replicas (read-only)

Applications connect using:

```
host: customer-postgres-rw.database.svc.cluster.local
port: 5432
```

No Ingress or NodePort is required.

---

## 9. Backups & Point-in-Time Recovery (PITR)

### 9.1 S3 Credentials Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-secret
  namespace: database
type: Opaque
stringData:
  accessKey: ACCESS_KEY
  secretKey: SECRET_KEY
```

### 9.2 Backup Configuration

```yaml
backup:
  barmanObjectStore:
    destinationPath: s3://pg-backups/customer
    s3Credentials:
      accessKeyId:
        name: s3-secret
        key: accessKey
      secretAccessKey:
        name: s3-secret
        key: secretKey
  retentionPolicy: "30d"
```

**What the operator does:**
- Runs backup pods
- Streams full backups and WAL
- Enables PITR

---

## 10. Data Cleanup & Maintenance

The operator does **not** manage business data lifecycle.

Use **Kubernetes CronJobs** for:
- Dropping old partitions
- Deleting expired metadata
- Cleaning orphaned records

Example tasks:
- Remove customer data older than retention
- Validate S3 vs DB consistency

---

## 11. Scaling & Failover

### Scale Replicas

```yaml
instances: 5
```

Operator handles:
- Replica creation
- Resynchronization

### Failover

If the primary node fails:
1. Operator detects failure
2. Promotes a replica
3. Updates RW service
4. Applications reconnect automatically

No manual action required.

---

## 12. What the Operator Manages vs What You Manage

### Operator Manages
- StatefulSets
- PVCs
- Replication
- Failover
- Backups
- Services

### You Manage
- Schema design
- Partitioning strategy
- Cleanup logic
- Performance tuning
- Capacity planning

---

## 13. Common Best Practices

- Use **time-based partitioning** for large tables
- Prefer **DROP PARTITION** over DELETE
- Monitor:
  - Replication lag
  - Disk usage
  - WAL growth
  - Autovacuum
- Test restores regularly
- Rotate secrets and enforce least-privilege access to backups

---

## 14. Enterprise Production Enhancements

- **Multi-AZ placement**: spread primaries/replicas across zones and nodes
- **TLS everywhere**: require encrypted connections between apps and Postgres
- **Connection pooling**: use PgBouncer for high-concurrency workloads
- **Backups + DR**: verify PITR, cross-region copies, and restore drills
- **Observability**: metrics, logs, alerts (replication lag, disk, WAL)
- **Security hardening**: network policies, RBAC, audit logging, secret rotation
- **Capacity planning**: set growth alarms, test vacuum/maintenance windows
- **Upgrade process**: test minor/major upgrades in staging before production

---

## 15. Observability Stack (Recommended)

- **Metrics**: Prometheus + Grafana, scrape CNPG and Postgres metrics
- **Logs**: Loki or Elasticsearch/Opensearch with Fluent Bit
- **Tracing**: OpenTelemetry + Tempo/Jaeger for app-to-DB latency
- **Alerting**: Alertmanager with SLO-based alerts (replication lag, disk, WAL)
- **SQL insights**: `pg_stat_statements`, slow query logs, and `auto_explain`

---

## 16. Validate the Database with `psql`

Use `psql` to confirm connectivity and basic read/write operations.

### 16.1 Connect to the Primary Pod

```bash
PRIMARY_POD=$(kubectl get pods -n database -l cnpg.io/cluster=customer-postgres,cnpg.io/role=primary \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it -n database "$PRIMARY_POD" -- psql -U postgres -d customer_db
```

### 16.2 Create Table and Insert Sample Data

```sql
CREATE TABLE IF NOT EXISTS demo_events (
  id SERIAL PRIMARY KEY,
  customer_id TEXT NOT NULL,
  event_type TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO demo_events (customer_id, event_type)
VALUES
  ('cust-001', 'signup'),
  ('cust-002', 'payment'),
  ('cust-001', 'support_ticket');
```

### 16.3 View Table and Query Data

```sql
\dt

SELECT * FROM demo_events ORDER BY created_at DESC;

SELECT customer_id, count(*) AS event_count
FROM demo_events
GROUP BY customer_id
ORDER BY event_count DESC;
```

If these commands succeed, the database is operational and accepting writes.

---

## 17. UI-Based SQL Tools (Developer Access)

Common tools:
- **DBeaver** (cross-platform)
- **DataGrip** (JetBrains)
- **pgAdmin** (PostgreSQL GUI)
- **TablePlus** / **SQLPro**

**Do not expose PostgreSQL publicly**. UI tools must connect **only from internal company networks using private IPs**.

- **Private networking (required)**: VPN, VPC peering, or bastion host
  - Keep the service internal (`ClusterIP`) or use an **internal** LoadBalancer
  - Use private DNS `customer-postgres-rw.database.svc.cluster.local` or a private IP

- **Port-forward (temporary)**: use only from corporate VPN/internal network
  ```bash
  kubectl port-forward -n database svc/customer-postgres-rw 5432:5432
  ```
  Then connect your UI tool to `localhost:5432` with the same username/password.

- **Network policies + RBAC**: restrict which namespaces and pods can reach the DB

If UI access is required regularly, prefer a **bastion + SSH tunnel** or **VPN** instead of opening the service externally.

### pgAdmin Connection Example (Internal Only)

Use the following values in pgAdmin (all must resolve to private IPs):

- **Name**: `customer-postgres` (any label)
- **Host**: `customer-postgres-rw.database.svc.cluster.local` or a private IP
- **Port**: `5432`
- **Maintenance DB**: `customer_db`
- **Username**: `postgres`
- **Password**: from `postgres-secret`
- **SSL Mode**: `require` (or `verify-full` if you have internal CA certs)

---

## 18. Enterprise Operational Checklist

- Define RPO/RTO targets and validate PITR restore times
- Enforce multi-AZ scheduling and anti-affinity
- Automate secret rotation via vault integrations
- Run periodic restore drills and failover exercises
- Track capacity forecasts and storage growth

---

## 19. Mental Model

```
You declare intent
↓
Operator creates Kubernetes resources
↓
Kubernetes provisions storage
↓
PostgreSQL runs safely
```

---

**End of document**

