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
| grafana | grafana.matflixlab.pl | monitoring |
| argocd | argocd.matflixlab.pl | argocd |

---

## GitOps flow (ArgoCD) — podstawowy workflow

Od wdrożenia ArgoCD **deploy = git push**. ArgoCD synchronizuje klaster z repozytorium automatycznie co 3 minuty.

```
edytujesz manifest / index.html
    │
    git add + git commit + git push
    │
    ArgoCD wykrywa zmianę (max 3 min)
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
# ArgoCD synchronizuje automatycznie w ciągu ~3 minut
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

Landing page jest serwowany przez nginx z **ConfigMap** generowanego z `html/index.html` przez Kustomize. ArgoCD zarządza ConfigMap, ale **po zmianie ConfigMap trzeba zrestartować pod** żeby nginx załadował nową wersję.

**WAŻNE:** `git push` + ArgoCD sync aktualizuje ConfigMap ale nie restartuje poda automatycznie. Trzeba to zrobić ręcznie.

### Deploy landing page

```bash
cd /home/matflix/matflixlab

# 1. edytuj plik
# k8s/apps/landing/html/index.html

# 2. commit i push
git add k8s/apps/landing/html/index.html
git commit -m "landing: opis zmiany"
git push

# 3. poczekaj na sync ArgoCD (~3 min) lub wymuś
sudo kubectl patch application matflixlab -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'

# 4. restart poda (wymagany żeby nginx załadował nowy ConfigMap)
sudo kubectl rollout restart deployment/landing -n matflixlab
sudo kubectl rollout status deployment/landing -n matflixlab
```

### Dev preview na localhost

```bash
cd /home/matflix/matflixlab/k8s/apps/landing/html
python3 -m http.server 9998
# otwórz http://localhost:9998
# plik dev: http://localhost:9998/index.dev.html
```

---

## SpeakStats — deploy nowej wersji aplikacji

SpeakStats używa lokalnego Docker registry (`localhost:30500`). Kod w `/home/matflix/matflixlab/speakstats/` (osobne repo, w gitignore).

```bash
cd /home/matflix/matflixlab/speakstats

# 1. build obrazu
docker build -t localhost:30500/speakstats:latest .

# 2. push do lokalnego registry
docker push localhost:30500/speakstats:latest

# 3. restart poda (pobierze nowy obraz)
sudo kubectl rollout restart deployment/speakstats -n matflixlab
sudo kubectl rollout status deployment/speakstats -n matflixlab
```

**Model whisper:** `ggml-base.bin` (multilingual, 142MB) — `/home/matflix/matflixlab/speakstats/models/`

---

## Struktura repo

```
k8s/
├── kustomization.yaml              # root — lista wszystkich aplikacji
├── apps/
│   ├── landing/
│   │   ├── kustomization.yaml      # generuje ConfigMap z html/index.html
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── html/
│   │       └── index.html          # ← EDYTUJ TEN PLIK dla landing page
│   ├── speakstats/
│   │   ├── kustomization.yaml
│   │   ├── deployment-app.yaml
│   │   ├── deployment-whisper.yaml ← model whisper config
│   │   ├── services.yaml
│   │   ├── ingress.yaml
│   │   └── pvc.yaml
│   ├── jellyfin/
│   ├── umami/
│   ├── monitoring/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── prometheus/
│   │   ├── grafana/                ← grafana-secret.yaml jest w gitignore!
│   │   ├── node-exporter/
│   │   └── kube-state-metrics/
│   └── argocd/
│       ├── kustomization.yaml
│       ├── ingress.yaml
│       └── application.yaml        # ← definicja ArgoCD Application
└── infrastructure/
    ├── namespaces/
    ├── registry/
    └── traefik/
```

---

## Secrets — ważne uwagi

Pliki z sekretami są w `.gitignore` i **nie trafiają do repo**:

| Secret | Lokalizacja | Jak zaaplikować |
|---|---|---|
| `grafana-secret` | `k8s/apps/monitoring/grafana/secret.yaml` | `sudo kubectl apply -f <plik>` |
| `umami-secret` | `k8s/apps/umami/secret.yaml` | `sudo kubectl apply -f <plik>` |
| ArgoCD repo key | tworzony przez kubectl, nie plik | już istnieje w klastrze |

Po świeżej instalacji klastra sekrety trzeba zaaplikować ręcznie przed uruchomieniem ArgoCD sync.

---

## ArgoCD — konfiguracja

- **URL:** https://argocd.matflixlab.pl
- **User:** admin
- **Repo:** `git@github.com:matflixlab/matflixlab.git` (SSH, deploy key)
- **SSH key:** `~/.ssh/id_ed25519` (klucz na serwerze)
- **Sync policy:** auto-sync, selfHeal=true, prune=false

### Dodawanie nowej subdmomeny Cloudflare

Cloudflare Zero Trust → Networks → Tunnels → tunel → Edit → Public Hostname → Add:
- Subdomain: `<nazwa>`
- Domain: `matflixlab.pl`
- Service: `HTTP` / `localhost:80`

---

## Diagnostyka

```bash
# status wszystkich podów
sudo kubectl get pods -n matflixlab
sudo kubectl get pods -n monitoring
sudo kubectl get pods -n argocd

# logi konkretnego poda
sudo kubectl logs deployment/<nazwa> -n <namespace> --tail=30

# ArgoCD status
sudo kubectl get application matflixlab -n argocd

# zasoby klastra
sudo kubectl top nodes
sudo kubectl top pods -n matflixlab

# sprawdź czy Traefik routuje poprawnie
curl -s -H "Host: matflixlab.pl" http://localhost:80/ | head -3
```

---

## Landing page — architektura kodu

`index.html` to single-file aplikacja zawierająca:

- **CSS** z CSS variables dla dark/light theme (`[data-theme="light"]`)
- **Sticky navbar** z hamburger menu na mobile
- **Mermaid** diagramy (architektura sieci)
- **i18n** — słownik `TRANSLATIONS` z kluczami EN/PL, atrybuty `data-i18n` na elementach
- **Theme toggle** — `localStorage` klucz `mllab-theme`, respektuje `prefers-color-scheme`
- **Lang toggle** — `localStorage` klucz `mllab-lang`, domyślnie EN
- **Terminal widget** — animacja typing z `mkdir` i `cd && ls`
- **Grafana embeds** — 4 panele z Node Exporter Full w sekcji homelab-stack
- **Umami stats** — live stats z API (visitors, pageviews, visits, countries)
- **Formcarry contact form** — endpoint `https://formcarry.com/s/D8zQLX7XeAC`, limit 3/przeglądarka
- **CV download tracking** — `umami.track('cv-download')` przy kliknięciu
- **Fade-in animacje** — Intersection Observer na `.fade-in` elementach
