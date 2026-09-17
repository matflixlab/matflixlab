# dev-tech-news → homelab migration plan

## Goal

Re-host `dev-tech-news` (currently dead — the Tencent VM is disabled) on
`matflix-server`, the k3s homelab host. Fresh setup, no data migration:
`processed_urls.json` and the 7 days of `site/data/*.json` on the old VM are
not worth recovering — the ledger rebuilds itself (first run just re-analyses
some already-seen articles once, costing a few cents) and the cards roll over
within a week anyway.

## Decision: Kubernetes-native, not a docker-compose lift-and-shift

The tempting move is to copy what already worked on Tencent — `docker-compose.yml`
plus the systemd timer — straight onto the laptop. It would be running in
twenty minutes. **Don't**, for one reason: this box is otherwise 100% k3s +
ArgoCD, and a compose-plus-systemd service would be a second deployment model
living outside GitOps. Every other app here is "create a directory under
`k8s/apps/`, commit, ArgoCD picks it up" (the ApplicationSet from
`gitops-hardening-plan.md` auto-discovers it). A snowflake breaks that, and
nothing about this workload justifies the exception.

**Worth noting explicitly:** the reason we *rejected* Kubernetes for this app
back in `tencent-dev-tech-news-plan.md` no longer holds. That objection was
that k8s CronJobs are triggered by the control-plane's controller-manager,
so a Tencent *agent* node would have depended on the home box being up —
defeating the point of off-site hosting. Here the control plane **is** the
box running the job. The objection evaporates; the CronJob is now the
natural fit.

It also buys something that matters a lot given the risk below: a hard
memory limit.

## The real risk: this box is already tight on memory

Not a footnote — the thing most likely to go wrong.

Grafana's last reading before the migration: **~65% RAM used and ~44% swap
in use**, with 8GB total, while running k3s, Traefik, Prometheus, Grafana,
Loki, Promtail, Tempo, Jellyfin, Umami + Postgres, speakstats (Next.js +
whisper.cpp), ArgoCD and the registry. Into that, this app wants to run
concurrent headless Chromium sessions — the exact workload that took a
*dedicated* 2 vCPU/4GB Tencent VM down hard twice on 2026-09-08, each time
needing a console reboot (CHANGELOG §0.3.1).

The difference now is blast radius: on Tencent, an OOM killed a box that
only ran this app. Here it could take Jellyfin, Grafana or ArgoCD with it.

Three mitigations, all cheap, and the first is non-negotiable:

1. **A hard `resources.limits.memory` on the CronJob pod.** This is what
   turns "the node OOMs and random pods die" into "the newsletter job dies
   and retries tomorrow". Start at 1.5–2Gi.
2. **Keep `--crawl-concurrency 2`** in the container args (see §0.3.1 for
   why the default of 8 is unsafe — the per-site and per-article limits
   multiply, so the real worst case is the square). Consider starting at 1
   on this box and raising it only if runs are too slow.
3. **`concurrencyPolicy: Forbid`** on the CronJob, so a slow run can never
   overlap with the next day's — we hit exactly that failure mode manually
   on Tencent (two containers racing on the same `processed_urls.json`).

If it still destabilises the box, the honest fallback is dropping the
website-crawling sources (`scraping_websites.csv`) and running RSS-only —
RSS costs a fetch, not a browser.

## Steps

### Step 1 — Get the repo and image onto the host

`dev-tech-news` is a private repo; the Tencent VM used a read-only deploy
key, and this host needs its own (don't reuse the VM's — generate fresh,
same convention as every other key here).

```bash
# on matflix-server
ssh-keygen -t ed25519 -C "matflix-server-dev-tech-news" -f ~/.ssh/id_ed25519_dev_tech_news -N ""
cat ~/.ssh/id_ed25519_dev_tech_news.pub     # → add as read-only deploy key on the repo
# ~/.ssh/config: Host github-dev-tech-news / HostName github.com / IdentityFile that key
git clone git@github-dev-tech-news:matflixlab/dev-tech-news.git ~/dev-tech-news
cd ~/dev-tech-news && git checkout feature/html-news-site
```

Then build and push to the local registry, same path speakstats uses
(`deploy.md` → "SpeakStats — deploy nowej wersji"):

```bash
docker build -t localhost:30500/dev-tech-news:latest .
docker push localhost:30500/dev-tech-news:latest
```

**Heads-up:** this image is ~4.7GB (Playwright + Chromium) and the build
pulls a few hundred MB. It's slow on this hardware and it lands in
`registry-data/` on the host disk — fine at 915GB free, but don't be
surprised by the build time.

**Acceptance:** `docker images | grep dev-tech-news` shows the image, and
`curl -s localhost:30500/v2/dev-tech-news/tags/list` lists `latest`.

### Step 2 — Secret

The OpenAI key must be **freshly generated** — the one the VM used is
revoked (it's been 401ing since 2026-09-10) and was compromised anyway
(exposed in VS Code local history back on 2026-09-04).

Per `gitops-hardening-plan.md` Step 2, secrets are SealedSecrets, not
plaintext — `.gitignore` still blocks `secret.yaml`/`secrets.yaml` as a
safety net:

```bash
kubectl create secret generic dev-tech-news-secret \
  --namespace matflixlab \
  --from-literal=OPENAI_API_KEY='sk-...' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > k8s/apps/dev-tech-news/sealed-secret.yaml
```

**Acceptance:** the sealed file commits cleanly, and after ArgoCD syncs,
`kubectl -n matflixlab get secret dev-tech-news-secret` exists.

### Step 3 — Manifests: `k8s/apps/dev-tech-news/`

Five files, mirroring the conventions already used by speakstats/cv-gate
(namespace `matflixlab`, `local-path` PVC, Traefik ingress on the `web`
entrypoint, `kustomization.yaml` listing resources):

| File | What |
|---|---|
| `pvc.yaml` | one `local-path` RWO PVC, ~2Gi — holds `site/` (7 PDFs' worth of JSON + `index.html`) **and** `processed_urls.json` |
| `cronjob.yaml` | daily 06:00, runs `generate_newsletter.py` |
| `deployment.yaml` | `nginx:alpine` serving the PVC read-only |
| `service.yaml` + `ingress.yaml` | `news.matflixlab.pl` → nginx:80 |
| `sealed-secret.yaml` | from Step 2 |

Key details that aren't boilerplate:

- **CronJob**: `schedule: "0 6 * * *"` **plus `timeZone: "Europe/Warsaw"`** —
  without that field the schedule is UTC, which silently drifts an hour
  twice a year. (Supported on k3s 1.28; the field went stable in k8s 1.27.)
- **`concurrencyPolicy: Forbid`**, `successfulJobsHistoryLimit: 3`,
  `restartPolicy: OnFailure`, `backoffLimit: 1` — one retry, not six, so a
  systematically broken run (like the 401 outage) doesn't burn six rounds of
  crawling each morning.
- **`resources.limits.memory`** as argued above. Also set requests, so the
  scheduler accounts for it rather than overcommitting the node.
- **Args**: keep the full flag set from the systemd unit, including
  `--crawl-concurrency 2` and `--site-dir /data/site`.
- **Both** the CronJob and the nginx Deployment mount the same PVC — the job
  writes, nginx serves read-only. RWO is fine: single node, so they're
  always co-located.
- **`/dev/shm`**: mount an `emptyDir` with `medium: Memory` (~256Mi) at
  `/dev/shm`. Containers default to 64MB, which Chromium is notorious for
  crashing on. It happened not to bite under docker-compose on Tencent;
  don't assume that carries over — this is the classic k8s+Chromium gotcha
  and it fails in confusing ways (crawls silently returning nothing).

The CSVs (`feeds.csv`, `scraping_websites.csv`, `conference_sources.csv`)
stay **baked into the image** rather than becoming a ConfigMap. A ConfigMap
would let you edit feeds without a rebuild, but it means the same CSVs exist
in two repos and can silently diverge — worse failure mode than the rebuild
friction. Revisit if feed edits get frequent enough to annoy.

**Acceptance:** commit the directory; the `apps` ApplicationSet
auto-discovers it (no manual `kubectl apply`), `kubectl -n argocd get
applications` shows `dev-tech-news` Synced/Healthy, and the nginx pod is
Running.

### Step 4 — Cloudflare: A record → Tunnel route

This reverses the earlier setup, and it's worth saying plainly: the Tunnel
approach you originally reached for **is now the correct one**. It was wrong
on 2026-09-08 only because the origin was a Tencent box with its own public
IP, so routing through the home tunnel added a pointless dependency. Now the
origin *is* the tunnel host.

1. Delete the `news` **A record** (`162.62.135.86` — dead IP).
2. Zero Trust → Networks → Tunnels → `matflixlab` → add a published
   application: hostname `news.matflixlab.pl`, service `http://127.0.0.1:80`
   — identical to `speakstats`, `jellyfin`, `grafana` etc.

**Acceptance:** `dig news.matflixlab.pl` resolves to Cloudflare IPs and
`curl -I https://news.matflixlab.pl/` returns 200 served by the homelab
nginx.

### Step 5 — Verify end to end

1. Trigger a run without waiting for 06:00:
   `kubectl -n matflixlab create job --from=cronjob/dev-tech-news dtn-manual-1`
2. Watch it: `kubectl -n matflixlab logs -f job/dtn-manual-1` — look for
   `→ RELEVANT` lines (LLM actually working, i.e. the new key is good) and a
   final `Successfully published`.
3. Watch the node while it runs — `kubectl top nodes`, and the Node Exporter
   Full dashboard. This is the moment the memory risk either shows up or
   doesn't.
4. Confirm `https://news.matflixlab.pl/` renders the card.
5. Confirm the job pod terminated and didn't linger.

### Step 6 — Clean up the Tencent leftovers

Easy to forget, and two of these cost money or create noise:

- **`terraform destroy`** in `dev-tech-news/deploy/terraform/` — a disabled
  VM may still bill for its disk and its public IP. This also removes the
  security group, key pair, VPC and subnet.
- **Remove the dead Prometheus scrape target.** `k8s/apps/monitoring/prometheus/configmap.yaml`
  still has the `node-exporter-tencent` job pointing at `162.62.135.86:9100`
  — it'll sit there as a permanently-down target. (Prometheus now has the
  Reloader annotation, so removing it needs no manual restart.)
- **Decide what happens to `deploy/terraform/` and `deploy/systemd/`** in the
  dev-tech-news repo. They're dead weight once the VM is gone, but they're
  also the record of how that deployment worked. Keeping them is defensible;
  leaving them *undocumented* as dead is not — if they stay, note it in the
  repo's CLAUDE.md so the next reader doesn't assume they're live.
- **Remove the `news` A record** if Step 4 didn't already.

## Out of scope

- Migrating any data from the old VM (explicitly not wanted).
- Alerting on job failure — Alertmanager still isn't installed anywhere in
  this stack. Relevant though: the 401 outage ran silently for five days.
  A k8s CronJob that *fails* is at least visible in ArgoCD/`kubectl`, which
  is better than the systemd timer was, but nothing actively tells you.
  Worth its own small piece of work later.
- Any CI/CD for image builds — still a manual `docker build && push`, same
  as speakstats.

## Execution log

<!-- Append dated entries here per step, same convention as the other plan docs -->
