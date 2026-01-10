# openova/flux

GitOps configuration for openova kubernetes clusters.

## Structure

```
flux/
├── clusters/
│   └── contabo-europe/
│       ├── flux-system/     # Flux controllers
│       ├── network/         # istio, cilium, stunner
│       ├── security/        # kyverno, external-secrets, cert-manager
│       ├── database/        # cnpg, mongodb, dragonfly
│       ├── middleware/      # redpanda
│       ├── storage/         # minio, velero
│       ├── observability/   # grafana (LGTM stack)
│       ├── autoscaling/     # keda
│       ├── workplace/       # stalwart
│       └── tenants/         # product deployments (talentmesh)
└── docs/
    ├── ADR-016-FLUX-GITOPS.md
    └── ADR-041-GITOPS-RELEASE-MANAGEMENT.md
```

## Component File Pattern

Each component is self-contained with both GitRepository source and Kustomization:

```yaml
# clusters/contabo-europe/database/cnpg.yaml
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
| network | istio, cilium, stunner | Service mesh, CNI, TURN |
| security | kyverno, external-secrets, cert-manager | Policy, secrets, TLS |
| database | cnpg, mongodb, dragonfly | Database operators |
| middleware | redpanda | Event streaming |
| storage | minio, velero | Object storage, backup |
| observability | grafana | LGTM stack |
| autoscaling | keda | Event-driven scaling |
| workplace | stalwart | Email server |
| tenants | talentmesh | Product deployments |

## Bootstrap

```bash
# Initial cluster bootstrap
flux bootstrap github \
  --owner=openova-io \
  --repository=flux \
  --branch=main \
  --path=clusters/contabo-europe \
  --personal
```

## Key Commands

```bash
# Status overview
flux get all

# Force reconciliation
flux reconcile kustomization tenants

# View logs
flux logs --all-namespaces

# Suspend production (manual gate)
flux suspend kustomization talentmesh-prod
```

## Sync Intervals

| Resource | Interval |
|----------|----------|
| GitRepository | 1 minute |
| Kustomization | 10 minutes |
| HelmRelease | 10 minutes |

## Related Documentation

- [ADR-016: Flux GitOps](./docs/ADR-016-FLUX-GITOPS.md)
- [ADR-041: GitOps Release Management](./docs/ADR-041-GITOPS-RELEASE-MANAGEMENT.md)

---

*Part of [openova](https://github.com/openova-io)*
