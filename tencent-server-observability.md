# Tencent server observability — plan

## Goal

Get the Tencent CVM's host metrics (CPU/RAM/disk/network) into the same
**Node Exporter Full** Grafana dashboard already used for `matflix-server`,
scraped by the home-hosted Prometheus (`grafana.matflixlab.pl` /
`k8s/apps/monitoring/prometheus/`).

## Networking approach: reuse the SSH IP allowlist, not Tailscale

`node_exporter` itself is trivial — the real question is how Prometheus
(running in a pod inside the home k3s cluster, which has no public IP)
reaches `node_exporter` on the Tencent VM (which does).

A Tailscale mesh was the first design considered here (avoids any new public
exposure, avoids dynamic-IP staleness) but was explicitly deprioritized:
Mateusz wanted something simple today over the more robust long-term fix.
**Chosen instead:** one more Tencent Security Group rule for port 9100,
reusing the **same `allowed_ssh_cidr` Terraform variable** already in place
for SSH — same home IP, same rule shape, zero new infrastructure to learn or
maintain.

**Tradeoffs, accepted deliberately, not overlooked:**
- Same dynamic-IP staleness SSH already has — when the home ISP rotates the
  IP, this rule goes stale the same moment the SSH rule does, and both get
  fixed by the same `terraform apply` (see `tencent-dev-tech-news-plan.md`
  for that exact failure mode already happening once).
- `node_exporter` has no authentication of its own — the Security Group's IP
  restriction *is* the only protection. Acceptable here because the data
  exposed (CPU/RAM/disk/network stats) is low-sensitivity — not secrets, not
  credentials — and it's restricted to one IP, not the open internet.
- Tailscale (or another mesh/tunnel) remains the better long-term answer,
  particularly since it would also fix the SSH dynamic-IP problem at the
  same time — tracked as a follow-up in `tencent-dev-tech-news-plan.md`, not
  abandoned, just not today's problem to solve.

## Steps

### Step 1 — Open port 9100 on the Security Group

Added `tencentcloud_security_group_rule.node_exporter_in` to
`deploy/terraform/main.tf`, reusing `var.allowed_ssh_cidr` — same CIDR as
the SSH rule, same VM, same instance.

**Status: done.** Applied via `terraform apply`.

### Step 2 — Run `node_exporter` on the Tencent VM

Mirrors the home DaemonSet's exact flags
(`k8s/apps/monitoring/node-exporter/daemonset.yaml`) for consistent metric
semantics between the two nodes — same image version, same
`--path.sysfs`/`--path.rootfs` convention. Deliberately **not** added to
`dev-tech-news`'s own `docker-compose.yml` — this is host-level
infrastructure, unrelated to that app's deploy lifecycle.

```yaml
# ~/node-exporter/docker-compose.yml on the VM
services:
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - /sys:/host/sys:ro
      - /:/host/root:ro
    command:
      - --path.sysfs=/host/sys
      - --path.rootfs=/host/root
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
```

**Status: done.** `docker compose up -d`, verified `curl
http://localhost:9100/metrics` on the VM and `curl
http://162.62.135.86:9100/metrics` from the allowlisted home IP both return
real metrics.

### Step 3 — Add the target to Prometheus's scrape config

Added a `static_configs` block directly into the **existing** `node-exporter`
job in `k8s/apps/monitoring/prometheus/configmap.yaml` (Prometheus supports
combining `kubernetes_sd_configs` and `static_configs` in one job) — keeping
the same `job_name` means this target satisfies the Node Exporter Full
dashboard's existing `job="node-exporter"` filter with no dashboard changes.
A `location: tencent-cloud` label was added purely so ad-hoc queries can
distinguish home vs. cloud at a glance — the dashboard's "Nodename" dropdown
doesn't need it, since that's driven by `node_uname_info`, a metric
`node_exporter` reports about itself using the real hostname.

**Status: done**, committed to `configmap.yaml`.

`k8s/apps/monitoring/prometheus/deployment.yaml` has **no Reloader
annotation** (confirmed — unlike `landing`/`grafana`/`umami`), so the new
scrape config won't take effect from the ConfigMap sync alone. After ArgoCD
syncs: `kubectl -n monitoring rollout restart deployment/prometheus`. Worth
adding the Reloader annotation while touching this — small, obvious
follow-up, same fix already applied elsewhere, never extended to Prometheus.

### Step 4 — Verify

1. Prometheus's targets page shows `162.62.135.86:9100` `UP` under the
   `node-exporter` job.
2. Grafana → **Node Exporter Full** dashboard → "Nodename"/"Instance"
   dropdown includes the Tencent host; graphs populate with real data.
3. Sanity-check the numbers against `free -h`/`uptime` run live on the VM,
   to confirm labels didn't get crossed with the home server's data.

**Status: pending** — needs the ConfigMap synced + Prometheus restarted,
then a look at the dashboard.

## Out of scope for this plan

- Tailscale (either for this or for SSH) — deliberately deferred, tracked in
  `tencent-dev-tech-news-plan.md`.
- Authentication/TLS on `node_exporter` itself (`--web.config.file`) — would
  reduce reliance on the IP restriction being the only protection, but adds
  setup complexity that was explicitly traded away for speed today.
- Alerting on the new metrics — Alertmanager still isn't installed anywhere
  in this stack, pre-existing gap.

## Execution log

<!-- Append dated entries here per step, same convention as the other plan docs -->
