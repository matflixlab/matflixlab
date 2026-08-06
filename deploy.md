# Deploy — matflixlab

## Środowisko

- Homelab: single-node k3s na laptopie (`matflix-server`)
- Workspace: `/home/matflix/matflixlab/`
- Manifesty k8s: `/home/matflix/matflixlab/k8s/`
- Namespace aplikacji: `matflixlab`
- Ekspozycja: Cloudflare Tunnel → Traefik (k3s built-in) → pody

## Serwisy

| Serwis | URL | Typ |
|---|---|---|
| landing | matflixlab.pl | nginx, HTML z ConfigMap |
| speakstats | speakstats.matflixlab.pl | Next.js 15 |
| jellyfin | jellyfin.matflixlab.pl | media server |
| umami | umami.matflixlab.pl | analytics |

## Landing page — dev & deploy

### Dev (localhost preview)

```bash
cd /home/matflix/matflixlab/k8s/apps/landing/html
python3 -m http.server 9998
# otwórz http://localhost:9998
```

### Deploy na produkcję

Landing page serwowany jest przez nginx z **ConfigMap** (`landing-html`) generowanego z pliku `html/index.html` przez Kustomize.

**WAŻNE:** `kubectl apply -k` nie zawsze aktualizuje ConfigMap poprawnie (problem z last-applied-configuration). Używaj zawsze poniższego polecenia:

```bash
cd /home/matflix/matflixlab

# 1. zaktualizuj ConfigMap
sudo kubectl create configmap landing-html \
  --from-file=index.html=./k8s/apps/landing/html/index.html \
  -n matflixlab --dry-run=client -o yaml | sudo kubectl apply -f -

# 2. restart poda
sudo kubectl rollout restart deployment/landing -n matflixlab
sudo kubectl rollout status deployment/landing -n matflixlab
```

### Weryfikacja

```bash
# sprawdź czy pod ma nową wersję pliku
sudo kubectl exec -n matflixlab deployment/landing -- cat /usr/share/nginx/html/index.html | grep -c "data-i18n"

# sprawdź czy Traefik serwuje poprawnie
curl -s -H "Host: matflixlab.pl" http://localhost:80/ | head -5
```

## Struktura k8s

```
k8s/
├── kustomization.yaml          # root kustomization
├── apps/
│   ├── landing/
│   │   ├── kustomization.yaml  # generuje ConfigMap z html/index.html
│   │   ├── deployment.yaml     # nginx:alpine, montuje ConfigMap + hostPath CV
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── html/
│   │       └── index.html      # JEDYNY plik do edycji dla landing page
│   ├── speakstats/
│   ├── jellyfin/
│   └── umami/
└── infrastructure/
    ├── namespaces/
    ├── registry/               # lokalny registry NodePort :30500
    └── traefik/
```

## CV

Plik PDF serwowany z `hostPath`:
- ścieżka na hoście: `/home/matflix/matflixlab/cv/`
- montowany jako: `/usr/share/nginx/html/cv/` w podzie
- URL: `matflixlab.pl/cv/3ys_DevOps_mjochemski.pdf`

## kubectl — wymaga sudo

k3s kubeconfig jest w `/etc/rancher/k3s/k3s.yaml` z ograniczonymi uprawnieniami. Wszystkie `kubectl` wymagają `sudo`.

```bash
# alternatywnie — skopiuj kubeconfig jednorazowo:
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown matflix:matflix ~/.kube/config
# potem kubectl działa bez sudo
```

## Git workflow

Repo zainicjowane lokalnie w `/home/matflix/matflixlab/`. Brak remote — tylko lokalne commity.

### Co jest w repo
- `k8s/` — wszystkie manifesty (bez `secret.yaml` — zawiera hasła)
- `deploy.md` — dokumentacja
- `.kiro/steering/` — steering file dla Kiro

### Co jest w .gitignore
- `registry-data/` — dane lokalnego Docker registry
- `media/` — pliki wideo
- `cv/` — pliki PDF
- `speakstats/` — osobne repo
- `k8s/**/secret.yaml` — sekrety k8s z hasłami w plaintexcie
- `*.pdf`, `*.jpeg`, `*.jpg`, `*.png`

### Commit po zmianie landing page

```bash
cd /home/matflix/matflixlab
git add k8s/apps/landing/html/index.html
git commit -m "landing: opis zmiany"
```

### Rollback do poprzedniej wersji

```bash
# znajdź hash commita
git log --oneline k8s/apps/landing/html/index.html

# przywróć plik
git checkout <hash> -- k8s/apps/landing/html/index.html

# wdróż na prod
sudo kubectl create configmap landing-html \
  --from-file=index.html=./k8s/apps/landing/html/index.html \
  -n matflixlab --dry-run=client -o yaml | sudo kubectl apply -f -
sudo kubectl rollout restart deployment/landing -n matflixlab
```

`index.html` to single-file aplikacja zawierająca:
- **CSS** z CSS variables dla dark/light theme (`[data-theme="light"]`)
- **Mermaid** diagramy (architektura sieci, serwisów)
- **i18n** — słownik `TRANSLATIONS` z kluczami EN/PL, atrybuty `data-i18n` na elementach
- **Theme toggle** — `localStorage` klucz `mllab-theme`, respektuje `prefers-color-scheme`
- **Lang toggle** — `localStorage` klucz `mllab-lang`, domyślnie EN
- **Umami stats** — live stats z API (visitors, pageviews, visits, countries)
- **Formcarry contact form** — endpoint `https://formcarry.com/s/D8zQLX7XeAC`, limit 3 wiadomości per przeglądarka (`localStorage` klucz `mllab-submits`)
- **CV download tracking** — `umami.track('cv-download')` przy kliknięciu
