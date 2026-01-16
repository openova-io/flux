# SPEC: Flux Repository Structure

## Overview

GitOps configuration structure for Flux-managed cluster.

## Directory Structure

```
flux/
├── clusters/
│   └── contabo-europe/
│       ├── flux-system/       # Flux controllers
│       ├── network/           # istio, cilium, stunner
│       ├── security/          # kyverno, external-secrets, cert-manager
│       ├── database/          # cnpg, mongodb, dragonfly
│       ├── middleware/        # redpanda
│       ├── storage/           # minio, velero
│       ├── observability/     # grafana (LGTM stack)
│       ├── autoscaling/       # keda
│       ├── workplace/         # stalwart
│       └── tenants/           # product deployments
└── docs/
```

## Component File Pattern

Each component uses self-contained GitRepository + Kustomization:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: cnpg
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/openova-io/cnpg
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cnpg
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: cnpg
  path: ./deploy/prod
  prune: true
  wait: true
```

## Categories

| Category | Components | Purpose |
|----------|------------|---------|
| network | istio, cilium, stunner | Mesh, CNI, TURN |
| security | kyverno, external-secrets, cert-manager | Policy, secrets, TLS |
| database | cnpg, mongodb, dragonfly | Database operators |
| middleware | redpanda | Event streaming |
| storage | minio, velero | Object storage, backup |
| observability | grafana | LGTM stack |
| autoscaling | keda | Event-driven scaling |
| workplace | stalwart | Email server |
| tenants | `<tenant>` | Product deployments |

## Sync Intervals

| Resource | Interval |
|----------|----------|
| GitRepository | 1 minute |
| Kustomization | 10 minutes |
| HelmRelease | 10 minutes |

## Key Commands

```bash
# Status overview
flux get all

# Force reconciliation
flux reconcile kustomization tenants

# View logs
flux logs --all-namespaces

# Suspend (manual gate)
flux suspend kustomization <tenant>-prod
```

## Related

- [ADR-FLUX-GITOPS](./ADR-FLUX-GITOPS.md)
- [ADR-GITOPS-RELEASE-MANAGEMENT](./ADR-GITOPS-RELEASE-MANAGEMENT.md)
