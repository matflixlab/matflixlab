# matflixlab

Production-grade self-hosted homelab infrastructure running on k3s. GitOps-managed deployments with full observability stack.

**Live:** [matflixlab.pl](https://matflixlab.pl)

---

## Overview

Single-node k3s cluster hosted on a spare laptop (Ubuntu 22.04, 8GB RAM, 4-core CPU). All services are containerized, deployed via GitOps with ArgoCD, and exposed through Cloudflare Tunnel — no open ports, no public IP required.

### Tech Stack

- **Orchestration:** k3s (lightweight Kubernetes)
- **GitOps:** ArgoCD — automatic sync from this GitHub repo
- **Ingress:** Traefik 3.6 (built-in k3s)
- **Networking:** Cloudflare Tunnel (cloudflared)
- **Observability:** Prometheus, Grafana, Loki, Tempo (full LGTM stack)
- **Configuration:** Kustomize manifests

---

## Architecture

```
Internet
    │
    ↓
Cloudflare (DNS + CDN + WAF)
    │
    ↓
Cloudflare Tunnel (cloudflared)
    │
    ↓
Traefik Ingress Controller
    │
    ├─→ matflixlab.pl          → landing (static nginx)
    ├─→ speakstats.matflixlab.pl → speakstats (Next.js + whisper.cpp)
    ├─→ jellyfin.matflixlab.pl  → jellyfin (media server)
    ├─→ umami.matflixlab.pl     → umami (analytics)
    ├─→ grafana.matflixlab.pl   → grafana (monitoring dashboards)
    └─→ argocd.matflixlab.pl    → argocd (GitOps UI)
```

---

## Services

### Public Services

| Service | URL | Description |
|---------|-----|-------------|
| **Landing** | [matflixlab.pl](https://matflixlab.pl) | Personal portfolio & homelab showcase |
| **SpeakStats** | [speakstats.matflixlab.pl](https://speakstats.matflixlab.pl) | Speech analysis tool (whisper.cpp backend) |
| **Jellyfin** | [jellyfin.matflixlab.pl](https://jellyfin.matflixlab.pl) | Self-hosted media server |
| **Umami** | [umami.matflixlab.pl](https://umami.matflixlab.pl) | Privacy-focused web analytics |
| **Grafana** | [grafana.matflixlab.pl](https://grafana.matflixlab.pl) | Monitoring dashboards (anonymous read-only) |
| **ArgoCD** | [argocd.matflixlab.pl](https://argocd.matflixlab.pl) | GitOps deployment UI |

### Internal Services

- **Prometheus** — metrics scraping (30s interval, 30d retention)
- **Loki** — log aggregation (7d retention, 10Gi)
- **Tempo** — distributed tracing (Traefik OTLP, 24h retention)
- **Promtail** — log shipper (DaemonSet)
- **node-exporter** — host metrics (CPU, RAM, disk, network)
- **kube-state-metrics** — Kubernetes object metrics

---

## Repository Structure

```
matflixlab/
├── k8s/
│   ├── apps/
│   │   ├── landing/           # Static portfolio site (nginx)
│   │   ├── speakstats/        # Speech analytics (Next.js + whisper.cpp)
│   │   ├── jellyfin/          # Media server
│   │   ├── umami/             # Analytics (app + postgres)
│   │   └── monitoring/        # Prometheus, Grafana, Loki, Tempo, Promtail
│   ├── infrastructure/
│   │   ├── argocd/            # GitOps controller
│   │   ├── traefik/           # Ingress config (HelmChartConfig)
│   │   └── cloudflared/       # Cloudflare Tunnel
│   └── base/
│       └── kustomization.yaml # Root kustomization
├── deploy.md                  # Technical deployment notes
├── tempo-plan.md              # Tempo implementation plan (completed)
├── speakstats-cicd-plan.md    # CI/CD pipeline plan (in progress)
├── cv-download-email-confirmation.md  # Cloudflare Workers plan (future)
└── README.md                  # This file
```

---

## Deployment

### GitOps Flow

```
1. Code change → git push to GitHub
2. ArgoCD auto-sync (3min poll interval)
3. Kustomize build manifests
4. kubectl apply to k3s cluster
5. Traefik routes traffic
```

**Manual sync:**
```bash
# Trigger immediate ArgoCD refresh
kubectl patch application matflixlab -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

### Kustomize Structure

All apps use Kustomize overlays:

```yaml
# k8s/apps/<service>/kustomization.yaml
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - configmap.yaml
  - pvc.yaml (if needed)
```

Root kustomization at `k8s/base/kustomization.yaml` aggregates all apps.

---

## Observability

### Metrics (Prometheus + Grafana)

- **Prometheus:** `prometheus.monitoring.svc.cluster.local:9090`
- **Grafana:** [grafana.matflixlab.pl/dashboards](https://grafana.matflixlab.pl/dashboards)
- **Scrape interval:** 30s
- **Retention:** 30 days
- **Storage:** 5Gi PVC

**Dashboards:**
- Node Exporter Full (`rYdddlPWk`) — host CPU, RAM, disk, network
- Kubernetes Views Pods (`k8s_views_pods`) — pod status, restarts
- matflixlab Cluster (`matflixlab-cluster`) — container resources, limits
- Traefik (`n5bu_kv45`) — HTTP request rate, latency, status codes
- Loki Kubernetes Logs (`o6-BGgnnk`) — log search & filtering

### Logs (Loki + Promtail)

- **Loki:** `loki.monitoring.svc.cluster.local:3100`
- **Promtail:** DaemonSet on every node
- **Retention:** 7 days
- **Storage:** 10Gi PVC

**LogQL examples:**
```logql
# All logs from matflixlab namespace
{namespace="matflixlab"}

# Errors in speakstats
{namespace="matflixlab", app="speakstats"} |= "error"

# ArgoCD sync events
{namespace="argocd"} |= "sync"
```

### Traces (Tempo + Traefik)

- **Tempo:** `tempo.monitoring.svc.cluster.local:3200`
- **OTLP receiver:** `:4317` (gRPC)
- **Retention:** 24 hours
- **Storage:** 5Gi PVC

Traefik sends OTLP traces for every HTTP request. View in Grafana → Explore → Tempo.

**Trace → Logs correlation:** Click trace in Tempo → "View related logs" opens Loki with matching traceID.

---

## Key Technologies

### k3s

Lightweight Kubernetes distribution. Installed with:
- Traefik ingress (built-in)
- Local-path storage provisioner
- Metrics server

**Version:** v1.28.x  
**Config:** Single-node, no HA

### ArgoCD

GitOps continuous delivery tool. Watches this GitHub repo and auto-syncs changes to cluster.

**Application:** `matflixlab` (namespace: `argocd`)  
**Repo:** `https://github.com/matflixlab/matflixlab.git`  
**Path:** `k8s/`  
**Sync policy:** Automated (prune: false, self-heal: true)

### Traefik

Ingress controller routing external traffic to services.

**Config:** `k8s/infrastructure/traefik/traefik-config.yaml` (HelmChartConfig)  
**Features:**
- HTTP/HTTPS on ports 80/443 (hostPort)
- OTLP tracing to Tempo
- Prometheus metrics on `:9100`

### Cloudflare Tunnel

Exposes cluster to internet without open ports or public IP.

**Tunnel:** `matflixlab-tunnel`  
**Config:** `/etc/cloudflared/config.yml` on host  
**Routes:** All `*.matflixlab.pl` subdomains → `http://localhost:80`

**DNS:** Managed in Cloudflare dashboard, CNAME records point to tunnel.

---

## Network & Security

### Ingress Flow

```
User request → Cloudflare CDN → Cloudflare Tunnel → Traefik → Service → Pod
```

- **No open ports** on router/firewall
- **No public IP** required
- **Cloudflare WAF** protects against DDoS, bots
- **TLS termination** at Cloudflare (Flexible SSL mode)

### Authentication

- **Grafana:** Anonymous Viewer role (read-only dashboards)
- **ArgoCD:** OAuth via GitHub (admin only)
- **Umami:** Username/password
- **Jellyfin:** Username/password
- **Other services:** No auth (public or internal-only)

---

## Resource Usage

### Cluster Capacity

- **CPU:** 4 cores (Intel)
- **RAM:** 8GB total
- **Disk:** 256GB SSD (local-path PVCs)

### Current Allocation

| Namespace | Pods | RAM Usage | Storage (PVC) |
|-----------|------|-----------|---------------|
| `matflixlab` | 7 | ~1.5GB | 15Gi (Jellyfin media) |
| `monitoring` | 6 | ~1.2GB | 20Gi (Prometheus 5Gi, Loki 10Gi, Tempo 5Gi) |
| `argocd` | 7 | ~800MB | 0 |
| `kube-system` | 6 | ~600MB | 0 |
| **Total** | **26** | **~4.1GB** | **~35Gi** |

**Available:** ~3.6GB RAM, ~200GB disk

---

## Development Workflow

### Local Testing

```bash
# Build Kustomize locally
kubectl kustomize k8s/ > /tmp/manifests.yaml

# Validate
kubectl apply --dry-run=client -f /tmp/manifests.yaml

# Apply directly (bypasses ArgoCD)
kubectl apply -k k8s/
```

### Image Updates

Most images use `latest` tag. To force pod restart after image push:

```bash
kubectl rollout restart deployment/<name> -n <namespace>
```

### Logs

```bash
# Pod logs
kubectl logs -n matflixlab deployment/speakstats --tail=100 -f

# All pods in namespace
kubectl logs -n matflixlab --all-containers=true --tail=50

# Via Loki (Grafana Explore)
{namespace="matflixlab", app="speakstats"} | json
```

---

## Common Tasks

### Add New Service

1. Create directory: `k8s/apps/<service>/`
2. Add manifests: `namespace.yaml`, `deployment.yaml`, `service.yaml`, `ingress.yaml`
3. Create `kustomization.yaml` listing resources
4. Add to root: `k8s/kustomization.yaml` → `resources: [apps/<service>]`
5. Commit & push → ArgoCD syncs automatically

### Update Existing Service

1. Edit manifest in `k8s/apps/<service>/`
2. Commit & push
3. ArgoCD syncs within 30s
4. Verify in ArgoCD UI or `kubectl get pods`

### Rollback

```bash
# Via git
git revert <commit-hash>
git push

# Manual (bypasses ArgoCD)
kubectl rollout undo deployment/<name> -n <namespace>
```

### Scale Service

```bash
kubectl scale deployment/<name> -n <namespace> --replicas=<N>
```

### Shell into Pod

```bash
kubectl exec -it -n <namespace> deployment/<name> -- /bin/sh
```

---

## Monitoring & Alerts

### Dashboards

All accessible at [grafana.matflixlab.pl/dashboards](https://grafana.matflixlab.pl/dashboards):

- **Host Metrics** — CPU, RAM, disk I/O, network bandwidth
- **Cluster Metrics** — Pod CPU/memory usage vs requests/limits
- **Traefik** — HTTP request rate, response times, errors by route
- **Logs** — Search and filter cluster-wide logs

### Alerts

Currently **no alerting configured**. Prometheus Alertmanager not installed.

**Future:** Add Alertmanager + email/Slack notifications for:
- High CPU/RAM usage
- Pod crash loops
- Disk space <10%
- Certificate expiry

---

## Future Roadmap

### In Progress

- **CI/CD for SpeakStats** (see `speakstats-cicd-plan.md`)
  - Self-hosted GitHub Actions runner
  - Blue/green deployments
  - Automated smoke tests

### Planned

- **CV Email Gate** (see `cv-download-email-confirmation.md`)
  - Cloudflare Workers OTP verification
  - MailChannels email delivery
- **Backup Strategy**
  - Velero for cluster state backup
  - Rclone for PVC backup to cloud storage
- **HA / Multi-node**
  - Add second node for redundancy
  - Distributed storage (Longhorn or Ceph)

---

## Troubleshooting

### Pod won't start

```bash
# Check events
kubectl describe pod <pod-name> -n <namespace>

# Check logs
kubectl logs <pod-name> -n <namespace>

# Check resource limits
kubectl top pod <pod-name> -n <namespace>
```

### Ingress not working

```bash
# Check Traefik logs
kubectl logs -n kube-system deployment/traefik --tail=50

# Verify Ingress exists
kubectl get ingress -n <namespace>

# Test from within cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -H "Host: <domain>" http://<service>.<namespace>
```

### ArgoCD out of sync

```bash
# Check sync status
kubectl get application matflixlab -n argocd -o yaml

# Force sync
kubectl patch application matflixlab -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

### Prometheus not scraping

```bash
# Check Prometheus targets
curl -s http://prometheus.monitoring.svc.cluster.local:9090/targets

# Restart Prometheus
kubectl rollout restart deployment/prometheus -n monitoring
```

---

## Contributing

This is a personal homelab project, but open to suggestions and learning from others.

**Issues/PRs welcome** for:
- Bug fixes
- Documentation improvements
- Architecture suggestions
- Security recommendations

---

## License

This project is open source for educational purposes. No formal license applied.

---

## Contact

- **GitHub:** [@matflixlab](https://github.com/matflixlab)
- **Email:** mjochemski@proton.me
- **Website:** [matflixlab.pl](https://matflixlab.pl)

---

*Last updated: 2026-08-24*
