# Cluster Resource Reference

This document catalogues every Kubernetes resource type in use across the platform, organised by namespace and service. It is meant as a quick-reference guide — for authoritative configuration, read the YAML files directly.

---

## How Resources Get Into the Cluster

All resources are managed declaratively through **ArgoCD**. No `kubectl apply` is used in day-to-day operations. The flow is:

```
Git push → ArgoCD detects change → ArgoCD applies manifests to cluster
```

The entry point is the **App-of-Apps** pattern:

| File | Role |
|---|---|
| `bootstrap/root-app.yaml` | Applied once by hand; tells ArgoCD to watch `bootstrap/apps/` |
| `bootstrap/apps/platform-*.yaml` | One ArgoCD Application per platform service |
| `bootstrap/apps/apps-*.yaml` | One ArgoCD Application per workload |

Each Application points ArgoCD at a directory in the repo. ArgoCD reconciles whatever YAML or Helm values are there against the live cluster state.

---

## Namespaces

| Namespace | Purpose |
|---|---|
| `argocd` | ArgoCD itself (installed separately, not managed by this repo) |
| `cert-manager` | TLS certificate automation |
| `ingress-nginx` | Ingress controller |
| `longhorn-system` | Distributed block storage |
| `minio` | S3-compatible object storage |
| `monitoring` | Prometheus, Grafana, Alertmanager, Loki, Alloy |
| `sample-app` | Demo three-tier application |

---

## Platform Services

### cert-manager (`cert-manager` namespace)

**What it does**: Automatically provisions and renews TLS certificates. Every `Ingress` resource in the cluster that carries the annotation `cert-manager.io/cluster-issuer: cluster-ca` gets a signed certificate automatically.

**How it's managed**: Helm chart, `platform/cert-manager/values.yaml`

**Key resources created by the chart**:

| Kind | Name | Notes |
|---|---|---|
| Deployment | `cert-manager` | Main controller, 1 replica |
| Deployment | `cert-manager-cainjector` | Injects CA bundles into webhooks |
| Deployment | `cert-manager-webhook` | Admission webhook for Certificate validation |
| CustomResourceDefinitions | `Certificate`, `ClusterIssuer`, `Issuer`, … | Extend the Kubernetes API to describe certificates |
| ServiceMonitor | `cert-manager` | Prometheus scrape target |

**Key configuration choices**:
- `crds.enabled: true` — CRDs are installed as part of the Helm release rather than separately
- Resources are intentionally small (10m CPU / 32Mi memory requests) — cert-manager is mostly idle

---

### ingress-nginx (`ingress-nginx` namespace)

**What it does**: Receives external HTTP/HTTPS traffic and routes it to the correct Service based on `Ingress` rules. All `Ingress` objects in the cluster use `ingressClassName: nginx`.

**How it's managed**: Helm chart, `platform/nginx-ingress/values.yaml`

**Key resources created by the chart**:

| Kind | Name | Notes |
|---|---|---|
| DaemonSet / Deployment | `ingress-nginx-controller` | The actual NGINX process |
| Service (LoadBalancer) | `ingress-nginx-controller` | Exposes ports 80 and 443 |
| IngressClass | `nginx` | Makes this the default ingress class |

---

### Longhorn (`longhorn-system` namespace)

**What it does**: Provides the `longhorn` StorageClass. When a PersistentVolumeClaim requests `storageClassName: longhorn`, Longhorn automatically creates a replicated volume across the cluster nodes.

**How it's managed**: Helm chart, `platform/longhorn/values.yaml`

**Key resources created by the chart**:

| Kind | Name | Notes |
|---|---|---|
| DaemonSet | `longhorn-manager` | Runs on every node; manages volumes |
| Deployment | `longhorn-ui` | Web dashboard |
| StorageClass | `longhorn` | Default storage class, 3 replicas |
| Ingress | `longhorn-ui` | `longhorn.cluster.lan` (HTTPS) |

**Key configuration choices**:
- `defaultReplicaCount: 3` — every volume is replicated to 3 nodes for high availability
- `persistence.defaultClass: true` — `longhorn` is the cluster's default StorageClass
- **Never run `helm uninstall`** on Longhorn — it would delete all volume data. ArgoCD adoption uses `prune: false` to prevent accidental deletion.

---

### MinIO (`minio` namespace)

**What it does**: S3-compatible object storage. Used now for general storage needs; in Phase 4 it will store ML model artefacts and datasets for MLflow.

**How it's managed**: Helm chart, `platform/minio/values.yaml`

**Key resources created by the chart**:

| Kind | Name | Notes |
|---|---|---|
| Deployment | `minio` | Single-node (standalone) mode |
| Service | `minio` | S3 API on port 9000 |
| Service | `minio-console` | Web UI on port 9001 |
| PersistentVolumeClaim | `minio` | 4 Gi on Longhorn |
| Ingress | `minio` (API) | `minio-api.cluster.lan` (HTTPS) |
| Ingress | `minio` (console) | `minio.cluster.lan` (HTTPS) |

**Key configuration choices**:
- `existingSecret: minio-credentials` — root username and password are never stored in git; they live in a Secret created manually with `kubectl create secret`
- `mode: standalone` — single-node for the lab; a distributed setup would require multiple nodes

---

### kube-prometheus-stack (`monitoring` namespace)

**What it does**: The full observability stack — Prometheus (metrics collection and storage), Grafana (dashboards), and Alertmanager (alert routing and deduplication).

**How it's managed**: Helm chart, `platform/monitoring/values.yaml`

#### Prometheus

| Kind | Name | Notes |
|---|---|---|
| StatefulSet | `prometheus-kube-prometheus-stack-prometheus` | Managed by the Prometheus Operator CRD |
| PersistentVolumeClaim | auto-generated | 3 Gi on Longhorn, 5-day retention |
| Service | `kube-prometheus-stack-prometheus` | Port 9090 |
| Ingress | `prometheus` | `prometheus.cluster.lan` (HTTPS) |

`enableRemoteWriteReceiver: true` is set so that k6 and Alloy can push metrics directly to Prometheus via the remote write API.

#### Grafana

| Kind | Name | Notes |
|---|---|---|
| Deployment | `kube-prometheus-stack-grafana` | |
| PersistentVolumeClaim | `kube-prometheus-stack-grafana` | 1 Gi on Longhorn (dashboard storage) |
| Secret | `grafana-admin-credentials` | **Not in git** — created manually; holds `admin-user` and `admin-password` |
| ConfigMap | datasources | Auto-provisioned: Prometheus (default) + Loki |
| Ingress | `grafana` | `grafana.cluster.lan` (HTTPS) |

**Important**: Grafana is pinned to version `12.0.2` via `grafana.image.tag`. The chart default (13.0.1) has a bug that crashes datasource provisioning. Do not change this without testing.

The Loki datasource UID is fixed at `loki` (configured in values). Dashboard JSON must reference this UID by variable — a hardcoded string `"loki"` will not match the provisioned UID.

#### Alertmanager

| Kind | Name | Notes |
|---|---|---|
| StatefulSet | `alertmanager-kube-prometheus-stack-alertmanager` | Managed by the Alertmanager Operator CRD |
| PersistentVolumeClaim | auto-generated | 512 Mi on Longhorn |
| Service | `kube-prometheus-stack-alertmanager` | |
| Ingress | `alertmanager` | `alertmanager.cluster.lan` (HTTPS) |

**Alert routing**:
- `critical` → `webhook-critical` receiver (`http://alertmanager-webhook.monitoring.svc.cluster.local/critical`), repeat every 1 h
- `warning` → `webhook-warning` receiver, repeat every 4 h
- `info` → silenced (`null-receiver`)
- Inhibition rule: a `critical` alert suppresses the matching `warning` alert

---

### Loki (`monitoring` namespace)

**What it does**: Log aggregation backend. All pod logs collected by Alloy are pushed here and made queryable in Grafana via LogQL.

**How it's managed**: Helm chart, `platform/loki/values.yaml`

**Key resources created by the chart**:

| Kind | Name | Notes |
|---|---|---|
| StatefulSet | `loki` | SingleBinary mode — all Loki components in one pod |
| PersistentVolumeClaim | `storage-loki-0` | 3 Gi on Longhorn |
| Deployment | `loki-gateway` | Nginx gateway that routes read/write to the correct Loki component |
| Service | `loki-gateway` | Port 80 — used by Alloy to push logs and by Grafana to query |

**Key configuration choices**:
- `deploymentMode: SingleBinary` — all roles (ingester, querier, ruler) run in one replica; appropriate for a single-cluster lab
- `loki.storage.type: filesystem` — logs stored on the Longhorn PVC, not in object storage
- `retention_period: 168h` (7 days)
- `loki.auth_enabled: false` — no multi-tenancy; simpler for a single-cluster setup

---

### Grafana Alloy (`monitoring` namespace)

**What it does**: The log and metrics collection agent. Runs as a DaemonSet so there is one instance per node. It:
1. Reads pod log files from `/var/log/pods/` on the host
2. Parses CRI-O/containerd log format
3. Ships logs to Loki
4. Collects node-level OS metrics via the built-in unix exporter
5. Remote-writes those metrics to Prometheus

**How it's managed**: Helm chart, `platform/alloy/values.yaml`

**Key resources created by the chart**:

| Kind | Name | Notes |
|---|---|---|
| DaemonSet | `alloy` | One pod per node |
| ConfigMap | `alloy` | Contains the Alloy River pipeline config |
| ServiceAccount | `alloy` | RBAC to list/watch pods and nodes |

**Key configuration choices**:
- `controller.type: daemonset` — must run on every node to access that node's log files
- Toleration for `node-role.kubernetes.io/control-plane: NoSchedule` — ensures logs are also collected from control plane nodes
- `mounts.varlog: true` — mounts `/var/log` from the host so Alloy can read pod log files
- Position file persisted to a host path (`/var/lib/alloy/positions`) so Alloy does not re-read logs after a pod restart

---

## Workload: sample-app (`sample-app` namespace)

A three-tier demo application: browser → frontend → API → PostgreSQL. It exists to generate realistic traffic and demonstrate the observability stack.

```
[browser / k6 / tgen]
       ↓
  frontend (Python :8080)   — serves HTML, proxies /api/* requests
       ↓
  api (Python :8000)         — REST API, reads/writes tasks table, exposes /metrics
       ↓
  postgres (:5432)           — stores tasks
```

### Namespace

```yaml
# apps/sample-app/namespace.yaml
kind: Namespace
name: sample-app
```

---

### PostgreSQL

**File**: `apps/sample-app/postgres.yaml`

| Kind | Name | Purpose |
|---|---|---|
| PersistentVolumeClaim | `sample-app-postgres-pvc` | 1 Gi on Longhorn — persists the database across pod restarts |
| Deployment | `sample-app-postgres` | Runs PostgreSQL 15 (`quay.io/sclorg/postgresql-15-c9s`) |
| Service | `sample-app-postgres` | Port 5432 — internal only, no Ingress |

**PVC detail**:
- `accessModes: [ReadWriteOnce]` — only one pod can mount it at a time
- `storageClassName: longhorn` — 3-replica block volume
- `strategy: Recreate` on the Deployment — required because RWO volumes cannot be mounted by two pods simultaneously (the old pod must terminate before the new one starts)

**Secret** (`sample-app-postgres-secret`) — **not in git**, created manually:
```
key: db        → database name  (taskdb)
key: user      → database user  (taskuser)
key: password  → database password
```
The Deployment references this Secret via `secretKeyRef` for the `POSTGRESQL_DATABASE`, `POSTGRESQL_USER`, and `POSTGRESQL_PASSWORD` environment variables.

**Security context**: `fsGroup: 26` — the PostgreSQL process runs as UID 26; Kubernetes sets this group on the mounted volume so the process can write to it.

---

### API

**File**: `apps/sample-app/api.yaml`

| Kind | Name | Purpose |
|---|---|---|
| ConfigMap | `sample-app-api-script` | Contains `api.py` — the entire application code |
| Deployment | `sample-app-api` | Runs the API server |
| Service | `sample-app-api` | Port 8000 — internal only |

**What the API does** (`api.py`):
- `GET /api/tasks` — queries PostgreSQL, returns last 50 tasks as JSON
- `POST /api/tasks` — inserts a new task into PostgreSQL
- `GET /api/health` — returns `{"status": "ok", "time": ..., "db_host": ...}`
- `GET /metrics` — exposes Prometheus metrics in text format

**Prometheus metrics exposed**:
- `http_requests_total` (counter, labels: method / endpoint / status)
- `http_request_duration_seconds` (histogram, labels: method / endpoint)

**Image**: `registry.access.redhat.com/ubi9/python-311:latest`
The container installs `pg8000` and `prometheus-client` at startup via pip, then runs `api.py`.

**Probes**:
- Readiness: `GET /api/health`, starts after 40 s (pip install + DB wait takes ~30 s)
- Liveness: `GET /api/health`, starts after 60 s

**Database connection**: reads all DB parameters from the `sample-app-postgres-secret` Secret via environment variables.

---

### Frontend

**File**: `apps/sample-app/frontend.yaml`

| Kind | Name | Purpose |
|---|---|---|
| ConfigMap | `sample-app-frontend-config` | Contains `frontend.py` and `index.html` |
| Deployment | `sample-app-frontend` | Runs the frontend server |
| Service | `sample-app-frontend` | Port 80 → targetPort 8080 |
| Ingress | `sample-app-frontend` | `app.cluster.lan` (HTTPS, cluster-ca TLS) |

**What the frontend does** (`frontend.py`):
- `GET /` or `/index.html` — serves the static HTML page
- `GET /api/*` — proxies request to the API service
- `POST /api/*` — proxies request to the API service

The frontend has no direct database access. It only knows about the API's internal DNS name (`sample-app-api.sample-app.svc.cluster.local:8000`).

**HTML page** (`index.html`):
- Shows a task list, polls `/api/tasks` every 5 seconds
- Displays counts: total tasks in DB, polls this session, errors
- Shows request path info from `/api/health`
- Form to add new tasks via POST

**TLS**: cert-manager annotation `cert-manager.io/cluster-issuer: cluster-ca` causes cert-manager to automatically provision a certificate and store it in Secret `sample-app-tls`.

---

### Traffic Generator (tgen)

**File**: `apps/sample-app/tgen.yaml`

| Kind | Name | Purpose |
|---|---|---|
| ConfigMap | `sample-app-tgen-script` | Contains `tgen.py` |
| Deployment | `sample-app-tgen` | Runs the traffic generator |

**What it does** (`tgen.py`):
- Runs forever in a loop with a 3-second sleep between iterations
- Every cycle: `GET /api/tasks`
- Every 5th cycle: additionally `POST /api/tasks` with a random title from a fixed list
- Logs every action as structured JSON to stdout (picked up by Alloy → Loki)

**Purpose**: Keeps a baseline of traffic flowing at all times so Prometheus always has non-zero metrics and Loki always has logs — useful for testing dashboards and alerts without waiting for real user traffic.

**Image**: `registry.access.redhat.com/ubi9/python-311:latest` — uses only Python stdlib, no pip install needed.

---

### k6 Load Generator

**File**: `apps/sample-app/k6.yaml`

| Kind | Name | Purpose |
|---|---|---|
| ConfigMap | `k6-load-script` | Contains `load.js` — the k6 test script |
| Deployment | `k6-load-generator` | Runs k6 continuously |

**What it does** (`load.js`):
- 10 virtual users running for 720 hours (30 days)
- Per-iteration traffic mix:
  - 70% → `GET /api/tasks`
  - 20% → `POST /api/tasks` (random title)
  - 10% → `GET /api/health`
- **Diurnal sleep pattern** (UTC+1): simulates real human usage — shorter sleeps during business hours, longer at night

  | Time window | Base sleep |
  |---|---|
  | 07:00–09:00 | 6 s (morning ramp-up) |
  | 09:00–18:00 | 3 s (business hours — peak) |
  | 18:00–22:00 | 7.5 s (evening wind-down) |
  | 22:00–07:00 | 18 s (night — minimal) |

**Metrics output**: k6 writes all metrics to Prometheus via remote write (`--out experimental-prometheus-rw`). The env var `K6_PROMETHEUS_RW_SERVER_URL` points at the Prometheus remote write endpoint. `K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM=true` makes k6 latency histograms compatible with Prometheus native histograms.

---

## Secrets Reference

Secrets are **never committed to git**. The `.gitignore` excludes `*.secret.yaml`, `*-secret.yaml`, and the `secrets/` directory. All secrets are created manually with `kubectl create secret` and referenced by name in the manifests.

| Secret name | Namespace | Keys | Used by |
|---|---|---|---|
| `grafana-admin-credentials` | `monitoring` | `admin-user`, `admin-password` | Grafana Deployment (kube-prometheus-stack) |
| `minio-credentials` | `minio` | `rootUser`, `rootPassword` | MinIO Deployment |
| `sample-app-postgres-secret` | `sample-app` | `db`, `user`, `password` | PostgreSQL and API Deployments |
| `argocd-repo-creds` | `argocd` | `url`, `username`, `password` | ArgoCD — credentials to read the GitHub repo |
| `alertmanager-tls` | `monitoring` | `tls.crt`, `tls.key` | Auto-created by cert-manager for Alertmanager Ingress |
| `grafana-tls` | `monitoring` | `tls.crt`, `tls.key` | Auto-created by cert-manager for Grafana Ingress |
| `prometheus-tls` | `monitoring` | `tls.crt`, `tls.key` | Auto-created by cert-manager for Prometheus Ingress |
| `longhorn-tls` | `longhorn-system` | `tls.crt`, `tls.key` | Auto-created by cert-manager for Longhorn Ingress |
| `minio-console-tls` | `minio` | `tls.crt`, `tls.key` | Auto-created by cert-manager for MinIO console Ingress |
| `minio-api-tls` | `minio` | `tls.crt`, `tls.key` | Auto-created by cert-manager for MinIO API Ingress |
| `sample-app-tls` | `sample-app` | `tls.crt`, `tls.key` | Auto-created by cert-manager for frontend Ingress |

---

## ConfigMaps Reference

ConfigMaps store configuration data (non-sensitive). In this cluster they are used to embed application code and Alloy pipeline config.

| ConfigMap | Namespace | Contents |
|---|---|---|
| `sample-app-api-script` | `sample-app` | `api.py` — full API server source |
| `sample-app-frontend-config` | `sample-app` | `frontend.py`, `index.html` |
| `sample-app-tgen-script` | `sample-app` | `tgen.py` — internal traffic generator |
| `k6-load-script` | `sample-app` | `load.js` — k6 diurnal load test |
| `alloy` | `monitoring` | Alloy River pipeline config (log collection + metrics) |
| `kube-prometheus-stack-grafana` | `monitoring` | Grafana datasource provisioning |

---

## PersistentVolumeClaims Reference

All PVCs use `storageClassName: longhorn` (3-replica distributed volumes) and `accessModes: [ReadWriteOnce]`.

| PVC | Namespace | Size | Used by |
|---|---|---|---|
| `sample-app-postgres-pvc` | `sample-app` | 1 Gi | PostgreSQL |
| `prometheus-db-prometheus-…` | `monitoring` | 3 Gi | Prometheus (auto-created by StatefulSet) |
| `alertmanager-db-alertmanager-…` | `monitoring` | 512 Mi | Alertmanager (auto-created by StatefulSet) |
| `kube-prometheus-stack-grafana` | `monitoring` | 1 Gi | Grafana |
| `storage-loki-0` | `monitoring` | 3 Gi | Loki |
| `minio` | `minio` | 4 Gi | MinIO |

---

## Ingress Summary

All Ingresses terminate TLS. Certificates are issued by the `cluster-ca` ClusterIssuer (managed by cert-manager). All hostnames resolve on the local network only (`.cluster.lan`).

| Host | Namespace | Backend service | Port |
|---|---|---|---|
| `app.cluster.lan` | `sample-app` | `sample-app-frontend` | 80 |
| `grafana.cluster.lan` | `monitoring` | `kube-prometheus-stack-grafana` | 80 |
| `prometheus.cluster.lan` | `monitoring` | `kube-prometheus-stack-prometheus` | 9090 |
| `alertmanager.cluster.lan` | `monitoring` | `kube-prometheus-stack-alertmanager` | 9093 |
| `longhorn.cluster.lan` | `longhorn-system` | `longhorn-frontend` | 80 |
| `minio.cluster.lan` | `minio` | `minio` (console) | 9001 |
| `minio-api.cluster.lan` | `minio` | `minio` (API) | 9000 |

---

## ArgoCD Applications Reference

| Application | Namespace watched | Chart / Path | `prune` | `ServerSideApply` |
|---|---|---|---|---|
| `root` | `argocd` | `bootstrap/apps/` (git) | `true` | — |
| `cert-manager` | `cert-manager` | cert-manager Helm chart | `true` | — |
| `nginx-ingress` | `ingress-nginx` | ingress-nginx Helm chart | `true` | — |
| `longhorn` | `longhorn-system` | Longhorn Helm chart | `false` | `true` |
| `minio` | `minio` | MinIO Helm chart | `false` | `true` |
| `monitoring` | `monitoring` | kube-prometheus-stack Helm chart | `false` | `true` |
| `loki` | `monitoring` | Loki Helm chart | `false` | `true` |
| `alloy` | `monitoring` | Grafana Alloy Helm chart | `true` | — |
| `sample-app` | `sample-app` | `apps/sample-app/` (git) | `false` | `true` |

**`prune: false`** means ArgoCD will never delete a resource even if it disappears from the repo. This is set for all stateful services (Longhorn, MinIO, Prometheus, Loki) to prevent accidental data loss.

**`ServerSideApply: true`** is required when adopting pre-existing Helm releases into ArgoCD management — it avoids field ownership conflicts that would otherwise cause `kubectl apply` to fail.
