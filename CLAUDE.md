# matflixlab

Single-node k3s homelab, GitOps via ArgoCD. Day-to-day operational runbook:

@deploy.md

## Landmines specific to this repo

- **[k8s/kustomization.yaml](k8s/kustomization.yaml) is not a live reconciliation
  path.** Since the Step 3 ApplicationSet migration (see
  [gitops-hardening-plan.md](gitops-hardening-plan.md)), ArgoCD reconciles each
  `k8s/apps/*` and `k8s/infrastructure/*` directory individually via
  `k8s/apps/argocd/appset-*.yaml`. The root file is kept only for local
  `kustomize build k8s` validation and must be updated by hand when apps are
  added/removed — don't assume editing it affects the live cluster.
- **[k8s/apps/argocd/repo-secret.yaml](k8s/apps/argocd/repo-secret.yaml) must stay
  a placeholder** (`REPLACE_WITH_PRIVATE_KEY`). The real deploy key is applied
  out-of-band on the host — never commit a real key here.
- **Adding a new app = create a directory under `k8s/apps/` and commit.** The
  `apps` ApplicationSet auto-discovers it (excludes `apps/argocd` itself). No
  manual `kubectl apply` needed except for genuine one-time bootstrap installs
  (k3s, ArgoCD, sealed-secrets, Reloader) — anything else should go through git.
- **Secrets are SealedSecrets, not plaintext.** `.gitignore` still blocks
  `secret.yaml`/`secrets.yaml` as a safety net; commit `sealed-secret.yaml`
  instead (`kubeseal`). See Step 2 of [gitops-hardening-plan.md](gitops-hardening-plan.md).
