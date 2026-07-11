# FinTrack GitOps

Kubernetes manifests for FinTrack - the single source of truth ArgoCD watches and reconciles the cluster against. Nothing here is applied manually in normal operation; CI updates image tags, ArgoCD detects changes in Git, and reconciles the live cluster to match.

## Repository Structure

```
apps/                          - ArgoCD Application definitions (one per component)
  frontend.yaml                 - watches manifests/frontend
  backend.yaml                  - watches manifests/backend
  cluster.yaml                  - watches manifests/cluster (Kyverno policies)

manifests/
  frontend/
    deployment.yaml              - fintrack-frontend Deployment
    service.yaml
    ingress.yaml                 - routes /* to frontend

  backend/
    deployment.yaml               - fintrack-backend Rollout (Argo Rollouts, canary)
    service.yaml
    hpa.yaml                      - HorizontalPodAutoscaler (2-5 replicas)
    secretstore.yaml               - ClusterSecretStore (AWS Secrets Manager via IRSA)
    externalsecret.yaml            - ExternalSecret (syncs backend credentials)
    alerts.yaml                    - PrometheusRule (fintrack-alerts)

  cluster/
    policy-disallow-root.yaml      - Kyverno ClusterPolicy: blocks root containers
    policy-restrict-registries.yaml - Kyverno ClusterPolicy: ECR-only images
```

## What ArgoCD Manages Here

| Application         | Path                 | Manages                                                                 |
| ------------------- | -------------------- | ----------------------------------------------------------------------- |
| `fintrack-frontend` | `manifests/frontend` | Frontend Deployment, Service, Ingress                                   |
| `fintrack-backend`  | `manifests/backend`  | Backend Rollout, Service, HPA, SecretStore, ExternalSecret, alert rules |
| `fintrack-cluster`  | `manifests/cluster`  | Cluster-scoped Kyverno policies                                         |

All three Applications use:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

`selfHeal: true` means any manual `kubectl` change to a resource ArgoCD manages will be reverted back to match Git automatically. Intentional manual changes should go through Git, not `kubectl apply`, or they'll be undone within minutes.

## Backend Rollout - Canary Strategy

`manifests/backend/deployment.yaml` is a `Rollout` (Argo Rollouts), not a `Deployment`:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - pause: { duration: 30s }
      - setWeight: 50
      - pause: { duration: 30s }
      - setWeight: 100
```

Every push to `fintrack-backend`'s `main` branch results in a gradual traffic shift to the new version, not an all-at-once replacement. Check live status:

```bash
kubectl-argo-rollouts get rollout fintrack-backend -n fintrack --watch
```

## Kyverno Policies

Both currently run in `Audit` mode - violations are logged as Kubernetes Events, not blocked:

```bash
kubectl get clusterpolicy
kubectl get events -n fintrack --field-selector reason=PolicyViolation
```

## Making Changes

```bash
# 1. Edit the manifest
vim manifests/backend/deployment.yaml

# 2. Commit and push
git add manifests/backend/deployment.yaml
git commit -m "describe the change"
git push

# 3. ArgoCD auto-syncs within ~3 minutes, or force it immediately:
kubectl annotate app fintrack-backend -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

Never edit a live resource with `kubectl edit` or `kubectl apply` directly - `selfHeal` will revert it. Always go through Git.

## Related Repositories

| Repo                                                                         | Description                                                                                                                |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| [fintrack-infrastructure](https://github.com/qezman/fintrack-infrastructure) | Terraform - provisions everything this repo's manifests run on top of                                                      |
| [fintrack-frontend](https://github.com/qezman/fintrack-frontend)             | React + Vite source, builds the image referenced in `manifests/frontend/deployment.yaml`                                   |
| [fintrack-backend](https://github.com/qezman/fintrack-backend)               | Fastify + Prisma source, builds the image referenced in `manifests/backend/deployment.yaml`, owns the `prisma-migrate` Job |

## Documentation

Full setup guide, architecture decisions, and redeployment walkthrough:

[FinTrack Platform Documentation](https://polarized-boater-990.notion.site/FinTrack-EKS-Platform-38d604d0a68980168e51cf384b92a454)

## Author

**Kazeem**
