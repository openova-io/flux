# ADR: Flux GitOps

**Status:** Accepted
**Date:** 2024-05-01

## Context

Need a GitOps delivery mechanism for Kubernetes. Options: Flux, ArgoCD, or manual kubectl.

## Decision

Use Flux as the GitOps delivery engine.

## Rationale

| Factor | Flux | ArgoCD |
|--------|------|--------|
| Memory overhead | ~200MB | ~500-800MB |
| Architecture | Kubernetes-native CRDs | Separate UI/API |
| SOPS integration | Native | Plugin |
| CLI workflow | Excellent | UI-focused |

**Key Decision Factors:**
- Lower resource overhead
- Native SOPS integration for secrets
- CLI-focused fits single-developer workflow
- Kubernetes-native CRDs

## Components

| Controller | Memory | Purpose |
|------------|--------|---------|
| source-controller | 64MB | Git/Helm repo sync |
| kustomize-controller | 64MB | Kustomization apply |
| helm-controller | 64MB | HelmRelease management |
| notification-controller | 32MB | Alerts |

## Consequences

**Positive:** Low overhead, native secrets, CLI-focused, K8s-native
**Negative:** No GUI (use Grafana dashboards), learning curve

## Related

- [ADR-GITOPS-RELEASE-MANAGEMENT](./ADR-GITOPS-RELEASE-MANAGEMENT.md)
- [SPEC-FLUX-STRUCTURE](./SPEC-FLUX-STRUCTURE.md)
