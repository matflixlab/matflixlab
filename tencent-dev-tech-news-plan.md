# dev-tech-news on Tencent Cloud — Deployment Plan

## Purpose of this document

A task brief for deploying `dev-tech-news` on a standalone Tencent Cloud VM —
not joined to the matflixlab k3s cluster. Like the other plan docs in this repo,
treat this as something to verify against actual state and adjust, not a script
to run blindly. Steps 1–2 need a human (Tencent account, console/API access,
DNS). Once the VM exists and is SSH-reachable, an agent can execute the rest if
given access.

## Why standalone, not a k3s node

Two options were considered for running this on Tencent: joining the VM to the
existing k3s cluster as a second node, or standing it up as a fully independent
host. Standalone won, for two reasons:

1. **Correctness:** Kubernetes CronJobs are triggered by the control-plane's
   controller-manager, not by the local kubelet. If the Tencent box joined as
   an *agent* node while the control plane stayed at home (which it would,
   since ArgoCD lives there), the daily newsletter job would still depend on
   the home laptop/ISP being up — defeating the point of moving the workload.
   Fixing that properly means putting the control plane in the cloud instead,
   which is a much bigger structural change than "run one small app" justifies.
2. **Learning goal:** the point of this exercise is hands-on Tencent Cloud
   experience — CVM provisioning, VPC/security groups, public IP management,
   billing. None of that requires Kubernetes. Standing up k3s+ArgoCD on Tencent
   for a single one-shot daily job would be over-engineering relative to what
   the app currently needs (see [dev-tech-news/CLAUDE.md](../dev-tech-news/CLAUDE.md)
   for its current maturity — no container, no scheduler, no delivery mechanism
   yet).

Plain Docker + a systemd timer on one small VM is proportionate. GitOps-style
management can come later if this graduates into something bigger.

## ⚠️ Region gotcha — read before provisioning anything

**Do not provision the VM in a Mainland China region.** Tencent requires ICP
filing (a Chinese government business/website registration process) to serve
HTTP traffic on a domain from Mainland China regions — this is not realistically
available to an individual outside China and will block Step 6 (exposing the
subdomain) entirely if hit late. Chosen region: **eu-frankfurt** — similar
pricing to Hong Kong/Singapore and lower latency to this newsletter's mostly-
European audience, and it doesn't require ICP filing either (Hong Kong or
Singapore remain fine fallbacks if Frankfurt has instance-type/capacity
issues).

## Architecture

```
GitHub (matflixlab/dev-tech-news)
    │  push to master
    ▼
Self-hosted GitHub Actions runner (runs ON the Tencent VM — no NAT problem,
    │  it already has a public IP, unlike the home node)
    │  rebuilds image, restarts containers
    ▼
Cloudflare DNS (news.matflixlab.pl)
    │  A record → Tencent VM public IP, proxied (orange cloud)
    ▼
Tencent CVM (eu-frankfurt, small instance)
    │  Security Group: 80/443 from Cloudflare IP ranges only, 22 from home IP only
    ▼
Docker
    ├── dev-tech-news container   ← systemd timer, daily: generate PDF → build_site.py
    │                                (adds today's card, prunes anything >7 days old)
    └── static file server (nginx/caddy) ← serves site/ (index.html + last 7 PDFs)
```

Two independent triggers worth keeping distinct:
- **Content cadence** (daily, via systemd timer, Step 4): generates a new
  newsletter and adds a card. Unaffected by code pushes.
- **Code cadence** (on push to master, via the self-hosted runner, Step 8):
  deploys changes to the generator or the site itself — a style tweak to
  `index.html`'s template, a bugfix, etc. Does **not** by itself generate a new
  newsletter or spend LLM calls; it re-runs `build_site.py` against whatever
  PDFs already exist so a template/style change is visible immediately without
  waiting for the next scheduled run.

## Ground rules

- Treat the OpenAI API key and any Tencent API credentials as secrets — never
  commit them. A plain `.env` file on the VM with `chmod 600` is proportionate
  for a single-operator single-VM setup (no SealedSecrets-equivalent needed
  here, that machinery exists for the k3s cluster, not this).
- Deployment updates to this VM are **manual through Step 7, automated from
  Step 8 onward** — do Steps 1–7 by hand first and get a working manual deploy
  before adding the push-to-deploy runner, so there's a known-good baseline to
  fall back to if the automation misbehaves.
- Commit the deployable artifacts (Dockerfile, docker-compose.yml, systemd unit
  files) to the `dev-tech-news` repo under `deploy/` so there's a git record
  even though the deploy step itself isn't automated yet. Never commit the
  `.env` file itself.
- Set a billing alert in the Tencent console before provisioning anything —
  easy to forget on a new cloud account.

## Step 1 — Containerize dev-tech-news

This is needed regardless of where it runs, so do it locally first and verify
before touching Tencent at all.

- Add a `Dockerfile` to the `dev-tech-news` repo. Watch out for `crawl4ai`'s
  Playwright/Chromium dependency — it needs `playwright install --with-deps
  chromium` (or the equivalent apt packages) baked into the image, which makes
  this a heavier image than the app's small source size suggests. Budget VM
  memory accordingly (Step 2).
- Add a `docker-compose.yml` (or a single `docker run` invocation is fine given
  there's only one service) wiring in `OPENAI_API_KEY`/`OPENAI_MODEL` from
  `.env`, mounting the CSV source files and `processed_urls.json` as a volume
  so state persists across runs, and an output volume for the generated PDF.
- Verify locally: `docker compose run --rm dev-tech-news` produces a valid PDF
  with `--skip-crawling --area AI --max-articles 5` (cheap smoke test per
  [dev-tech-news/CLAUDE.md](../dev-tech-news/CLAUDE.md)) before doing a full run.

**Acceptance:** a full, un-flagged run inside the container produces the same
PDF you'd get running it locally with `uv`.

## Step 2 — Provision the Tencent VM with Terraform

Provisioning (this step) and baseline hardening (formerly a separate manual
Step 3) are both handled by
[dev-tech-news/deploy/terraform/](../dev-tech-news/deploy/terraform/) —
Terraform for the VPC/subnet/security-group/CVM, cloud-init (passed as
`user_data`) for the non-root deploy user, sshd hardening, Docker install, and
unattended upgrades. One `terraform apply` produces a ready-to-deploy box.
Full prerequisites and usage: that directory's own README.

- Region: eu-frankfurt (default in `variables.tf`; the ICP-filing gotcha above
  is enforced by a Terraform validation rule, but don't rely on that alone,
  double-check the console too).
- Generate a dedicated SSH keypair for this box first (same pattern as the
  per-repo deploy keys already in use):
  `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_tencent-dev-tech-news -N ""`
- Security Group: Terraform opens SSH (22) only from `var.allowed_ssh_cidr`
  (your real IP, no default — the variable has a validation rule refusing
  `0.0.0.0/0`). 80/443 are intentionally not opened by Terraform at all; that
  happens by hand in Step 6 once the app is ready and the Cloudflare-IP-range
  restriction can be applied at the same time.
- Tencent API credentials go in `TENCENTCLOUD_SECRET_ID`/`TENCENTCLOUD_SECRET_KEY`
  env vars, never in a committed file. Terraform state is local and gitignored
  — fine for one operator, one environment.

**Acceptance:** `terraform apply` succeeds; `terraform output ssh_command`
works and connects as the non-root `deploy` user; `docker run hello-world`
works there without `sudo` (proves the cloud-init hardening ran); SSH from any
other source IP is refused.

## Step 3 — First manual deploy

- Clone `dev-tech-news` onto the VM (read-only deploy key, same pattern as the
  `github-dev-tech-news` SSH alias already used elsewhere).
- Create `.env` with `OPENAI_API_KEY` (and any Tencent-specific config), `chmod
  600`.
- `docker compose build && docker compose run --rm dev-tech-news` for a full,
  real run. Confirm the PDF looks right and `processed_urls.json` persists
  correctly across a second run (no duplicate re-analysis of the same URLs).

**Acceptance:** two consecutive manual runs produce a sane, non-duplicated
newsletter and don't re-spend LLM calls on already-judged URLs.

## Step 4 — Schedule it

- Use a systemd timer rather than cron — better logging via `journalctl -u
  dev-tech-news`, easier to inspect failures.
- The service runs a single entrypoint (`deploy/run_daily.sh` or similar) that
  does two things in sequence: `generate_newsletter.py` (produces today's PDF),
  then `build_site.py` (Step 5 — turns it into a card and prunes old ones).
  Keep these as two separate scripts even though one entrypoint calls both —
  Step 8's push-to-deploy path needs to invoke `build_site.py` alone, without
  re-running the LLM pipeline.
- `deploy/systemd/dev-tech-news.service` + `dev-tech-news.timer` (daily, pick a
  time that doesn't collide with source sites' own maintenance windows — early
  morning UTC is a safe default), committed to the repo under `deploy/`.

**Acceptance:** `systemctl list-timers` shows the next scheduled run; let one
unattended run complete and confirm a new card appeared without anyone SSH'd
in.

## Step 5 — Build the news.matflixlab.pl site (cards, 7-day rotation, matflixlab visual identity)

**Revised 2026-09-08: PDF output is dropped entirely.** The original version of
this step kept `generate_newsletter.py` producing a PDF via reportlab and just
wrapped it in a card linking to that file. Mateusz decided the PDF serves no
purpose once the destination is a web page — `reporters.py` (the whole
reportlab pipeline) is removed, along with the `reportlab` dependency and
`test_reporters.py`. `build_site.py` becomes the sole output renderer.

**Card design (decided):** a single page, one card per day, **full content
inline** (not a summary linking to a separate per-day page) — all 7 days'
article listings visible on `news.matflixlab.pl` directly, grouped by
category/subcategory the same way the old PDF was (AI, AWS & Infrastructure,
CI/CD, Databases, Observability & Monitoring, Conferences). A `<details>`
per-day wrapper (collapsed by default, most recent expanded) is worth
considering purely for page length/usability, but the content itself is not
gated behind a click-through.

**Security note this change introduces:** with a PDF, `sanitizers.escape_markup`
existed to stop malformed reportlab markup from crashing the whole document —
see [dev-tech-news/CLAUDE.md](../dev-tech-news/CLAUDE.md) and the CHANGELOG
incident it's based on. Rendering scraped article content **directly into an
HTML page** raises the stakes on that same invariant from "one bad article
breaks the PDF" to "one bad article runs a script in every visitor's browser."
Every piece of LLM/scraped-derived text (title, summary, subcategory) must be
HTML-escaped at render time — reuse the same "never let raw/unparsed content
through" discipline `ArticleAnalyzer` already enforces, but the escaping
mechanism itself changes from reportlab-markup-escaping to standard
HTML-escaping (e.g. Jinja2 autoescape, or Python's `html.escape` if
hand-rolling the template).

- Add a structured output step to the pipeline (a new small module or a method
  on the existing pipeline result) that serializes each day's analyzed
  articles to `site/data/<YYYY-MM-DD>.json` — grouped the same way the PDF
  grouped them (category → subcategory → list of `{title, url, summary}`).
  This is what `build_site.py` reads; it does not need to touch
  `ArticleAnalyzer`/`connectors.py` at all, only the terminal step that used
  to hand results to `PdfReporter`.

- **Visual identity — reuse, don't reinvent.** Copy the CSS custom properties
  and general aesthetic straight from
  [k8s/apps/landing/html/index.html](k8s/apps/landing/html/index.html): the
  `:root`/`[data-theme="light"]` token blocks (`--bg`, `--surface`, `--border`,
  `--accent: #6366f1`, `--accent2: #22d3ee`, `--text`, `--muted`, `--green`,
  `--yellow`), the monospace font stack (`'SF Mono', 'Fira Code', 'Fira Mono',
  monospace`), the `~/matflixlab$`-style terminal prompt header, and the same
  `favicon.svg`. The goal is that `news.matflixlab.pl` reads as unmistakably
  part of the same site as `matflixlab.pl`, not a separately-branded tool.
- **`scripts/build_site.py`** (new, in `dev-tech-news`), run after every PDF is
  generated:
  1. Copy today's PDF into `site/pdfs/<YYYY-MM-DD>.pdf`.
  2. List `site/pdfs/`, sort by date descending, delete anything beyond the 7
     most recent — this is the entire retention mechanism, no cron-based
     cleanup needed separately.
  3. Regenerate `site/index.html` from a template
     (`site/template.html`, holding the copied matflixlab styling) with one
     card per retained PDF: date, a short stat line if easy to source (e.g.
     `RunStats.summary_line()` already produced per run — see
     [dev-tech-news/CLAUDE.md](../dev-tech-news/CLAUDE.md) — worth persisting
     alongside each dated PDF, e.g. `site/pdfs/<date>.json`, so the card can
     show "N articles, M rejected" without re-parsing the PDF), and a link to
     that day's PDF.
  4. When there are zero PDFs yet (first-ever deploy), render an empty state
     rather than erroring.
- The nginx/Caddy container from the original plan now serves `site/` as its
  docroot — no server-side logic needed at request time, it's all static
  files regenerated on each run.

**Acceptance:** after 8+ daily runs (or 8 manual invocations with `--skip-crawling
--max-articles 5` for a fast smoke test), `site/pdfs/` contains exactly 7
files, `site/index.html` shows exactly 7 cards in date-descending order, and
visually matches matflixlab.pl's theme (check both light and dark mode via the
existing `data-theme` toggle mechanism).

## Step 6 — DNS and exposure

- Cloudflare DNS: `news.matflixlab.pl` → A record → VM's public IP, proxied
  (orange cloud). This is simpler than the Cloudflare Tunnel setup used at home
  — that exists specifically because the home node has no public IP; the
  Tencent VM does, so a normal proxied DNS record is the standard approach and
  a good contrast to learn against the tunnel pattern.
- Open the Security Group's 80/443 to Cloudflare's published IP ranges only
  (never the origin's raw IP) — this is the point of doing this via Tencent's
  own firewall construct rather than skipping straight to "just open it."
- Set Cloudflare SSL/TLS mode to **Full (strict)** with a Cloudflare Origin
  Certificate installed on the VM, for end-to-end encryption rather than
  terminating TLS at Cloudflare and going plaintext to the origin.

**Acceptance:** `https://news.matflixlab.pl/` resolves and serves the card
site (from Step 5), the most recent card's PDF link works; a direct request to
the VM's raw IP on 80/443 from outside Cloudflare's ranges is refused.

## Step 7 — Cheap dead-man's-switch monitoring

matflixlab's own monitoring stack has no alerting yet (Alertmanager isn't
installed — see [gitops-hardening-plan.md](gitops-hardening-plan.md)), and this
VM won't be part of that stack at all. Rather than standing up Prometheus here
too, add a single `curl` ping to [healthchecks.io](https://healthchecks.io)
(free tier) at the end of the systemd service — it alerts you (email/webhook)
if the daily job *doesn't* run or exits non-zero, which is the actual failure
mode worth catching for a single scheduled job.

**Acceptance:** manually fail a run (e.g. temporarily rename `.env`) and
confirm healthchecks.io fires an alert.

## Step 8 — Push-to-deploy via self-hosted GitHub Actions runner

Unlike speakstats' deferred CI/CD plan ([speakstats-cicd-plan.md](speakstats-cicd-plan.md)),
this one is explicitly requested: a push to `master` on `dev-tech-news` should
update prod without SSH'ing in. It's a smaller lift here than the speakstats
plan for one reason: **this VM has a public IP**, so the NAT/Tailscale problem
that made the speakstats runner plan await self-hosting doesn't exist here —
the runner lives directly on the box being deployed to.

- Install a self-hosted GitHub Actions runner on the Tencent VM as a systemd
  service (`./config.sh --url https://github.com/matflixlab/dev-tech-news
  --token <TOKEN>` then `svc.sh install && svc.sh start`), labeled e.g.
  `[self-hosted, tencent-dev-tech-news]`.
- Add `.github/workflows/deploy.yml` to `dev-tech-news`, triggered on `push:
  branches: [master]`, `runs-on: [self-hosted, tencent-dev-tech-news]`:
  1. Checkout.
  2. `docker compose build` (picks up code/Dockerfile changes).
  3. `docker compose up -d --force-recreate` (or restart the systemd unit) —
     this only affects the containers, it does not run the newsletter
     pipeline.
  4. Re-run `build_site.py` alone (no `generate_newsletter.py`) so any
     template/style change in the push is reflected on the live site
     immediately, against the PDFs that already exist — without spending an
     LLM call.
  5. A smoke test: `curl -sf https://news.matflixlab.pl/` returns 200.
- Same secrets-handling note as elsewhere in this plan: the runner needs no
  extra credentials beyond what's already on the box (`OPENAI_API_KEY` in
  `.env`) since it's building and running the same image the systemd timer
  uses — it doesn't need repo push access or anything more privileged than
  read+build.

**Acceptance:** push a trivial change (e.g. a copy tweak in
`site/template.html`) to `master` and confirm it's live on
`https://news.matflixlab.pl/` within the workflow's run time, with no manual
`ssh`/`docker` command run by hand.

## Out of scope for this plan

- Bringing this workload under the matflixlab k3s cluster/ArgoCD — explicitly
  rejected above.
- Multi-region/HA for this VM — it's one box for one daily batch job; not
  warranted.
- Access control on `news.matflixlab.pl` (public vs. restricted to the team)
  — not decided; default to public unless told otherwise, revisit with
  Cloudflare Access if needed later.

## Execution log

<!-- Append dated entries here per step, same convention as gitops-hardening-plan.md -->

### 2026-09-08 - Step 2: Terraform provisioning ✅

**Provisioned:** VPC (`vpc-q6vh6a4n`), subnet (`subnet-7bqa6g36`), security group (`sg-a4otijt4`, SSH from home IP only), key pair (`skey-jt2t79n5`), CVM instance (`ins-b7twf9d8`, `eu-frankfurt-1`, `SA2.MEDIUM4`, 20GB). Public IP `162.62.135.86`. SSH alias `tencent-dev-tech-news` added to `~/.ssh/config`.

**Real-world fixes needed against the actual Tencent API/provider (plan/doc assumptions didn't all survive contact with `terraform plan`):**
- `data.tencentcloud_images`: `os_name` and `image_name_regex` are mutually exclusive — dropped `os_name`.
- Security group `description` has a **100-character limit** — the original description was too long, shortened it.
- `tencentcloud_security_group_rule` for `ip_protocol = "all"` (egress) rejects an explicit `port_range = "all"` — omit `port_range` entirely for all-protocol rules.
- `tencentcloud_key_pair.key_name` only allows letters/numbers/underscore — no hyphens (`${var.instance_name}-deploy` → `${replace(var.instance_name, "-", "_")}_deploy`).
- `security_groups` argument on `tencentcloud_instance` is deprecated in favor of `orderly_security_groups` — switched.
- **Tags:** `project` is a **reserved tag key** on Tencent Cloud (collides with their native project-management feature) — `FailedOperation.TagKeyReserved`. Renamed to `app`/`managed_by`.
- **CAM payment rights are separate from CAM action policies.** `QcloudCVMFullAccess`/`QcloudVPCFullAccess`/`QcloudTagFullAccess` were sufficient for every free resource (VPC/subnet/SG/key pair) but the billable `RunInstances` call failed with `UnauthorizedOperation` until the sub-account was also granted `QcloudCVMFinanceAccess` (CVM-scoped payment rights) — separately from `QcloudFinanceBudgetReadOnlyAccess`/`QcloudFinanceBillReadOnlyAccess`, which are read-only and don't grant purchase rights.
- **Account balance** is checked independently of CAM permissions — `InvalidAccount.InsufficientBalance` until the account was topped up (~$5).
- **`user_data_raw` bug (the significant one):** it was set to `base64encode(templatefile(...))`, but per the provider docs `user_data_raw` expects **plain text** (`user_data` is the one that wants pre-encoded base64) — the value was getting double-encoded, so cloud-init received a literal base64 string it didn't recognize as `#cloud-config` and silently did nothing with it (logged as "Unhandled non-multipart userdata"). First instance (`ins-mp5nqexu`) booted with none of the cloud-init hardening applied; fixed and force-replaced (`terraform apply -replace=`) to get a clean boot.
- Saved `-out=tfplan` files went stale almost immediately between `plan` and `apply` in this session (unclear root cause, possibly a volatile provider-computed field) — worked around by planning and applying in the same command rather than relying on a saved plan file across separate steps.

**Verified on the rebuilt instance:**
- `deploy` user: SSH key auth works, in `sudo`+`docker` groups, no password set.
- sshd hardening (`/etc/ssh/sshd_config.d/99-hardening.conf`): root login and password auth both disabled.
- Docker: `docker run hello-world` succeeds.
- `ufw`: active, default-deny incoming, only 22/tcp allowed (belt-and-suspenders alongside the Security Group, as intended).
- `fail2ban` and `unattended-upgrades`: installed.
- Disk: 6.3G used / 20G total after the full apt upgrade — confirms the 20GB sizing decision was correct, comfortable headroom.
- **Known-benign cosmetic issue:** `cloud-init status` permanently reports `error`, not `done` — the failing module is Tencent's own platform-injected password-reset step trying `chpasswd` for a `ubuntu` user that doesn't exist (our `users:` list only creates `deploy`). Harmless since password auth is disabled entirely, but don't be alarmed by the "error" status on future checks — it's not our config failing.

**Deviations:** region is `eu-frankfurt` per Mateusz's call (similar pricing to Hong Kong/Singapore, better latency for a mostly-European audience), not Hong Kong/Singapore as the plan originally drafted — the ICP-filing constraint still applies equally and is satisfied. Disk sized to 20GB, not the plan's original 50GB, after a cost/usage evaluation (50GB would have cost an extra ~$1.50/month for headroom that wasn't needed).

---

### 2026-09-08 - Step 5: implemented and tested locally (no PDF) ✅

Built on `feature/html-news-site` in `dev-tech-news` (PDF behavior preserved
on `master-pdf`, pushed to origin — see Step 5's revised design above for why
PDF was dropped and what the card content decision was).

**Implemented:**
- `src/dev_tech_news/site_builder.py`: `write_daily_data()` (sanitizes +
  groups + writes `site/data/<date>.json`) and `build_site()` (prunes beyond
  7 days, renders `index.html`, no network/LLM — this is what the future
  push-to-deploy step calls alone).
- `scripts/build_site.py`: thin CLI wrapper around `build_site()`.
- `scripts/generate_newsletter.py`: `--output <pdf>` replaced with
  `--site-dir`/`--max-days-published`; calls `write_daily_data` +
  `build_site` instead of `PdfReporter`. The old "skip publishing entirely on
  zero articles" behavior was replaced with "publish an empty card for the
  day" — more honest for a calendar of daily cards than a silent gap.
- Removed: `reporters.py`, `tests/test_reporters.py`, `reportlab` dependency,
  `utils.pdf_page_count`. Renamed `PdfReportAdapter` → `ReportAdapter`.
- `tests/test_site_builder.py`: 12 new tests (escaping/XSS, raw-page
  rejection, truncation, URL-scheme validation, 7-day pruning, empty-state,
  Conferences flat rendering, most-recent-day-expanded).

**Verified:**
- Full suite: `170 passed` (up from 158 before this change — net of the 12
  new tests and the 4 removed PDF-specific ones).
- Real end-to-end run against live RSS feeds (`--skip-classifiers`, no
  OpenAI calls): correctly rejected a **41KB raw scrape** from a real
  ClickHouse blog page and fell back to its title — the exact 13.08.2026
  failure mode, caught live rather than only in a synthetic test.
- XSS escaping verified against a `<script>` payload in both a unit test and
  manually — rendered as `&lt;script&gt;`, never executes.
- `scripts/build_site.py` re-render verified independently (no LLM/network
  calls, rebuilds `index.html` from existing JSON only).

**Not yet done:** the matflixlab-visual-identity CSS was hand-written to
match the landing page's tokens (colors, font stack, prompt-style header) but
has not been visually compared side-by-side in a browser against
matflixlab.pl — worth doing before Step 7 (DNS/exposure). Containerizing
this (Step 1) and deploying it to the actual Tencent VM (Step 3 onward)
haven't happened yet — this step only covers local implementation/testing.

**Deviations:** none from the revised (no-PDF, inline-content) design agreed
before implementation.

---

### 2026-09-08 - Step 1: Containerized and verified locally ✅

Added `Dockerfile`, `.dockerignore`, and `docker-compose.yml` to
`dev-tech-news` (still on `feature/html-news-site`).

**Non-obvious things this needed:**
- `crawl4ai-setup` (the documented post-install step) is just
  `playwright install --with-deps --force chromium` plus creating
  `~/.crawl4ai/` — confirmed by reading `crawl4ai/install.py` rather than
  guessing, since running it manually on the dev workstation earlier had
  needed `sudo` (it detects non-root and elevates). Running it as root
  *during the build* (Docker's default) sidesteps that entirely.
- **Root-install / non-root-run mismatch:** installing the browser as root
  then dropping to a non-root `appuser` for runtime would normally leave
  Playwright looking in the wrong `$HOME/.cache` and not finding the
  browser it just installed. Fixed with `ENV PLAYWRIGHT_BROWSERS_PATH=/ms-playwright`
  set *before* the install — a fixed path independent of whichever user is
  active, chowned to `appuser` afterward.
- `ENV UV_NO_SYNC=1` after switching to the runtime user — without it, every
  `uv run` re-verifies the project sync (harmless here, ~20ms, but means a
  production container could attempt network dependency resolution at
  runtime for no reason; dependencies are already frozen at build time).
- Local Docker context gotcha (specific to this dev workstation, not the
  Tencent VM): `docker compose build` initially failed with "Cannot connect
  to the Docker daemon" — the active context (`desktop-linux`) pointed at an
  inactive Docker Desktop socket, while a real `dockerd` was already running
  and reachable via the `default` context. `docker context use default`
  fixed it.

**Verified (acceptance criteria from the plan):**
- `docker compose run --rm dev-tech-news python scripts/generate_newsletter.py
  --skip-crawling --skip-classifiers --area AI --max-articles 3 --site-dir site`
  — real RSS fetch + crawl4ai/Playwright scraping *inside the container*
  worked, including the raw-page rejection path firing correctly on real
  scraped content (same check verified on the host earlier).
- `docker compose run --rm dev-tech-news python scripts/build_site.py
  --site-dir site` — the standalone rebuild path (no LLM/network) also
  verified inside the container, confirming both CMD-override invocation
  modes the push-to-deploy step (Step 8) will need.
- Bind-mounted `site/`, `processed_urls.json`, and the three CSVs all
  read/write correctly from the non-root container user back to the host
  filesystem, with matching ownership (both happened to be UID 1000 —
  worth a quick re-check on the actual Tencent VM's `deploy` user, though
  not expected to differ).
- Added `site/` to `.gitignore` (generated output, not source) — it was
  briefly untracked-but-not-ignored during this testing.

**Not yet done:** a real full run with actual LLM classification (no
`--skip-classifiers`) hasn't been verified inside the container, since that
needs real OpenAI credits — the code path is identical to what already runs
on the host today and isn't something Docker packaging could plausibly
affect, so this is a low-risk gap, not a skipped acceptance criterion.

**Deviations:** none from the plan's Step 1 design.

---

### 2026-09-08 - Step 3: first deploy — hit a real incident, VM crashed twice ⚠️

**Setup completed:** dedicated read-only deploy key generated on the VM
(`github-dev-tech-news` SSH alias there), repo cloned on
`feature/html-news-site`, `.env` created **directly on the VM by Mateusz via
SSH** (deliberately never passed through this conversation, after an earlier
close call — see note below), `processed_urls.json`/`site/` seeded, image
built successfully.

**Incident:** the first real (unflagged) `docker compose run --rm
dev-tech-news` — default args, no `--crawl-concurrency` override — pegged
CPU at 100% and made SSH to the VM completely unresponsive (confirmed by both
Mateusz and independently from this session — a background `docker ps` check
also hung). Required a console restart (More Actions → Restart) to recover.
Retried with `--crawl-concurrency 2`, watched via a load-average monitor this
time — load climbed past 11 and SSH died again, **during the website-crawling
phase**, before the RSS phase (which had respected the flag) showed any
danger.

**Root cause found and fixed** (full detail: `dev-tech-news/CHANGELOG.md`
§0.3.1): `--crawl-concurrency` only ever applied to `RssConnector`.
`WebsiteCrawlingConnector`/`ConferenceCrawlingConnector` each hardcoded their
own limit (5) and additionally called a markdown-extraction helper with its
own unrelated default (8) — and because that inner call happens *inside* the
outer per-site semaphore, the real worst case was `crawl_concurrency²` (up to
40 concurrent headless-Chromium sessions), completely independent of the
flag. Fixed by moving `crawl_concurrency` into the actual shared base class
and threading it through every call site; added `tests/test_connectors.py`
to pin the wiring. Pushed to `feature/html-news-site` (commits `0a499ed`,
`7b65005` — the latter adds 2GB swap via cloud-init as a defense-in-depth
safety net for *future* memory spikes, not a fix for this one).

**Process note — a mistake worth recording:** during recovery, an attempt to
find a possibly-lost `.env` via VS Code's local history **printed two real
OpenAI API keys in plaintext into the conversation transcript**. Both were
told to be revoked immediately and replaced. Going forward in this project,
secrets are created directly on the target host by Mateusz over his own SSH
session — never generated, displayed, or piped through by the agent, even
via a redirect-without-echo. This is now the standing convention, not a
one-time fix.

**Not yet done:** the real full run with actual LLM classification still
hasn't successfully completed end-to-end on the VM — both attempts crashed
mid-run. Next attempt should use the fixed code with `--crawl-concurrency 2`
(now actually enforced everywhere) and confirm completion before considering
Step 3 done. Concurrency 2 still has a `2²=4` worst case per the
not-fully-fixed multiplicative issue noted in the CHANGELOG — acceptable for
now, worth revisiting (single shared semaphore) if 2 still isn't safe enough.

**Deviations:** none from the plan's own design — this was an unanticipated
real-world resource constraint on the chosen instance size, not a plan
error, though `SA2.MEDIUM4` (2 vCPU/4GB) is clearly closer to the edge for
this workload than hoped.

---

### 2026-09-08 - Step 3 complete: first real run succeeded, plus two more findings ✅

With the concurrency fix pulled and rebuilt, the real (unflagged) pipeline
completed successfully — monitored end-to-end via a load/memory watcher this
time rather than found out about problems after the fact. **103 articles
published**, 96 real LLM calls (~293K tokens), zero rejected raw pages,
dedup working correctly, `site/index.html` + `processed_urls.json` both
written. `curl localhost` on the VM returns `200` with real content
(previously `403` — empty site before any successful run). Step 3's actual
acceptance criterion (a full real run, not just a flagged test) is met.

**Two more operational findings from this run, both fixed:**

1. **Dynamic home IP locked us out.** `allowed_ssh_cidr` was still pointed
   at the IP set during initial provisioning; the ISP rotated it, and SSH
   became unreachable — easily mistaken for another VM crash, but the VM
   itself was fine (confirmed once back in: `uptime` showed a clean 8-minute
   idle boot). Fixed by updating `terraform.tfvars` and re-applying. Mateusz
   asked to open SSH to `0.0.0.0/0` to avoid repeating this — declined (the
   Terraform validation rule exists specifically to block that, and a public
   SSH port gets brute-forced constantly regardless of key-only auth being
   safe against actual compromise). Agreed instead to consider **Tailscale**
   as the permanent fix (removes the dependency on any public IP for access
   entirely) as a follow-up, not yet implemented.
2. **A duplicate container was running concurrently**, doubling load and
   risking corrupt writes to the shared `processed_urls.json`/`site/`. Root
   cause: an earlier attempt was killed via a local `timeout 15` wrapper on
   the *SSH client*, but the remote `docker compose run` process survived
   the disconnect and kept running server-side. Caught via `docker ps -a`
   creation timestamps (two containers ~40s apart) and killed the stray one.
   Process lesson: a local `timeout` on an SSH command does not reliably
   kill the remote process it invoked — don't assume it does.

**DNS/TLS status:** `news.matflixlab.pl` DNS record already existed
(proxied through Cloudflare), but requests were returning `502` — Cloudflare
was almost certainly set to Full/Full(strict) SSL mode, trying HTTPS to an
origin that only had port 80. Added an nginx TLS listener on 443
(`deploy/nginx/nginx.conf` + `docker-compose.yml`) using a Cloudflare Origin
Certificate — Mateusz creating the cert/key directly on the VM himself, same
convention as `.env`. **Not yet verified end-to-end** (waiting on the
certificate being placed) — do that next, then re-test
`curl https://news.matflixlab.pl/`.

**Deviations:** SSH access hardening (Tailscale) and Step 8/9 (push-to-deploy
runner) both still pending — this entry only closes out Step 3 and the
immediate DNS/TLS troubleshooting, not the whole plan.

---

### 2026-09-08 - Step 6 (exposure) complete: news.matflixlab.pl is live ✅

Three separate, stacked problems had to be found and fixed before the
Cloudflare→origin path actually worked — worth recording all three since any
one of them alone would have looked like "the fix," but none of them alone
was sufficient:

1. **Wrong routing mechanism.** `news.matflixlab.pl` was initially configured
   as a **Cloudflare Tunnel route** (`http://162.62.135.86:80` as the tunnel's
   Service URL) — the same mechanism used for the home-hosted subdomains.
   This meant traffic went browser → Cloudflare edge → `cloudflared` on the
   **home server** → out to the Tencent VM, making the Tencent-hosted site
   depend on home-server/home-internet uptime for no reason, and it doesn't
   match the security group (which only allows Cloudflare's own edge IP
   ranges — the home server's outbound IP isn't in that list, so the
   connection was silently dropped). Fixed by replacing it with a plain
   proxied **DNS A record** (`news` → `162.62.135.86`), which makes Cloudflare's
   edge connect directly, from an IP that's actually in the allowlisted range.
   The other subdomains keep using the Tunnel — no conflict, different DNS
   records.
2. **`ufw` never had 80/443 opened.** The cloud-init hardening only ever
   explicitly allowed 22/tcp; nginx being added later (Step 5/6) never got a
   corresponding `ufw allow`. This is independent of the Security Group and
   blocked *all* external traffic to 80/443 regardless of source IP — the
   Security Group being correctly scoped to Cloudflare's ranges didn't matter
   because the host firewall behind it was still closed. Symptom was
   Cloudflare error 522 (connection timeout) even though `curl localhost`
   on the VM itself worked fine (loopback doesn't hit the same filtering
   path). Fixed with `ufw allow 80/tcp` / `ufw allow 443/tcp` — not scoped to
   Cloudflare's ranges specifically, since the Security Group is already the
   authoritative IP restriction here (matches the existing "ufw is
   belt-and-suspenders, not load-bearing" design note in cloud-init.yaml.tftpl).
   **Not yet reflected in the Terraform template** — cloud-init only runs on
   first boot, so a future fresh VM would hit this same gap; worth adding
   `ufw allow 80/tcp`/`443/tcp` to `cloud-init.yaml.tftpl` as a follow-up.
3. **Cloudflare's SSL/TLS mode was "Automatic," which had detected/pinned
   "Full."** With nginx TLS removed in favor of Flexible mode (see the
   revert commit `8350e54` — Cloudflare's edge terminates TLS, origin gets
   plain HTTP, which is safe here specifically because the Security Group
   already restricts 80/443 to Cloudflare's ranges), "Automatic" mode was
   still enforcing HTTPS-to-origin against a port 443 nothing was listening
   on. Symptom was Cloudflare error 521 (origin refused the connection) —
   different from 522, and the distinction is what pointed at this being an
   application/TLS-mode issue rather than another network/firewall gap.
   Fixed by explicitly selecting **Flexible** (not "Automatic") in
   SSL/TLS → Overview. This is a **zone-wide** setting on the Free plan, so
   it also applies to the Tunnel-routed subdomains — verified no regression
   there (`speakstats.matflixlab.pl` still returns `200` after the change;
   Tunnel-routed hosts use their own Service URL scheme, not this setting).

**Verified:** `https://news.matflixlab.pl/` returns `200` with the real
generated site (correct title, real day-cards from the successful Step 3
run). `https://speakstats.matflixlab.pl/` still returns `200` (regression
check on the zone-wide SSL mode change).

**Not yet done:** `ufw` port-opening isn't yet folded back into
`cloud-init.yaml.tftpl` (see finding #2). Tailscale for SSH access, and
Step 8/9 (push-to-deploy runner), are still open.
