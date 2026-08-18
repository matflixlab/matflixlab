# matflixlab — deploy & operations guide

## Środowisko

- **Host:** laptop Ubuntu (`matflix-server`)
- **Klaster:** single-node k3s
- **Workspace:** `/home/matflix/matflixlab/`
- **Namespace aplikacji:** `matflixlab`
- **Namespace monitoringu:** `monitoring`
- **Namespace ArgoCD:** `argocd`
- **kubectl:** wymaga `sudo` (kubeconfig w `/etc/rancher/k3s/k3s.yaml`)
- **Ekspozycja:** Cloudflare Tunnel → Traefik (k3s built-in) → pody

## Serwisy

| Serwis | URL | Namespace |
|---|---|---|
| landing | matflixlab.pl | matflixlab |
| speakstats | speakstats.matflixlab.pl | matflixlab |
| matflix (jellyfin) | jellyfin.matflixlab.pl | matflixlab |
| umami | umami.matflixlab.pl | matflixlab |
| umami-proxy | umami-proxy.matflixlab.pl | matflixlab |
| grafana | grafana.matflixlab.pl | monitoring |
| argocd | argocd.matflixlab.pl | argocd |

---

## GitOps flow (ArgoCD) — podstawowy workflow

**Deploy = git push.** ArgoCD synchronizuje klaster z repozytorium automatycznie co 3 minuty.

```
edytujesz manifest / index.html
    │
    git add + git commit + git push
    │
    ArgoCD wykrywa zmianę (~3 min)
    │
    kubectl apply automatycznie
    │
    klaster zsynchronizowany
```

### Remote repo

```
git@github.com:matflixlab/matflixlab.git
```

### Standardowy deploy (manifesty k8s)

```bash
cd /home/matflix/matflixlab
git add <zmieniony plik>
git commit -m "opis zmiany"
git push
# ArgoCD synchronizuje automatycznie
```

### Wymuszenie natychmiastowej synchronizacji

```bash
sudo kubectl patch application matflixlab -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

### Sprawdzenie statusu synchronizacji

```bash
sudo kubectl get application matflixlab -n argocd
# oczekiwany wynik: Synced / Healthy
```

---

## Landing page — szczególny przypadek

Landing page jest serwowany przez nginx z **ConfigMap** generowanego z `html/index.html`.
ArgoCD aktualizuje ConfigMap ale **pod trzeba zrestartować ręcznie** żeby nginx załadował nową wersję.

### Deploy landing page

```bash
cd /home/matflix/matflixlab

# 1. edytuj plik DEV
# k8s/apps/landing/html/index.dev.html

# 2. skopiuj na prod i commituj
cp k8s/apps/landing/html/index.dev.html k8s/apps/landing/html/index.html
git add k8s/apps/landing/html/index.html
git commit -m "landing: opis zmiany"
git push

# 3. wymuś sync ArgoCD
sudo kubectl patch application matflixlab -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'

# 4. restart poda (wymagany)
sudo kubectl rollout restart deployment/landing -n matflixlab
sudo kubectl rollout status deployment/landing -n matflixlab
```

### Dev preview na localhost

```bash
cd /home/matflix/matflixlab/k8s/apps/landing/html
python3 -m http.server 9998
# otwórz http://localhost:9998/index.dev.html
```

**Uwaga:** statystyki Umami (SITE STATS) nie wyświetlają się na dev — to oczekiwane zachowanie.
CORS proxy (`umami-proxy`) akceptuje tylko origin `https://matflixlab.pl`.

---

## Umami Stats Proxy

Hasło Umami jest **wyłącznie w Kubernetes Secret** (`umami-proxy-secret`), nie w HTML.

Landing page odpytuje `https://umami-proxy.matflixlab.pl/stats` — zero credentials po stronie klienta.

### Pliki

```
k8s/apps/umami-proxy/
├── configmap.yaml      ← kod Python Flask (~50 linii)
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── kustomization.yaml
└── secret.yaml         ← w gitignore! zawiera hasło Umami
```

### Po świeżej instalacji klastra

```bash
sudo kubectl apply -f k8s/apps/umami-proxy/secret.yaml
```

---

## SpeakStats — deploy nowej wersji aplikacji

SpeakStats używa lokalnego Docker registry (`localhost:30500`).
Kod w `/home/matflix/matflixlab/speakstats/` (osobne repo, w gitignore).

```bash
cd /home/matflix/matflixlab/speakstats

# 1. build
docker build -t localhost:30500/speakstats:latest .

# 2. push
docker push localhost:30500/speakstats:latest

# 3. restart
sudo kubectl rollout restart deployment/speakstats -n matflixlab
sudo kubectl rollout status deployment/speakstats -n matflixlab
```

**Model whisper:** `ggml-base.bin` (multilingual, 142MB)
Lokalizacja: `/home/matflix/matflixlab/speakstats/models/`

---

## Struktura repo

```
k8s/
├── kustomization.yaml              # root — lista wszystkich aplikacji
├── apps/
│   ├── landing/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── html/
│   │       ├── index.html          ← PROD (kopiuj z index.dev.html)
│   │       └── index.dev.html      ← DEV — edytuj ten plik
│   ├── speakstats/
│   │   ├── deployment-app.yaml
│   │   ├── deployment-whisper.yaml ← model whisper config
│   │   ├── services.yaml
│   │   ├── ingress.yaml
│   │   └── pvc.yaml
│   ├── umami-proxy/
│   │   ├── configmap.yaml          ← kod proxy Python
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── secret.yaml             ← gitignore! hasło Umami
│   ├── jellyfin/
│   ├── umami/
│   ├── monitoring/
│   │   ├── prometheus/
│   │   ├── grafana/                ← grafana/secret.yaml w gitignore
│   │   ├── node-exporter/
│   │   └── kube-state-metrics/
│   └── argocd/
│       ├── ingress.yaml
│       └── application.yaml        ← definicja ArgoCD Application
└── infrastructure/
    ├── namespaces/
    ├── registry/
    └── traefik/
```

---

## Secrets — ważne uwagi

Pliki z sekretami są w `.gitignore` — **nie trafiają do repo**.

| Secret | Lokalizacja | Jak zaaplikować |
|---|---|---|
| `umami-proxy-secret` | `k8s/apps/umami-proxy/secret.yaml` | `sudo kubectl apply -f <plik>` |
| `grafana-secret` | `k8s/apps/monitoring/grafana/secret.yaml` | `sudo kubectl apply -f <plik>` |
| `umami-secret` | `k8s/apps/umami/secret.yaml` | `sudo kubectl apply -f <plik>` |
| ArgoCD repo SSH key | tworzony przez kubectl | już istnieje w klastrze |

**Po świeżej instalacji klastra** zaaplikuj sekrety ręcznie przed uruchomieniem ArgoCD sync.

---

## ArgoCD

- **URL:** https://argocd.matflixlab.pl
- **User:** admin
- **Anonymous access:** włączony (read-only bez logowania)
- **Repo:** `git@github.com:matflixlab/matflixlab.git` (SSH, deploy key)
- **SSH key:** `~/.ssh/id_ed25519`
- **Sync policy:** auto-sync, selfHeal=true, prune=false

### Dodawanie nowej subdomeny Cloudflare

Cloudflare Zero Trust → Networks → Tunnels → tunel → Edit → Public Hostname → Add:
- Subdomain: `<nazwa>`
- Domain: `matflixlab.pl`
- Service Type: `HTTP`
- URL: `localhost:80`

---

## Monitoring

- **Prometheus:** scrape co 30s, retencja 30 dni, PVC 5Gi
- **Grafana:** https://grafana.matflixlab.pl — anonymous read-only
- **Loki:** agregator logów ze wszystkich namespace'ów, retencja 7 dni, PVC 10Gi
- **Promtail:** DaemonSet zbierający logi ze wszystkich podów → Loki
- **Dashboardy:** Node Exporter Full (`rYdddlPWk`), Kubernetes Views Pods (`k8s_views_pods`), matflixlab Cluster (`matflixlab-cluster`), Loki Kubernetes Logs (`o6-BGgnnk`)
- **node-exporter:** metryki hosta CPU/RAM/disk/network
- **kube-state-metrics:** metryki podów/deploymentów

### Przykładowe LogQL queries w Grafana Explore → Loki

```logql
# wszystkie logi z namespace matflixlab
{namespace="matflixlab"}

# tylko błędy speakstats
{namespace="matflixlab", app="speakstats"} |= "error"

# logi wszystkich namespace'ów z poziomem error
{namespace=~".+"} |= "error" | logfmt | level="error"

# logi ArgoCD sync
{namespace="argocd"} |= "sync"
```

---

## Diagnostyka

```bash
# status wszystkich podów
sudo kubectl get pods -n matflixlab
sudo kubectl get pods -n monitoring
sudo kubectl get pods -n argocd

# logi
sudo kubectl logs deployment/<nazwa> -n <namespace> --tail=30

# ArgoCD status
sudo kubectl get application matflixlab -n argocd

# zasoby klastra
sudo kubectl top nodes
sudo kubectl top pods -n matflixlab

# weryfikacja Traefik routing
curl -s -H "Host: matflixlab.pl" http://localhost:80/ | head -3
```

---

## Landing page — architektura kodu

`index.html` (kopiowany z `index.dev.html`) to single-file aplikacja:

- **CSS** z CSS variables dla dark/light theme (`[data-theme="light"]`)
- **Sticky navbar** z hamburger menu na mobile, linki scroll do sekcji
- **Mermaid** diagramy (architektura sieci) w sekcji homelab-stack
- **Terminal widget** — animacja typing: `mkdir` → `cd && ls`, osobne dla desktop/mobile
- **Grafana embeds** — 4 panele Node Exporter + custom cluster dashboard w sekcji homelab-stack
- **i18n** — słownik `TRANSLATIONS` EN/PL, atrybuty `data-i18n`, `localStorage` klucz `mllab-lang`
- **Theme toggle** — `localStorage` klucz `mllab-theme`, respektuje `prefers-color-scheme`
- **Umami stats** — fetch do `https://umami-proxy.matflixlab.pl/stats` (bez credentials!)
- **Formcarry contact form** — `https://formcarry.com/s/D8zQLX7XeAC`, limit 3/przeglądarka (`mllab-submits`)
- **CV download tracking** — `umami.track('cv-download')`
- **Fade-in animacje** — Intersection Observer na `.fade-in`
