# matflixlab — GitOps Hardening Plan

## Purpose of this document

This is a task brief for an agent (Kiro, running on the matflix homelab host) to
**verify against the live cluster and prepare an execution plan** — not a script to
run blindly. Read a step fully, check it against what's actually running, note any
drift from what this document assumes, then propose the concrete commands/manifests
you intend to apply. Do not touch secrets or enable pruning without explicit human
sign-off between steps — everything else can proceed once verified.

## Goal

Right now, several parts of this cluster still require SSH-ing into the host and
running `kubectl` by hand: applying Secrets, applying Traefik's `HelmChartConfig`,
adopting Umami into ArgoCD, restarting the `landing` pod after a content change, and
deleting anything ArgoCD manages (since `prune: false`). The goal of this plan is to
close all of those gaps so that, going forward, **every change to the cluster's
running content happens through a git commit**, reconciled automatically by ArgoCD.

The one honest exception: cluster genesis (installing k3s, installing ArgoCD itself,
installing the sealed-secrets controller) is a one-time bootstrap that can't be
GitOps'd by definition — something has to exist before it can reconcile from git.
Treat those, and only those, as legitimate manual steps.

## Ground rules

- This is a single-node cluster with real running services (speakstats, jellyfin,
  umami, the monitoring stack) and **no backups implemented yet**. Treat every step
  as production. Favor caution over speed.
- Work through the steps **in order** — later steps assume earlier ones are done.
  Don't skip ahead.
- Before Step 2 (secrets) and Step 4 (prune), take a backup outside the git repo:
  - `kubectl get secrets -A -o yaml > /root/backups/secrets-$(date +%F).yaml`
  - a tarball of `/var/lib/rancher/k3s/server/db` (k3s's embedded datastore)
- Commit each step separately with a clear message. Don't bundle steps.
- The end state of every step must be "committed to git, reconciled by ArgoCD."
  Manual `kubectl apply` is acceptable only for genuine one-time bootstrap
  installs (sealed-secrets controller, Reloader) or read-only verification
  (`kubectl get`, `kubectl diff`) — never as the final way a change lands.
- If live state doesn't match what this document describes, or something feels
  destructive/irreversible, **stop and report back** rather than guessing.
- After each step, append a dated entry to the **Execution Log** section at the
  bottom of this file: what was verified, what was done, anything that deviated
  from the plan and why.

## Current state (verified 2026-09-05, may have drifted — re-check)

- ArgoCD Application: [k8s/apps/argocd/application.yaml](k8s/apps/argocd/application.yaml)
  — watches `k8s/` on `master`, `syncPolicy.automated: { prune: false, selfHeal: true }`.
- Root kustomization: [k8s/kustomization.yaml](k8s/kustomization.yaml) — lists
  `infrastructure/namespaces`, `infrastructure/registry`, and
  `apps/{landing,speakstats,jellyfin,umami-proxy,monitoring}`. **Missing**:
  `apps/umami` and `infrastructure/traefik` — these exist on disk but aren't
  reconciled by ArgoCD; they were applied by hand at some point.
- Namespaces are declared in three separate places:
  [k8s/infrastructure/namespaces/namespaces.yaml](k8s/infrastructure/namespaces/namespaces.yaml)
  (`matflixlab`, `registry`), [k8s/apps/monitoring/namespace.yaml](k8s/apps/monitoring/namespace.yaml)
  (`monitoring`), and `argocd`'s own install manifest (implicit, not in this repo).
- Secrets: `.gitignore` excludes `k8s/**/secret.yaml` and `k8s/**/secrets.yaml`
  entirely — these are applied manually and exist only on the live cluster/host.
  Known ones to account for: Grafana admin password, Umami/Postgres credentials,
  umami-proxy credentials, and the ArgoCD repo SSH deploy key (there's a *committed*
  placeholder at [k8s/apps/argocd/repo-secret.yaml](k8s/apps/argocd/repo-secret.yaml)
  with a `REPLACE_WITH_PRIVATE_KEY` marker — the real key is applied out-of-band;
  **do not overwrite this file with a real key**, it must stay a placeholder in git).
- `landing`'s content is a ConfigMap ([k8s/apps/landing/](k8s/apps/landing/)) mounted
  into an nginx Deployment; per [deploy.md](deploy.md), a content change currently
  requires a manual `kubectl rollout restart deployment/landing`.

## Step 1 — Consolidate namespace management

**Why first:** foundational and low-risk; later steps (adopting umami/traefik,
ApplicationSet) shouldn't inherit today's scattered namespace ownership.

- Move the `monitoring` Namespace definition out of
  `k8s/apps/monitoring/namespace.yaml` and into
  `k8s/infrastructure/namespaces/namespaces.yaml`, alongside `matflixlab` and
  `registry`.
- Delete `k8s/apps/monitoring/namespace.yaml` and remove its reference from
  `k8s/apps/monitoring/kustomization.yaml`.
- Leave the `argocd` namespace out of this file — it's created as part of ArgoCD's
  own one-time bootstrap install, and making ArgoCD's namespace depend on ArgoCD
  already running to reconcile it is circular. Note this explicitly rather than
  "fixing" it.

**Acceptance:** `kustomize build k8s` shows exactly one `Namespace` object each for
`matflixlab`, `registry`, `monitoring`, all sourced from the consolidated file;
`kubectl get ns` matches; ArgoCD stays `Synced`/`Healthy` after the commit.

## Step 2 — Bring Secrets under git via Sealed Secrets

**Why second:** everything after this point (adopting more apps into ArgoCD,
enabling prune) should not put unmanaged plaintext secrets at risk.

- One-time bootstrap (documented exception): install the
  [bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)
  controller and the `kubeseal` CLI. Check the current release tag rather than
  hardcoding an old version.
- **Immediately** back up the controller's auto-generated private key
  (`kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml`)
  to somewhere outside git, e.g. `/root/backups/`. Losing this key makes every
  committed SealedSecret permanently undecryptable — this is the single most
  important artifact in this whole plan to not lose.
- Enumerate every live plaintext Secret (`kubectl get secrets -A`) and cross-check
  against the known list above.
- For each one, `kubeseal` it into a `SealedSecret` manifest committed next to the
  app it belongs to (e.g. `k8s/apps/monitoring/grafana/sealed-secret.yaml`), and
  update that app's `kustomization.yaml` to reference the new file instead of the
  old gitignored `secret.yaml`.
- Keep the existing `.gitignore` rules for `secret.yaml`/`secrets.yaml` in place as
  a permanent safety net against ever committing plaintext by accident — the new
  files use a different name (`sealed-secret.yaml`) so there's no conflict.
- **Exception:** leave `k8s/apps/argocd/repo-secret.yaml` (the deploy key) as-is for
  now unless you also plan to seal it in this pass — either is fine, but call out
  the decision explicitly in the execution log.

**Acceptance:** no `kustomization.yaml` references a plaintext `secret.yaml`
filename anymore; deleting a live Secret for a low-stakes app (start with
umami-proxy, not the ArgoCD deploy key) and letting ArgoCD resync recreates it
correctly from the SealedSecret.

## Step 3 — App-of-Apps via ApplicationSet

**Why third:** this is what actually brings `apps/umami` and
`infrastructure/traefik` under GitOps for the first time, and it should happen only
after secrets are safe (Step 2), so adopting them doesn't interact badly with
unmanaged secret state.

- Replace the single root `application.yaml` with an ApplicationSet using a git
  directory generator over `k8s/apps/*`, so each subdirectory becomes its own
  ArgoCD Application automatically — adding a new app in the future means creating
  a directory and committing, nothing else.
- Handle `k8s/infrastructure/*` (registry, traefik, namespaces) the same way, either
  as a second ApplicationSet or one static Application — infra changes less often
  and may warrant more caution than app-level changes, so keep it separate from the
  apps ApplicationSet if that's easier to reason about.
- This is the step that adopts **already-running** resources (Umami, Traefik's
  `HelmChartConfig`) into ArgoCD for the first time. Before letting `selfHeal` touch
  them, run a diff (`kubectl diff` or ArgoCD's own dry-run) to confirm what's live
  matches what's in git — these were hand-applied, so there may be drift.
- Keep `prune: false` through this step. Pruning comes next, once you've confirmed
  the newly-adopted apps reconcile with no surprise diffs.

**Acceptance:** `kubectl get applications -n argocd` shows one Application per
directory under `k8s/apps/` (landing, speakstats, jellyfin, umami, umami-proxy,
monitoring) plus infra, all `Synced`/`Healthy`. Proof test: add a trivial ConfigMap
under a new `k8s/apps/test-app/` directory, commit, confirm a new Application
appears with zero manual `kubectl apply -f application.yaml`. Remove the test
directory afterward.

## Step 4 — Enable pruning

**Why fourth:** this is what makes *deletions* flow through git too — without it,
removing something from a manifest does nothing live. It's also the most
irreversible-feeling change here, so it comes after everything above is confirmed
stable.

- Pilot on the lowest-stakes app first: `landing`. Set `prune: true` for it (however
  the ApplicationSet from Step 3 allows a per-app override — ask/stop if it's
  unclear how to scope this to one app rather than flipping it globally).
- Prove it works: add a throwaway ConfigMap to `landing`'s manifests, commit, confirm
  ArgoCD creates it. Then remove it from git, commit, and confirm ArgoCD **deletes**
  it live with zero manual `kubectl`. This is the actual proof of "any change
  through git."
- Once proven, roll `prune: true` out to the remaining apps one at a time
  (speakstats, jellyfin, umami, umami-proxy, monitoring), checking sync status after
  each before moving to the next.
- Leave ArgoCD's own self-management Application out of this rollout — flag it for
  separate, explicit human sign-off rather than auto-enabling, since a mistake there
  risks ArgoCD deleting its own resources.

**Acceptance:** the create-then-delete test above works end-to-end with no manual
`kubectl`; every Application has `prune: true` except ArgoCD's own, which is called
out by name in the execution log as a deliberate hold.

## Step 5 — stakater/Reloader for auto-restart on config change

**Why last:** independent of the others, and it's the capstone that removes the one
manual step already written down in [deploy.md](deploy.md) (`kubectl rollout
restart deployment/landing` after a content change).

- One-time bootstrap install of [stakater/Reloader](https://github.com/stakater/Reloader)
  (same category as ArgoCD/sealed-secrets — install once, pin the version).
- Add the `reloader.stakater.com/auto: "true"` annotation to the `landing`
  Deployment ([k8s/apps/landing/deployment.yaml](k8s/apps/landing/deployment.yaml)).
- Check the other apps' manifests and `deploy.md` for the same "requires manual
  restart" pattern (e.g. Grafana's ConfigMap-driven config) and annotate those
  Deployments too.
- Update `deploy.md` to remove the now-obsolete manual restart instructions,
  replacing them with a note that config changes auto-apply via Reloader.

**Acceptance:** edit `k8s/apps/landing/html/index.html` via its ConfigMap, commit,
and confirm the landing pod restarts on its own (`kubectl get pods -n matflixlab -l
app=landing` shows a new pod age) with no manual `kubectl rollout restart`.

## Out of scope for this plan

- speakstats / dev-tech-news application-level CI/CD (build pipelines, image
  tagging) — tracked separately in [speakstats-cicd-plan.md](speakstats-cicd-plan.md).
- Any GitHub Actions runner, hosted or self-hosted.
- Multi-node HA, PVC/off-site backups (Velero/restic), Alertmanager — separate
  future initiatives, not part of this hardening pass.

## Execution log

<!-- Agent: append a dated entry here after each step — what was verified against
     live state, what was done, anything that deviated from this plan and why. -->
