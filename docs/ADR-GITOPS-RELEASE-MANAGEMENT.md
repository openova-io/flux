# ADR: GitOps Release Management

**Status:** Accepted
**Date:** 2024-10-01

## Context

Need a release strategy that supports safe deployments with rollback capability.

## Decision

Implement GitOps-based release management:
- Git as source of truth
- Flux for continuous reconciliation
- Branch-based environment promotion
- Flagger for canary analysis

## Release Flow

```
Feature Branch → PR → main → Flux Sync → Staging → Promote → Production
                                            ↓
                                      Canary Analysis
                                            ↓
                                      Auto-Rollback (on failure)
```

## Environment Strategy

| Environment | Branch/Path | Sync Interval |
|-------------|-------------|---------------|
| Staging | `clusters/contabo-europe/tenants/<tenant>-staging` | 1m |
| Production | `clusters/contabo-europe/tenants/<tenant>-prod` | 10m |

## Promotion Process

1. Changes merge to main
2. Flux syncs to staging (automatic)
3. Validation in staging
4. Manual promotion to production (via PR or gate)
5. Canary analysis (Flagger)
6. Automatic rollback if metrics degrade

## Rollback Strategy

| Trigger | Action |
|---------|--------|
| Flagger metric failure | Auto-rollback |
| Manual intervention | Git revert + sync |
| Emergency | `kubectl rollout undo` |

## Consequences

**Positive:** Auditable history, safe rollbacks, automated delivery
**Negative:** Git as bottleneck, learning curve

## Related

- [ADR-FLUX-GITOPS](./ADR-FLUX-GITOPS.md)
- [ADR-PROGRESSIVE-DELIVERY](../../handbook/docs/adrs/ADR-PROGRESSIVE-DELIVERY.md)
