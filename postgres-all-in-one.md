# PostgreSQL on Kubernetes (All-in-One YAML)

This document describes how to deploy a **single-instance PostgreSQL** database using the all-in-one manifest at `postgres-all-in-one.yaml`. This is a **non-operator** setup intended for simple environments and internal workloads.

---

## 1. What This YAML Deploys

`postgres-all-in-one.yaml` creates:
- **ConfigMap** (`postgres-config`) for non-sensitive settings
- **Secret** (`postgres-secret`) for credentials
- **PVC** (`postgres-data`) for persistent storage
- **Service** (`postgres`) for stable networking
- **StatefulSet** (`postgres`) running PostgreSQL 15
- **CronJob** (`postgres-cleanup`) for daily cleanup

---

## 2. Prerequisites

- Kubernetes v1.25+
- A `StorageClass` named `fast-ssd` (or update the YAML)
- Namespace `database`

Create the namespace:
```bash
kubectl create namespace database
```

---

## 3. Apply the Manifest

```bash
kubectl apply -f postgres-all-in-one.yaml
```

---

## 4. Verify the Deployment

```bash
kubectl get pods -n database
kubectl get pvc -n database
kubectl get svc -n database
```

Wait until the StatefulSet pod is `Running` and `Ready`.

---

## 5. Access the Database

Use port-forward for internal developer access:
```bash
kubectl port-forward -n database svc/postgres 5432:5432
```

Connect with `psql`:
```bash
psql -h localhost -p 5432 -U postgres -d dvr_metadata
```

> The database name and credentials come from the ConfigMap/Secret in the YAML.  
> Update them if you want a different database name.

---

## 6. Cleanup CronJob

The CronJob runs daily at 2 AM and executes:
```sql
DELETE FROM video_metadata
WHERE expires_at < now();
```

Update the query to match your schema and retention requirements.

---

## 7. Important Notes

- **Single instance only**: no replication, no automated failover
- **No operator**: upgrades and backups must be handled manually
- **Internal only**: keep the Service as `ClusterIP` to avoid public exposure
- **Secrets**: the Secret uses base64-encoded values; use a vault in production

---

## 8. Customize Common Settings

Update these in the YAML as needed:
- `POSTGRES_DB` in the ConfigMap
- `POSTGRES_USER` / `POSTGRES_PASSWORD` in the Secret
- `storageClassName` and PVC size
- CPU/memory resources
- Cleanup schedule and SQL

---

**End of document**
