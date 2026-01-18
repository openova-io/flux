# ADR: Flux GitOps

**Status:** Accepted
**Date:** 2024-05-01
**Updated:** 2026-01-17

## Context

Need a GitOps delivery mechanism for Kubernetes. Options: Flux, ArgoCD, or manual kubectl.

## Decision

Use **Flux** as the GitOps delivery engine with **Gitea** as the internal Git provider and External Secrets Operator for secrets management.

## Architecture

```mermaid
flowchart TB
    subgraph Gitea["Gitea (Internal Git)"]
        Components[Component Repos]
        Tenant[Tenant Repos]
    end

    subgraph K8s["Kubernetes Cluster"]
        subgraph Flux["Flux Controllers"]
            Source[source-controller]
            Kustomize[kustomize-controller]
            Helm[helm-controller]
            Notify[notification-controller]
        end

        subgraph ESO["Secrets"]
            ESOp[External Secrets Operator]
            PS[PushSecrets]
        end

        Resources[K8s Resources]
    end

    subgraph External["External"]
        Vault[Vault]
    end

    Components --> Source
    Tenant --> Source
    Source --> Kustomize
    Source --> Helm
    Kustomize --> Resources
    Helm --> Resources
    Notify --> Gitea
    ESOp --> Vault
    ESOp --> Resources
```

## Rationale

| Factor | Flux | ArgoCD |
|--------|------|--------|
| Memory overhead | ~200MB | ~500-800MB |
| Architecture | Kubernetes-native CRDs | Separate UI/API |
| Secrets | Via ESO (PushSecrets) | Via ESO (PushSecrets) |
| CLI workflow | Excellent | UI-focused |

**Key Decision Factors:**
- Lower resource overhead
- CLI-focused fits single-developer workflow
- Kubernetes-native CRDs
- Works well with External Secrets Operator
- Integrates seamlessly with Gitea

## Components

| Controller | Memory | Purpose |
|------------|--------|---------|
| source-controller | 64MB | Git/Helm repo sync |
| kustomize-controller | 64MB | Kustomization apply |
| helm-controller | 64MB | HelmRelease management |
| notification-controller | 32MB | Alerts |

## Gitea Integration

Flux connects to Gitea repositories using Gitea access tokens:

```mermaid
flowchart LR
    subgraph Gitea["Gitea"]
        Repos[Repositories]
        Actions[Gitea Actions]
    end

    subgraph K8s["Kubernetes"]
        Flux[Flux]
        Secret[gitea-token Secret]
    end

    Secret --> Flux
    Flux -->|"Clone/Pull"| Repos
    Actions -->|"Trigger"| Flux
```

### Gitea Token Secret

Created during bootstrap and stored via ESO:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitea-token
  namespace: flux-system
type: Opaque
data:
  username: Zm... # base64 encoded
  password: Z2l... # base64 encoded (Gitea access token)
```

### GitRepository for Gitea

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: <component>
  namespace: flux-system
spec:
  interval: 5m
  url: https://gitea.<domain>/<org>/<component>.git
  ref:
    branch: main
  secretRef:
    name: gitea-token
```

## Secrets Management

**Important:** Flux uses External Secrets Operator (ESO) with PushSecrets pattern:

```mermaid
flowchart LR
    subgraph Bootstrap["Bootstrap"]
        Vault[Vault]
    end

    subgraph K8s["Kubernetes"]
        ESO[ESO]
        PS[PushSecrets]
        Secret[K8s Secret]
    end

    Vault -->|"Fetch"| ESO
    ESO --> Secret
    PS -->|"Push to all clusters"| Vault
```

- **No SOPS**: SOPS has been eliminated from the architecture
- **PushSecrets**: 100% PushSecrets pattern for multi-region
- **K8s Secrets as source of truth**: Apps read from K8s Secrets only

See [ADR-SECRETS-MANAGEMENT](../../external-secrets/docs/ADR-SECRETS-MANAGEMENT.md) for details.

## Configuration

### Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <component>
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: <namespace>
  sourceRef:
    kind: GitRepository
    name: <component>
  path: ./manifests
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: <component>
      namespace: <namespace>
```

## Multi-Region GitOps

```mermaid
flowchart TB
    subgraph Gitea["Gitea (Bidirectional Mirror)"]
        G1[Gitea Region 1]
        G2[Gitea Region 2]
        G1 <-->|"Mirror"| G2
    end

    subgraph Region1["Region 1"]
        Flux1[Flux]
        K8s1[K8s Resources]
    end

    subgraph Region2["Region 2"]
        Flux2[Flux]
        K8s2[K8s Resources]
    end

    G1 --> Flux1
    G2 --> Flux2
    Flux1 --> K8s1
    Flux2 --> K8s2
```

- Each region has its own Flux installation
- Both Gitea instances mirror repositories bidirectionally
- Flux in each region pulls from local Gitea
- Region-specific configuration handled via Kustomize overlays

## Gitea Actions Integration

Gitea Actions can trigger Flux reconciliation:

```yaml
# .gitea/workflows/notify-flux.yaml
name: Notify Flux
on:
  push:
    branches: [main]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Flux reconciliation
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.FLUX_WEBHOOK_TOKEN }}" \
            https://flux-webhook.<domain>/hook/...
```

## Consequences

**Positive:**
- Low overhead (~200MB)
- K8s-native CRDs
- CLI-focused
- Works with ESO PushSecrets
- Seamless Gitea integration
- Self-hosted Git (no external dependency)

**Negative:**
- No GUI (use Grafana dashboards)
- Learning curve for Flux resources

## Related

- [ADR-GITEA](../../gitea/docs/ADR-GITEA.md)
- [ADR-SECRETS-MANAGEMENT](../../external-secrets/docs/ADR-SECRETS-MANAGEMENT.md)
- [ADR-GITOPS-RELEASE-MANAGEMENT](./ADR-GITOPS-RELEASE-MANAGEMENT.md)
- [SPEC-FLUX-STRUCTURE](./SPEC-FLUX-STRUCTURE.md)
