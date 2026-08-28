# Tempo — plan wdrożenia

## Cel

Dodanie distributed tracing do stosu observability:

```
Prometheus  → metryki (RED: Rate, Errors, Duration)
Loki        → logi
Tempo       → trace'y HTTP requestów przez Traefik
```

Rezultat: pełny **LGTM stack** (Loki, Grafana, Tempo, Mimir/Prometheus).

---

## Architektura

```
Request → Cloudflare → Traefik
                           │
                           │  OTLP gRPC (port 4317)
                           ▼
                       Tempo pod
                       namespace: monitoring
                       PVC: 5Gi, retencja: 24h
                           │
                           │  datasource
                           ▼
                       Grafana
                       Explore → Tempo
                       korelacja trace ↔ logi (Loki) ↔ metryki (Prometheus)
```

---

## Etap 1 — Tempo deployment

### Pliki do stworzenia

```
k8s/apps/monitoring/tempo/
├── configmap.yaml      ← konfiguracja Tempo (storage, retencja, OTLP receiver)
├── pvc.yaml            ← 5Gi local-path
├── deployment.yaml     ← grafana/tempo:2.4.0, ~150MB RAM
└── service.yaml        ← ClusterIP: 4317 (OTLP gRPC), 3200 (HTTP API dla Grafany)
```

**Brak Ingress** — Tempo nie jest publiczny. Tylko Traefik i Grafana mają do niego dostęp.

### Konfiguracja Tempo (configmap)

```yaml
stream_over_http_enabled: true
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal

compactor:
  compaction:
    block_retention: 24h    # 24h retencja — wystarczy do analizy

query_frontend:
  search:
    duration_slo: 5s
    throughput_bytes_slo: 1.073741824e+09
```

---

## Etap 2 — Grafana datasource

Dodać Tempo do istniejącego `configmap-datasource.yaml`:

```yaml
- name: Tempo
  type: tempo
  url: http://tempo.monitoring.svc.cluster.local:3200
  access: proxy
  isDefault: false
  jsonData:
    tracesToLogsV2:
      datasourceUid: loki-uid        # korelacja trace → logi w Loki
      filterByTraceID: true
      filterBySpanID: false
    serviceMap:
      datasourceUid: prometheus-uid  # korelacja trace → metryki
    nodeGraph:
      enabled: true
    lokiSearch:
      datasourceUid: loki-uid
```

**Korelacja trace ↔ logi:** po kliknięciu w trace w Tempo, Grafana automatycznie otworzy powiązane logi w Loki. Działa jeśli Traefik dodaje `traceID` do logów — konfigurujemy to w kroku 3.

---

## Etap 3 — Traefik konfiguracja

Aktualizacja istniejącego `k8s/infrastructure/traefik/traefik-config.yaml`:

```yaml
spec:
  valuesContent: |-
    ports:
      web:
        port: 8000
        exposedPort: 80
        hostPort: 80
      websecure:
        port: 8443
        exposedPort: 443
        hostPort: 443
    service:
      type: ClusterIP
    hostNetwork: false

    # NOWE — tracing
    tracing:
      otlp:
        grpc:
          endpoint: tempo.monitoring.svc.cluster.local:4317
          insecure: true

    # Dodanie traceID do access logów → korelacja z Loki
    logs:
      access:
        enabled: true
        fields:
          headers:
            defaultMode: keep
```

**Ważne:** Traefik 3.x wymaga restartu poda po zmianie HelmChartConfig. k3s automatycznie to obsługuje.

---

## Etap 4 — Kustomization

Dodać do `k8s/apps/monitoring/kustomization.yaml`:

```yaml
- tempo/pvc.yaml
- tempo/configmap.yaml
- tempo/deployment.yaml
- tempo/service.yaml
```

---

## Etap 5 — Deploy i weryfikacja

```bash
git add k8s/apps/monitoring/tempo/ \
        k8s/apps/monitoring/grafana/configmap-datasource.yaml \
        k8s/apps/monitoring/kustomization.yaml \
        k8s/infrastructure/traefik/traefik-config.yaml
git commit -m "feat: add Tempo distributed tracing + Traefik OTLP"
git push
# ArgoCD sync automatyczny

# Weryfikacja po ~2 min
sudo kubectl get pods -n monitoring | grep tempo
# oczekiwane: tempo Running

# Test czy Tempo odbiera trace'y
sudo kubectl exec -n monitoring deployment/tempo -- \
  wget -qO- http://localhost:3200/ready
# oczekiwane: ready

# Wygeneruj ruch przez Traefik
curl https://matflixlab.pl

# W Grafana → Explore → Tempo → Search
# Powinny pojawić się trace'y z Traefika
```

---

## Etap 6 — Grafana: import dashboardu

Import gotowego dashboardu Traefik + Tempo z Grafana.com:
- **ID 17501** — Traefik Official (z trace'ami)

Alternatywnie wbudowany w Grafanie: **Explore → Tempo → Search** działa bez żadnego dashboardu.

---

## Zasoby

| Komponent | RAM | Storage |
|---|---|---|
| Tempo | ~150MB | 5Gi PVC |
| Traefik overhead | ~0 (już działa) | brak |
| **Razem** | **~150MB** | **5Gi** |

Masz ~2.5GB wolnego RAM — spokojnie.

---

## Co zobaczysz w Grafanie po wdrożeniu

**Explore → Tempo → Search:**
```
Service: traefik
Operation: router/landing, router/speakstats, router/grafana...
Duration: 12ms, 3.4s (whisper!), 45ms
Status: OK / ERROR
```

**Korelacja:**
- Kliknij trace → "View related logs" → otwiera Loki z logami z tego samego traceID
- Service Map → topologia serwisów widziana przez Traefik

---

## Status

- [x] Etap 1 — Tempo manifesty
- [x] Etap 2 — Grafana datasource
- [x] Etap 3 — Traefik config
- [x] Etap 4 — Kustomization
- [x] Etap 5 — Deploy i weryfikacja
- [x] Etap 6 — Dashboard import (Traefik dashboard uid: `qPdAviJmz`)
