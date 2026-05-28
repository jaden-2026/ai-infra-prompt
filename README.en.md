# AI-Infra

> A self-hosted AI infrastructure platform for heterogeneous compute clusters — unified management of GPU/NPU resources, job scheduling, monitoring, and multi-tenant billing.

[简体中文](./README.md)  · [English](./README.en.md)

---

## What is this

AI-Infra is a **self-hosted AI compute PaaS** that helps enterprises manage heterogeneous GPU/NPU clusters from multiple vendors in their own data centers.

Core problems it solves:

- Multiple teams competing for the same GPU pool with no fair scheduling or isolation
- NVIDIA, Huawei Ascend, Cambricon, Enflame and other hardware each requiring separate operations, with driver and framework compatibility hard to track
- No on-site diagnostic capability when training jobs fail, leading to slow root-cause analysis
- Resource usage impossible to attribute per business unit, making cost accounting difficult

AI-Infra does not touch model business logic. It focuses on three things — **compute resource allocation, release, and monitoring** — and exposes a unified API as the compute foundation for upstream LLM serving platforms, MLOps platforms, and agent development platforms.

---

## Modules

```
┌─────────────────────────────────────────────────────────────┐
│                    Web Console / CLI / SDK                   │
├─────────────────────────────────────────────────────────────┤
│              Open API Gateway (REST · gRPC · Webhook)        │
├──────────────┬──────────────┬───────────────┬───────────────┤
│   Resource   │     Job      │  Monitoring & │  Multi-tenant │
│  Management  │  Lifecycle   │ Observability │  & Billing    │
│ & Scheduling │  Management  │               │               │
├──────────────┴──────────────┴───────────────┴───────────────┤
│              Runtime & Driver Adaptation Layer (HAL)         │
├─────────────────────────────────────────────────────────────┤
│  NVIDIA CUDA  │  Huawei Ascend CANN  │  Cambricon CNRT  │  Enflame TOPSRT │
└─────────────────────────────────────────────────────────────┘
```

### Module 1: Resource Management & Scheduling

- **Automatic device discovery**: Scans and registers GPU/NPU nodes in the cluster, collecting vendor, model, VRAM, and topology metadata. Node Agent heartbeats every 30s; nodes are evicted after 90s timeout.
- **Multi-dimensional resource pools**: Build pools by vendor / model / datacenter / business unit, with AND/OR rule combinations, overcommit ratios, and priority inheritance.
- **Intelligent scheduling engine**:
  - Gang scheduling (atomic all-or-nothing allocation for distributed training)
  - Preemptive scheduling (low-priority release → auto checkpoint save → re-queue)
  - Fair-Share scheduling
  - Topology-aware scoring (NVLink domain +20 / cross-RDMA domain −10)
  - Bin Packing to fill fragmented nodes first
- **Elastic scaling**: K8s Cluster Autoscaler triggered by queue depth; dynamic worker count for elastic training jobs

### Module 2: Runtime & Driver Adaptation Layer

- **Unified HAL**: Abstracts CUDA / CANN / CNRT / TOPSRT runtime differences behind a single API for device enumeration, memory management, and stream synchronization
- **Compatibility matrix**: Three-dimensional validation across driver × framework × device model; incompatible combinations are automatically blocked at job submission
- **Container runtime integration**: Device Plugins expose GPU/NPU resources to Kubernetes; Init Containers auto-inject NCCL/HCCL; `CUDA_VISIBLE_DEVICES` / `ASCEND_VISIBLE_DEVICES` set automatically

### Module 3: Monitoring & Observability

- **Tiered metric collection**: GPU utilization / VRAM / temperature / ECC errors / NVLink bandwidth, with collection frequency tiered by criticality (10s / 30s / 5min)
- **Automated fault diagnosis**: Xid error dictionary (74 NVIDIA Xid codes) + CANN fault code dictionary; on alert, automatically captures on-site snapshots, generates root-cause reports, and creates ops tickets
- **Multi-level alerting**: Warning / Critical / Emergency tiers, On-call escalation chain, same-type alert deduplication within 5min, maintenance silence windows
- **Full-stack observability**: Prometheus metrics + Loki logs + OpenTelemetry tracing, unified in Grafana dashboards

### Module 4: Job Lifecycle Management

- **Multiple job types**: Training (single-node single-GPU / single-node multi-GPU / multi-node multi-GPU), online inference, batch inference, Notebooks
- **Distributed training**: DP / MP / PP / SP strategy declaration; platform allocates topology-matched nodes; NCCL/HCCL auto-injected
- **Checkpoint management**: Async streaming writes, incremental deduplication via parameter hashing, cross-storage-domain replication, one-click restore from UI
- **Image security**: Harbor private registry, Trivy CVE scanning, Cosign signature verification, automatic admission block on critical vulnerabilities

### Module 5: Multi-tenant & Access Control

- **Four-level tenant hierarchy**: Organization → Tenant → Project → User, with full resource isolation between tenants
- **Quota engine**: Reserves quota at submission, reconciles actual usage after completion; 80% reminder / 90% alert / 100% blocks new job submissions
- **Quota borrowing**: Tenants can temporarily borrow idle quota from others; auto-reclaimed on expiry without interrupting running jobs
- **RBAC**: Casbin three-dimensional permissions (resource × action × scope), SSO integration (LDAP / OAuth 2.0 / SAML), TOTP MFA
- **Metering & billing**: Minute-level precision, independent pricing per GPU model, monthly invoice PDF export

### Module 6: Open API & Platform Operations

- RESTful API (OpenAPI 3.0), gRPC, Webhook event push, Kubernetes CRD extensions
- Python SDK / Go SDK / Terraform Provider
- Per-tenant Feature Flag rollout, hot config updates, blue-green deployments via Argo Rollouts

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend language | Go (service layer), C++ 17 (HAL layer) |
| Web framework | Gin / Echo |
| Relational database | PostgreSQL |
| Time-series database | VictoriaMetrics |
| Columnar database | ClickHouse (audit logs) |
| Cache | Redis |
| Message queue | Kafka |
| Service registry | etcd |
| Container orchestration | Kubernetes + Volcano |
| Job orchestration | Kubeflow Training Operator, Argo Workflows |
| Service mesh | Istio |
| Observability | VictoriaMetrics + Loki + Jaeger + Grafana |
| Frontend | React 18 + TypeScript + Ant Design Pro |
| Image registry | Harbor |
| Secrets management | HashiCorp Vault |

Full tech stack with rationale: [docs/tech-stack.md](./docs/tech-stack.md)

---

## Quick Start

### Prerequisites

- Kubernetes 1.24+
- Helm 3.10+
- kubectl configured with cluster access
- At least one worker node with GPU/NPU

### Deploy

```bash
# 1. Add Helm repo
helm repo add ai-infra https://charts.ai-infra.io
helm repo update

# 2. Create namespace
kubectl create namespace ai-infra

# 3. Install (minimal config)
helm install ai-infra ai-infra/ai-infra \
  --namespace ai-infra \
  --set global.storageClass=standard \
  --set postgresql.enabled=true \
  --set redis.enabled=true

# 4. Wait for rollout
kubectl rollout status deployment/ai-infra-api -n ai-infra

# 5. Get console address
kubectl get svc ai-infra-console -n ai-infra
```

For production deployment see [docs/deployment/production.md](./docs/deployment/production.md).

---

## Repository Layout

```
ai-infra/
├── agent/                  # Node Agent (Go) — heartbeat, device collection
├── api/                    # API service (Go + Gin)
│   ├── handlers/           # HTTP handlers
│   ├── middleware/         # Auth, rate-limit, logging middleware
│   └── openapi/            # OpenAPI 3.0 spec
├── scheduler/              # Scheduler extension (Volcano Plugin)
├── hal/                    # Hardware Abstraction Layer (C++ 17 + CGO bridge)
│   ├── cuda/
│   ├── cann/
│   ├── cnrt/
│   └── topsrt/
├── quota/                  # Quota engine (Go)
├── billing/                # Billing service (Go)
├── diagnostic/             # Fault diagnosis service (Go)
├── checkpoint/             # Checkpoint management service (Go)
├── console/                # Web console (React 18 + TypeScript)
├── charts/                 # Helm charts
├── docs/                   # Documentation
│   ├── tech-stack.md       # Tech stack with rationale
│   ├── architecture.md     # Architecture design
│   ├── api-reference.md    # API reference
│   └── deployment/         # Deployment guides
├── deploy/                 # Kubernetes manifests / Kustomize
└── scripts/                # Ops scripts
```

---

## Documentation

| Doc | Description |
|---|---|
| [Architecture](./docs/architecture.md) | Overall architecture, module boundaries, data flow |
| [Tech Stack](./docs/tech-stack.md) | Per-module technology choices with rationale |
| [API Reference](./docs/api-reference.md) | Full REST API documentation |
| [Production Deployment](./docs/deployment/production.md) | Production deployment guide |
| [Development Guide](./docs/development.md) | Local development setup |
| [Contributing](./CONTRIBUTING.md) | How to contribute |

---

## Scope Boundaries

AI-Infra **only handles**: compute resource allocation, release, and monitoring.

AI-Infra **does not handle**:

| Upstream system | That system's responsibility |
|---|---|
| LLM serving platform | Model version management, API gateway, traffic routing, rate limiting |
| Agent development platform | Agent orchestration, tool calls, memory management |
| MLOps / data platform | Data cleaning, feature engineering, experiment tracking (MLflow, etc.) |

---

## Roadmap

- [ ] Core scheduling engine (Module 1)
- [ ] HAL — NVIDIA CUDA implementation (Module 2)
- [ ] Basic metrics collection and alerting (Module 3)
- [ ] Training job lifecycle management (Module 4)
- [ ] Multi-tenant quota engine (Module 5)
- [ ] HAL — Huawei Ascend CANN adaptation
- [ ] Cross-cluster federated scheduling
- [ ] Terraform Provider
- [ ] HAL — Cambricon / Enflame adaptation

---

## Contributing

Issues and pull requests are welcome. 

---

## License

[Apache 2.0](./LICENSE)
