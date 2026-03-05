# Observability Stack

This repository contains all Kubernetes manifests for deploying an observability stack covering metrics collection, long-term storage, log aggregation, and visualization. It follows the Four Golden Signals and a defined SLO/SLA policy.

Two backend options are available. Choose one based on your operational needs:

- **Thanos** - production-grade, long-term storage via object storage, multi-component
- **VictoriaMetrics** *(optional)* - single-node, self-contained, much simpler to operate

---

## Choosing a Stack

| | Thanos | VictoriaMetrics (single-node) |
|---|---|---|
| **Setup complexity** | High - 4 components + object storage | Low - 2 deployments |
| **Resource usage** | Higher | Very low |
| **Long-term storage** | Unlimited (DigitalOcean Spaces/S3) | Local disk only (20Gi, 90d retention) |
| **Log aggregation** | Not included | VictoriaLogs included |
| **HA / deduplication** | Built-in | Not available in single-node |
| **Multi-cluster query** | Yes (Thanos Query fans out) | No |
| **Downsampling** | Yes (Compactor, up to 2 years) | No |
| **Operational overhead** | Higher | Minimal |
| **Prometheus-compatible API** | Yes (Thanos Query) | Yes |

**Use Thanos** if you need long retention (months/years), multi-cluster visibility, or high availability.

**Use VictoriaMetrics** if you want a simple, low-cost stack that is fast to deploy and easy to maintain - ideal for single-cluster setups where 90 days of metrics and 30 days of logs are sufficient.

---

## Accessing Grafana

**[https://grafana.$domain.com](https://grafana.$domain.com)**

All dashboards are available there. No port-forward required.

---

## Architecture Overview

### Option A: Thanos

```
Cluster Nodes
  └── node-exporter (DaemonSet)          # hardware & OS metrics per node
  └── kube-state-metrics (Deployment)    # Kubernetes object state metrics

Prometheus (StatefulSet)
  ├── scrapes node-exporter
  ├── scrapes kube-state-metrics
  ├── scrapes itself
  └── remote_write ──────────────────────► Thanos Receive

Thanos Receive
  ├── accepts remote_write from Prometheus
  ├── exposes StoreAPI (gRPC :10901)
  └── writes blocks ─────────────────────► DigitalOcean Spaces (S3)

Thanos Store Gateway
  ├── reads historical blocks from Spaces
  └── exposes StoreAPI (gRPC :10901)

Thanos Compactor (CronJob, every 15 min)
  ├── compacts and deduplicates blocks in Spaces
  └── applies downsampling retention:
        raw → 30 days
        5m  → 180 days
        1h  → 2 years

Thanos Query
  ├── fans out queries to Thanos Receive + Store Gateway
  └── serves as unified datasource for Grafana

Grafana
  ├── datasource: Thanos Query
  └── exposed at https://grafana.$domain.com (Traefik + cert-manager)
```

### Option B: VictoriaMetrics (single-node)

```
Cluster Nodes
  └── node-exporter (DaemonSet)          # hardware & OS metrics per node
  └── kube-state-metrics (Deployment)    # Kubernetes object state metrics
  └── promtail (DaemonSet)               # collects pod logs from /var/log

Prometheus (StatefulSet)
  ├── scrapes node-exporter
  ├── scrapes kube-state-metrics
  ├── scrapes itself
  └── remote_write ──────────────────────► VictoriaMetrics :8428

VictoriaMetrics (Deployment, single-node)
  ├── accepts remote_write from Prometheus
  ├── stores metrics locally (20Gi PVC, 90d retention)
  └── exposes Prometheus-compatible query API

Promtail (DaemonSet)
  └── ships pod logs ─────────────────────► VictoriaLogs :9428 (Loki API)

VictoriaLogs (Deployment, single-node)
  ├── receives logs from Promtail
  └── stores logs locally (50Gi PVC, 30d retention)

Grafana
  ├── datasource: VictoriaMetrics (Prometheus-compatible)
  ├── datasource: VictoriaLogs (requires victoriametrics-logs-datasource plugin)
  └── exposed at https://grafana.$domain.com (Traefik + cert-manager)
```

---

## Components

### Prometheus `v2.51.2`
The metrics collector. Runs as a StatefulSet with persistent storage and scrapes all configured targets at regular intervals. It does **not** store data long-term locally - everything is shipped via `remote_write` to either Thanos Receive or VictoriaMetrics depending on which stack is active.

- Config: [prometheus/prometheus-configmap.yaml](prometheus/prometheus-configmap.yaml)
- Manifest: [prometheus/prometheus.yaml](prometheus/prometheus.yaml)
- RBAC: [prometheus/prometheus-rbac.yaml](prometheus/prometheus-rbac.yaml)
- External labels: `cluster=do-prod`, `env=prod`

### node-exporter `v1.8.1`
Runs as a DaemonSet - one pod per node. Exports hardware and OS-level metrics: CPU, memory, disk I/O, network, filesystem usage. These feed the infrastructure panels in Grafana.

- Manifest: [exporters/node-exporter.yaml](exporters/node-exporter.yaml)
- Scrape job: `node-exporter`

### kube-state-metrics `v2.13.0`
A single Deployment that listens to the Kubernetes API and exposes state metrics for cluster objects: Deployments, Pods, DaemonSets, CronJobs, PersistentVolumes, etc. It answers questions like "how many replicas are unavailable?" or "is this CronJob failing?".

- Manifest: [kube/kube-state-metrics.yaml](kube/kube-state-metrics.yaml)
- Scrape job: `kube-state-metrics`

### metrics-server
Provides real-time resource usage (CPU/memory) to the Kubernetes API (`kubectl top`). Not scraped by Prometheus - used internally by the cluster for HPA and resource reporting.

- Manifest: [kube/metrics-server.yaml](kube/metrics-server.yaml)

### Thanos `v0.41.0` *(Option A)*
Thanos extends Prometheus with long-term storage and global query across multiple instances. It is deployed as four separate components:

| Component | Role |
|-----------|------|
| **Receive** | Accepts `remote_write` from Prometheus and writes TSDB blocks to object storage |
| **Store Gateway** | Serves historical blocks from object storage to Thanos Query |
| **Query** | Unified query layer - fans out PromQL to Receive + Store Gateway and deduplicates results |
| **Compactor** | Periodically merges and downsamples blocks in object storage to reduce size and improve query performance |

Object storage backend: **DigitalOcean Spaces** (S3-compatible).

- Manifests: [thanos/](thanos/)

### VictoriaMetrics `v1.112.0` *(Option B)*
A single-node deployment that replaces the entire Thanos stack. Accepts `remote_write` from Prometheus and exposes a fully Prometheus-compatible query API. No object storage, no multiple components - just one Deployment and a PVC.

- 20Gi local storage, 90-day retention
- Port: `:8428`
- Manifest: [victoriastack/metrics.yaml](victoriastack/metrics.yaml)

### VictoriaLogs `v1.21.0` *(Option B)*
A single-node log aggregation backend, part of the VictoriaMetrics stack. Receives logs from Promtail via the Loki-compatible HTTP API. Requires the `victoriametrics-logs-datasource` Grafana plugin - see [victoriastack/README.md](victoriastack/README.md) for setup instructions.

- 50Gi local storage, 30-day retention
- Port: `:9428`
- Manifest: [victoriastack/logs.yaml](victoriastack/logs.yaml)

### Promtail `v3.4.1` *(Option B)*
Runs as a DaemonSet - one pod per node. Reads container logs from `/var/log/pods` and ships them to VictoriaLogs via the Loki push API. Attaches labels for namespace, pod, container, node, and app.

- Manifest: [victoriastack/logs.yaml](victoriastack/logs.yaml) (bundled with VictoriaLogs)

### Grafana `v11.2.0`
The visualization layer. Connects to either Thanos Query or VictoriaMetrics as its Prometheus-compatible datasource.

Includes two dashboards:
- **SLO/SLA Dashboard** (`grafana/slo-dashboard.json`): SLO compliance, error budget, Four Golden Signals, infrastructure health, Kubernetes state, and observability stack health.
- **Default Dashboard** (`grafana/default-dashboard.json`): General cluster overview.

- Manifest: [grafana/grafana.yaml](grafana/grafana.yaml)
- Ingress: [grafana/grafana-ingress.yaml](grafana/grafana-ingress.yaml)

---

## Metrics Collection Flow

### Option A: Thanos

```
[Node hardware / OS]
       │
       ▼
node-exporter :9100
       │
       │   (scrape)
       ▼
[Kubernetes API]
       │
       ▼
kube-state-metrics :8080
       │
       │   (scrape)
       ▼
Prometheus :9090
       │
       │   remote_write
       ▼
Thanos Receive :19291
       │
       ├── StoreAPI (:10901) ◄── Thanos Query
       │
       ▼
DigitalOcean Spaces
       │
       ├── Store Gateway (:10901) ◄── Thanos Query
       │
       └── Compactor (every 15 min, downsampling + dedup)

Thanos Query :9090 / :10902
       │
       │   PromQL
       ▼
Grafana :3000
       │
       ▼
https://grafana.$domain.com
```

### Option B: VictoriaMetrics (single-node)

```
[Node hardware / OS]
       │
       ▼
node-exporter :9100
       │
       │   (scrape)
       ▼
[Kubernetes API]
       │
       ▼
kube-state-metrics :8080
       │
       │   (scrape)
       ▼
Prometheus :9090
       │
       │   remote_write
       ▼
VictoriaMetrics :8428
       │   (20Gi PVC, 90d retention)
       │
       │   PromQL / MetricsQL
       ▼
Grafana :3000 ◄──────────────────────── VictoriaLogs :9428
       │                                      ▲
       ▼                                      │  (Loki push API)
https://grafana.$domain.com          Promtail (DaemonSet)
                                              │
                                     /var/log/pods/* (all nodes)
```

---

## SLO / SLA Policy

| Metric | Target |
|--------|--------|
| SLO (internal) | 99.9% availability |
| SLA (contractual) | 99.5% availability |
| Error Budget | 0.1% / month (~43 minutes) |

Incident response targets:

| Priority | Response | RTO | RPO |
|----------|----------|-----|-----|
| P1 (critical) | 15 min | 1 h | 15 min |
| P2 (high) | 1 h | - | - |
| P3 (low) | 8 h | - | - |

---

## Namespace

All components run in the `opentelemetry` namespace.

```bash
kubectl get all -n opentelemetry
```

---

## Retention Policy

### Thanos

| Resolution | Retention |
|------------|-----------|
| Raw (per-scrape) | 30 days |
| 5-minute downsampled | 180 days |
| 1-hour downsampled | 2 years |

Downsampling is applied automatically by Thanos Compactor.

### VictoriaMetrics

| Signal | Retention |
|--------|-----------|
| Metrics | 90 days (local PVC) |
| Logs | 30 days (local PVC) |

No downsampling. Storage is bounded by the PVC size defined in the manifests.
