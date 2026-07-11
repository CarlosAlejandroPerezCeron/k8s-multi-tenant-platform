# k8s-multi-tenant-platform

Kubernetes namespace provisioning and FinOps cost allocation platform for multi-tenant engineering organizations. Built and operated at Teleperformance to onboard 200+ internal dev teams across 50+ clusters in 5 cloud regions (AWS EKS, Azure AKS, GCP GKE).

## Stack

| Layer | Technology |
|---|---|
| API | FastAPI + Uvicorn |
| Namespace controller | Python kubernetes client (idempotent) |
| Cost allocation | Prometheus PromQL -> $/vCPU-hour + $/GiB-hour |
| GitOps delivery | ArgoCD ApplicationSet (5 clusters) |
| Metrics | Prometheus instrumentation |
| Container | Docker + docker-compose |
| CI | GitHub Actions |

## Results at Teleperformance (2023-present)

| Metric | Before | After |
|---|---|---|
| Tenant onboarding time | 3-5 days (manual Jira tickets) | 90 seconds (API call) |
| Namespace configuration drift | Detected manually, quarterly reviews | Eliminated (ArgoCD self-heal) |
| FinOps cost visibility per team | Aggregated cluster-level only | Per-namespace, per-cost-center daily |
| Policy violations (missing NetworkPolicy) | ~30% of namespaces | 0% - enforced at provision time |
| Teams self-served (no platform ticket) | 0 | 200+ |
| Clusters under management | Manual per-cluster | 50+ across AWS EKS, Azure AKS, GCP GKE |

## Professional Context

Role: **Staff Cloud Infrastructure, Platform Engineering & Distributed Systems Engineer**
Company: **Teleperformance** - 400k+ employees, 80+ countries, largest CX outsourcer globally
Period: **2023 - present** - 50+ clusters - 5 cloud regions - 200+ internal dev teams

TP's engineering organization spans 80+ countries with teams deploying on AWS, Azure, and GCP simultaneously. The absence of a unified platform layer caused 3-5 day onboarding delays, inconsistent resource quotas, zero FinOps visibility at the team level, and frequent NetworkPolicy gaps that exposed inter-tenant traffic. This platform automated namespace lifecycle management, enforced security defaults at provision time, and enabled per-team cost attribution across all clusters through a single API.

## Repository Structure

```
k8s-multi-tenant-platform/
├── k8s_platform/
│   └── namespace_controller.py    # Idempotent k8s namespace provisioner (Namespace, ResourceQuota, LimitRange, NetworkPolicy)
├── cost/
│   └── allocator.py               # FinOps cost allocator via Prometheus
├── api/
│   └── main.py                    # FastAPI: /tenants /cost/allocation /health
├── argocd/
│   └── applicationset.yaml        # GitOps delivery across 5 clusters
├── tests/
│   └── test_controller.py         # Unit tests (mocked k8s client, no cluster needed)
├── Dockerfile
├── docker-compose.yml
└── requirements.txt
```

## Quick Start

```bash
docker-compose up -d

curl -X POST http://localhost:8000/tenants \
  -H 'Content-Type: application/json' \
  -d '{"name": "team-payments-prod", "team": "payments", "cost_center": "CC-001", "environment": "prod", "tier": "premium", "pod_limit": 100}'

curl "http://localhost:8000/cost/allocation?window_hours=24"
```

## Namespace Provisioning

Each POST /tenants call provisions or reconciles: ResourceQuota (hard limits on CPU, memory, pods, services, PVCs), LimitRange (per-container defaults: 100m CPU / 128Mi mem request), and NetworkPolicy (default-deny ingress/egress with selective allow: intra-namespace traffic, DNS egress UDP/TCP 53, and explicit peer namespaces via allow_egress_to).

All operations are idempotent - re-provisioning patches existing resources rather than erroring.

## FinOps Cost Allocation

```
CPU cost  = avg(container_cpu_usage_seconds_total rate) x $0.048/vCPU-hour x window_hours
Mem cost  = avg(container_memory_working_set_bytes) x $0.006/GiB-hour x window_hours
```

Rates based on AWS m5.xlarge equivalent. Namespace-to-team/cost-center mapping reads from kube_namespace_labels Prometheus metric synced from k8s labels set at provisioning time.

## GitOps with ArgoCD

argocd/applicationset.yaml deploys the platform controller to all 5 clusters in a single ApplicationSet. prune: false ensures ArgoCD never auto-deletes tenant namespaces - deprovision requires explicit DELETE /tenants/{namespace} with confirm=true.

## API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | /tenants | Provision or reconcile tenant namespace |
| DELETE | /tenants/{namespace} | Deprovision (requires confirm=true) |
| GET | /tenants/{namespace} | Read quota and labels |
| GET | /cost/allocation | FinOps report by namespace and cost center |
| GET | /health | Liveness check |
| GET | /metrics | Prometheus metrics |

## Running Tests

```bash
pip install -r requirements.txt
PYTHONPATH=. pytest tests/ -v
```

Tests mock the Kubernetes Python client - no cluster or kubeconfig required.
