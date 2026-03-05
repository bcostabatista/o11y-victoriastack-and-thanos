# Observability Stack

This repository contains all Kubernetes manifests for deploying an observability stack with either VictoriaMetrics + VictoriaLogs or Thanos. The stack covers metrics collection, long-term storage, and visualization, following the Four Golden Signals and a defined SLO/SLA policy.

## Accessing Grafana

**[https://grafana.$domain.com](https://grafana.$domain.com)**

All dashboards are available there. No port-forward required.

---

## Architecture Overview

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

---

## Components

### Prometheus `v2.51.2`
The metrics collector. Runs as a StatefulSet with persistent storage and scrapes all configured targets at regular intervals. It does **not** store data long-term locally - everything is shipped to Thanos Receive via `remote_write`.

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

### Thanos `v0.41.0`
Thanos extends Prometheus with long-term storage and global query across multiple instances. It is deployed as four separate components:

| Component | Role |
|-----------|------|
| **Receive** | Accepts `remote_write` from Prometheus and writes TSDB blocks to object storage |
| **Store Gateway** | Serves historical blocks from object storage to Thanos Query |
| **Query** | Unified query layer - fans out PromQL to Receive + Store Gateway and deduplicates results |
| **Compactor** | Periodically merges and downsamples blocks in object storage to reduce size and improve query performance |

Object storage backend: **DigitalOcean Spaces** (S3-compatible).

- Manifests: [thanos/](thanos/)

### Grafana `v11.2.0`
The visualization layer. Connects to Thanos Query as its Prometheus-compatible datasource, giving it access to both recent and historical metrics through a single endpoint.

Includes two dashboards:
- **SLO/SLA Dashboard** (`grafana/slo-dashboard.json`): SLO compliance, error budget, Four Golden Signals, infrastructure health, Kubernetes state, and observability stack health.
- **Default Dashboard** (`grafana/default-dashboard.json`): General cluster overview.

- Manifest: [grafana/grafana.yaml](grafana/grafana.yaml)
- Ingress: [grafana/grafana-ingress.yaml](grafana/grafana-ingress.yaml)

---

## Metrics Collection Flow

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

| Resolution | Retention |
|------------|-----------|
| Raw (per-scrape) | 30 days |
| 5-minute downsampled | 180 days |
| 1-hour downsampled | 2 years |

Downsampling is applied automatically by Thanos Compactor.
