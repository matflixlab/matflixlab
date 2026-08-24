# SpeakStats — CI/CD Pipeline Plan

## Cel

Pełny pipeline produkcyjny:

```
git push (master/dev)
    │
    ▼
GitHub Actions
    │  build Docker image
    │  push do lokalnego registry przez SSH
    │  aktualizacja tagu w manifeście k8s
    ▼
ArgoCD
    │  wykrywa zmianę w manifeście
    │  sync do klastra
    ▼
Blue/Green Deployment na k3s
    │  master → speakstats.matflixlab.pl      (prod, blue/green)
    │  dev    → speakstats-dev.matflixlab.pl  (staging, rolling)
```

---

## Architektura końcowa

```
GitHub repo: matflixlab/speakstats  (lub folder w matflixlab)
    ├── src/                         ← kod aplikacji Next.js
    ├── Dockerfile
    └── .github/
        └── workflows/
            ├── deploy-prod.yml      ← trigger: push to master
            └── deploy-dev.yml       ← trigger: push to dev

GitHub repo: matflixlab/matflixlab  (manifesty k8s)
    └── k8s/apps/speakstats/
        ├── deployment-blue.yaml     ← image: localhost:30500/speakstats:blue-<SHA>
        ├── deployment-green.yaml    ← image: localhost:30500/speakstats:green-<SHA>
        ├── deployment-dev.yaml      ← image: localhost:30500/speakstats:dev-<SHA>
        ├── service-active.yaml      ← selector: slot=blue (lub green)
        ├── service-preview.yaml     ← selector: slot=green (testowy przed przełączeniem)
        ├── service-dev.yaml         ← selector: slot=dev
        ├── service-whisper.yaml     ← wspólny whisper dla wszystkich slotów
        └── ingress.yaml             ← speakstats.matflixlab.pl + speakstats-dev.matflixlab.pl
```

---

## Etap 1 — Przygotowanie repo SpeakStats

### 1.1 Utwórz repo na GitHub

- Nowe prywatne repo: `matflixlab/speakstats`
- Lub zostaw kod w obecnym miejscu `/home/matflix/matflixlab/speakstats/` i usuń z gitignore

### 1.2 Dwa branche

```bash
git checkout -b dev    # środowisko staging
git push origin dev

# master = produkcja (już istnieje)
```

### 1.3 Semantic versioning obrazów

Zamiast `latest` używamy tagów z SHA commita:

```
localhost:30500/speakstats:blue-abc1234    ← prod, aktywny slot
localhost:30500/speakstats:green-def5678   ← prod, nowy slot
localhost:30500/speakstats:dev-xyz9012     ← staging
```

**Dlaczego SHA?** ArgoCD wykrywa zmianę w manifeście tylko gdy zmieni się wartość. `latest` zawsze wygląda tak samo — ArgoCD nie wie że obraz się zmienił.

---

## Etap 2 — SSH access dla GitHub Actions

GitHub Actions musi zbudować obraz **na serwerze** (bo registry jest lokalne `localhost:30500`).

### 2.1 Wygeneruj dedykowany klucz SSH dla Actions

```bash
# Na matflix-server
ssh-keygen -t ed25519 -C "github-actions-speakstats" -f ~/.ssh/github_actions -N ""
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/github_actions  # ← skopiuj klucz prywatny
```

### 2.2 Dodaj GitHub Secrets do repo speakstats

W GitHub repo → Settings → Secrets and variables → Actions:

| Secret | Wartość |
|---|---|
| `SSH_PRIVATE_KEY` | zawartość `~/.ssh/github_actions` |
| `SSH_HOST` | publiczny IP lub hostname serwera (przez Tailscale lub bezpośrednio) |
| `SSH_USER` | `matflix` |
| `REGISTRY` | `localhost:30500` |
| `ARGOCD_REPO_TOKEN` | GitHub Personal Access Token z dostępem do repo `matflixlab` |

### 2.3 Problem z dostępem SSH do serwera domowego

Twój serwer jest za NAT (Cloudflare Tunnel, brak publicznego IP). Trzy opcje:

**Opcja A — Tailscale (polecam)**
- Zainstaluj Tailscale na serwerze i na GitHub Actions runner
- Actions łączy się przez Tailscale IP (100.x.x.x)
- Nie wymaga otwierania portów

**Opcja B — Self-hosted GitHub Runner**
- Runner działa jako pod w klastrze lub serwis systemd
- Nie potrzebuje SSH — działa lokalnie na serwerze
- Najprostsze rozwiązanie, zero problemu z NAT

**Opcja C — Cloudflare Tunnel dla SSH**
- Wystawienie portu SSH przez Cloudflare Access
- Bardziej złożone, wymaga konfiguracji CF Access

**Rekomendacja: Self-hosted Runner** — runner lokalny eliminuje problem NAT całkowicie.

---

## Etap 3 — Self-hosted GitHub Actions Runner

### 3.1 Instalacja runnera

```bash
# Na matflix-server
mkdir -p /home/matflix/actions-runner && cd /home/matflix/actions-runner
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Konfiguracja (token z GitHub repo → Settings → Actions → Runners → New)
./config.sh --url https://github.com/matflixlab/speakstats --token <TOKEN>
```

### 3.2 Runner jako serwis systemd

```bash
sudo ./svc.sh install
sudo ./svc.sh start
```

Runner działa w tle, restartuje się automatycznie po rebocie.

### 3.3 Dodaj label do runnera

Podczas konfiguracji dodaj label: `self-hosted, matflix-server`

W workflow używasz:
```yaml
runs-on: [self-hosted, matflix-server]
```

---

## Etap 4 — Blue/Green manifesty k8s

### 4.1 deployment-blue.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: speakstats-blue
  namespace: matflixlab
  labels:
    app: speakstats
    slot: blue
spec:
  replicas: 1          # 0 gdy green jest aktywny
  selector:
    matchLabels:
      app: speakstats
      slot: blue
  template:
    metadata:
      labels:
        app: speakstats
        slot: blue
    spec:
      containers:
        - name: speakstats
          image: localhost:30500/speakstats:blue-PLACEHOLDER
          # ↑ GitHub Actions podmienia PLACEHOLDER na SHA commita
```

### 4.2 deployment-green.yaml

Identyczny jak blue, label `slot: green`, `replicas: 0` domyślnie.

### 4.3 service-active.yaml (produkcja)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: speakstats
  namespace: matflixlab
  annotations:
    active-slot: "blue"    # ← tylko informacyjne, zmieniane przy przełączaniu
spec:
  selector:
    app: speakstats
    slot: blue             # ← TO się zmienia przy przełączaniu blue↔green
  ports:
    - port: 3000
```

### 4.4 service-preview.yaml (testowy slot przed przełączeniem)

```yaml
spec:
  selector:
    app: speakstats
    slot: green            # zawsze wskazuje na nieaktywny slot
```

Dostępny pod `speakstats-preview.matflixlab.pl`.

### 4.5 ingress.yaml

```yaml
rules:
  - host: speakstats.matflixlab.pl        # → service speakstats (active)
  - host: speakstats-preview.matflixlab.pl # → service speakstats-preview
  - host: speakstats-dev.matflixlab.pl    # → service speakstats-dev
```

---

## Etap 5 — GitHub Actions Workflow

### 5.1 deploy-prod.yml (push to master)

```yaml
name: Deploy Production (Blue/Green)

on:
  push:
    branches: [master]

jobs:
  deploy:
    runs-on: [self-hosted, matflix-server]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set image tag
        run: echo "IMAGE_TAG=blue-${{ github.sha }}" >> $GITHUB_ENV
        # Uwaga: który slot jest aktywny — skrypt sprawdza i wybiera przeciwny

      - name: Determine target slot
        run: |
          # Sprawdź który slot jest aktywny przez kubectl
          ACTIVE=$(sudo kubectl get service speakstats -n matflixlab \
            -o jsonpath='{.spec.selector.slot}')
          if [ "$ACTIVE" = "blue" ]; then
            echo "TARGET_SLOT=green" >> $GITHUB_ENV
            echo "IMAGE_TAG=green-${{ github.sha }}" >> $GITHUB_ENV
          else
            echo "TARGET_SLOT=blue" >> $GITHUB_ENV
            echo "IMAGE_TAG=blue-${{ github.sha }}" >> $GITHUB_ENV
          fi

      - name: Build Docker image
        run: |
          docker build -t localhost:30500/speakstats:${{ env.IMAGE_TAG }} .

      - name: Push to registry
        run: |
          docker push localhost:30500/speakstats:${{ env.IMAGE_TAG }}

      - name: Update manifest
        run: |
          cd /home/matflix/matflixlab
          # Podmień tag obrazu w manifeście docelowego slotu
          sed -i "s|localhost:30500/speakstats:${{ env.TARGET_SLOT }}-.*|localhost:30500/speakstats:${{ env.IMAGE_TAG }}|" \
            k8s/apps/speakstats/deployment-${{ env.TARGET_SLOT }}.yaml
          # Włącz repliki dla docelowego slotu
          # (skrypt lub kustomize overlay)

      - name: Push manifest update
        run: |
          cd /home/matflix/matflixlab
          git config user.email "actions@matflixlab"
          git config user.name "GitHub Actions"
          git add k8s/apps/speakstats/
          git commit -m "deploy: speakstats ${{ env.TARGET_SLOT }} ${{ github.sha }}"
          git push

      - name: Wait for ArgoCD sync
        run: |
          sleep 30  # ArgoCD auto-sync
          sudo kubectl rollout status deployment/speakstats-${{ env.TARGET_SLOT }} \
            -n matflixlab --timeout=120s

      - name: Smoke test on preview
        run: |
          curl -sf https://speakstats-preview.matflixlab.pl/api/health
          # Jeśli health check przejdzie — przełącz Service

      - name: Switch active slot
        run: |
          sudo kubectl patch service speakstats -n matflixlab \
            -p '{"spec":{"selector":{"slot":"${{ env.TARGET_SLOT }}"}}}'
          sudo kubectl annotate service speakstats -n matflixlab \
            active-slot="${{ env.TARGET_SLOT }}" --overwrite

      - name: Scale down old slot
        run: |
          OLD_SLOT=$( [ "${{ env.TARGET_SLOT }}" = "blue" ] && echo "green" || echo "blue" )
          sudo kubectl scale deployment speakstats-$OLD_SLOT \
            -n matflixlab --replicas=0
```

### 5.2 deploy-dev.yml (push to dev)

```yaml
name: Deploy Dev (Staging)

on:
  push:
    branches: [dev]

jobs:
  deploy:
    runs-on: [self-hosted, matflix-server]
    steps:
      - uses: actions/checkout@v4

      - name: Build and push
        run: |
          TAG="dev-${{ github.sha }}"
          docker build -t localhost:30500/speakstats:$TAG .
          docker push localhost:30500/speakstats:$TAG

      - name: Update manifest and push
        run: |
          cd /home/matflix/matflixlab
          sed -i "s|localhost:30500/speakstats:dev-.*|localhost:30500/speakstats:$TAG|" \
            k8s/apps/speakstats/deployment-dev.yaml
          git add k8s/apps/speakstats/deployment-dev.yaml
          git commit -m "deploy: speakstats dev ${{ github.sha }}"
          git push

      - name: Wait for rollout
        run: |
          sleep 30
          sudo kubectl rollout status deployment/speakstats-dev \
            -n matflixlab --timeout=120s
```

---

## Etap 6 — ArgoCD Application update

Obecna Application synchronizuje `k8s/` — nie trzeba nic zmieniać. ArgoCD automatycznie podchwyci nowe manifesty po push.

Opcjonalnie — wyłączyć `automated` dla prod i zostawić ręczny sync jako dodatkowy checkpoint bezpieczeństwa:

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: true
  # Prod może wymagać ręcznego sync w ArgoCD UI przed przełączeniem
```

---

## Etap 7 — Cloudflare Tunnel

Dodać dwie nowe subdomeny (tak jak poprzednie):
- `speakstats-preview.matflixlab.pl` → `http://localhost:80`
- `speakstats-dev.matflixlab.pl` → `http://localhost:80`

---

## Podsumowanie kroków implementacji

| Krok | Opis | Trudność | Czas |
|---|---|---|---|
| 1 | Repo SpeakStats na GitHub, dwa branche | ⭐ | 15 min |
| 2 | Self-hosted runner jako systemd | ⭐⭐ | 30 min |
| 3 | Blue/green manifesty k8s | ⭐⭐ | 45 min |
| 4 | GitHub Actions workflows | ⭐⭐⭐ | 60 min |
| 5 | Cloudflare subdomeny | ⭐ | 10 min |
| 6 | Testy end-to-end | ⭐⭐ | 30 min |
| **Razem** | | | **~3h** |

---

## Co to daje na CV

```
Implemented full CI/CD pipeline for SpeakStats:
- Self-hosted GitHub Actions runner on k3s homelab
- Automated Docker build and push to local registry on every commit
- Blue/green deployment strategy with zero downtime
- ArgoCD GitOps sync from GitHub to k3s cluster
- Separate staging environment (dev branch → speakstats-dev.matflixlab.pl)
- Automated smoke tests and slot switching
```

---

## Status implementacji

### ✅ Wykonane

| Krok | Status | Szczegóły |
|---|---|---|
| Repo `matflixlab/speakstats` na GitHub | ✅ | Public, opis dodany |
| Branch `master` i `dev` | ✅ | master = prod, dev = staging |
| Deploy key SSH | ✅ | `~/.ssh/id_ed25519_speakstats`, dodany do repo |
| SSH config | ✅ | `~/.ssh/config` — alias `github-speakstats` |
| `/api/health` endpoint | ✅ | `src/app/api/health/route.ts`, scommitowany na `dev` |

### ⏳ Do zrobienia w następnej sesji

| Krok | Etap planu |
|---|---|
| Self-hosted GitHub Actions runner jako systemd | Etap 3 |
| Blue/green manifesty k8s | Etap 4 |
| GitHub Actions workflows (deploy-prod.yml, deploy-dev.yml) | Etap 5 |
| Cloudflare subdomeny: speakstats-preview, speakstats-dev | Etap 7 |
| Testy end-to-end | Etap 6 |

### Stan repo

```
github.com/matflixlab/speakstats

master  f1d1f22  version1.2 recording on web is working
dev     0660688  feat: add /api/health endpoint for CI/CD smoke tests
```

### Lokalizacja plików

```
/home/matflix/matflixlab/speakstats/   ← kod aplikacji
/home/matflix/.ssh/id_ed25519_speakstats     ← klucz prywatny SSH
/home/matflix/.ssh/id_ed25519_speakstats.pub ← klucz publiczny SSH
/home/matflix/.ssh/config                    ← aliasy SSH
/home/matflix/matflixlab/speakstats-cicd-plan.md ← ten plik
```

1. **Runner** → **self-hosted jako systemd** (~20MB w spoczynku, build lokalnie, nie obciąża k8s)
2. **Repo** → **osobne repo `matflixlab/speakstats`** — lepiej dla portfolio, czysta historia commitów
3. **Switch** → **automatyczny** po pomyślnym smoke teście
4. **Health check** → **BRAK w obecnym kodzie** — wymagana zmiana przed implementacją pipeline

### Wymagana zmiana w kodzie SpeakStats (krok 0)

Dodać `src/app/api/health/route.ts`:

```typescript
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json({ status: 'ok', timestamp: new Date().toISOString() })
}
```

Smoke test w Actions sprawdza:
```bash
curl -sf https://speakstats-preview.matflixlab.pl/api/health
# {"status":"ok",...} = OK → przełącz slot
```
