# CV Gate — email OTP przed pobraniem CV

## Cel

Nauka flow OTP + transactional email. Nowy przycisk `✉ Request my resume`
działa **równolegle** do istniejącego `⇓ Download CV`, który zostaje bez zmian.

**Czym to jest, a czym nie:** dopóki `Download CV` linkuje do publicznego
`/cv/CV_DevOps-SRE_mjochemski.pdf`, gate **niczego nie chroni** — jest demem
działającego mechanizmu. Dlatego świadomie nie ma tu tokenów, podpisów, banów
ani limitów prób. Prawdziwą bramką staje się dopiero po [cutoverze](#cutover).

Plik PDF **nie przenosi się nigdzie** — pod `cv-gate` montuje ten sam hostPath,
co nginx landingu. Jedno źródło prawdy.

## Architektura

```
matflixlab.pl                       (Cloudflare Tunnel → Traefik)
  ├── /          → landing   (nginx)      ← ⇓ Download CV, bez zmian
  └── /cv-gate   → cv-gate   (Flask)      ← ✉ Request my resume
                     │  POST /cv-gate/request {email}
                     │    kod 6 cyfr → dict w pamięci, TTL 300 s
                     │    Resend API → mail
                     │  POST /cv-gate/verify {email, code}
                     │    OK → send_file(PDF)   ← odpowiedzią JEST plik
                     ▼
              hostPath /home/matflix/matflixlab/cv   (ten sam co nginx)
```

Routing po ścieżce na istniejącym hoście, więc: **żadnej nowej subdomeny, żadnej
zmiany w Cloudflare Tunnel i żadnego CORS-u** (to samo origin).

Traefik wybiera trasę po długości reguły, więc `PathPrefix(/cv-gate)` wygrywa
z `PathPrefix(/)` landingu. Nie trzeba ustawiać priorytetów.

---

## Etap 1 — Resend + DNS

1. Rejestracja na resend.com → **API Keys** → Create, scope `Sending access`.
2. **Domains** → Add Domain → `send.matflixlab.pl`.
   Subdomena, nie apex — reputacja maili transakcyjnych zostaje odseparowana od
   domeny głównej.
3. Resend wygeneruje trzy rekordy (MX, SPF TXT, DKIM). Wklej je w Cloudflare DNS
   **dokładnie tak, jak je pokazuje kreator**.
4. ⚠️ Rekordy CNAME w Cloudflare są domyślnie proxowane — przestaw każdy na
   **DNS only** (szara chmurka), inaczej DKIM się nie zweryfikuje.
5. Opcjonalnie, dla lepszej dostarczalności: TXT `_dmarc` →
   `v=DMARC1; p=none; rua=mailto:mjochemski@match-trade.com`
6. Verify.

Free tier: 3 000 maili/mies., **100/dzień**, log wysyłek widoczny w dashboardzie
przez 30 dni — to jest zarazem Twoja lista adresów, bez pisania linijki kodu.

---

## Etap 2 — `k8s/apps/cv-gate/`

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cv-gate-app
  namespace: matflixlab
data:
  app.py: |
    import os, re, secrets, time
    import requests
    from flask import Flask, request, jsonify, send_file

    app = Flask(__name__)

    RESEND_API_KEY = os.environ['RESEND_API_KEY']
    FROM_EMAIL     = os.environ.get('FROM_EMAIL', 'matflixlab <cv@send.matflixlab.pl>')
    CV_PATH        = os.environ.get('CV_PATH', '/cv/CV_DevOps-SRE_mjochemski.pdf')
    TTL            = 300

    EMAIL_RE = re.compile(r'^[^\s@]+@[^\s@]+\.[^\s@]{2,}$')
    CODES    = {}          # email -> (code, expires_at); pamięć procesu, 1 replica

    def prune():
        now = time.time()
        for k in [k for k, v in CODES.items() if v[1] < now]:
            CODES.pop(k, None)

    @app.get('/cv-gate/health')
    def health():
        return jsonify(ok=True, pending=len(CODES))

    @app.post('/cv-gate/request')
    def request_code():
        prune()
        email = (request.get_json(silent=True) or {}).get('email', '').strip().lower()
        if not EMAIL_RE.match(email):
            return jsonify(error='Invalid email address'), 400

        # Ważny kod już istnieje -> nie wysyłamy drugiego maila.
        # To jedyna ochrona przed zalewaniem cudzej skrzynki i wystarcza:
        # twardym sufitem jest limit 100 maili/dzień u Resend.
        entry = CODES.get(email)
        if entry and entry[1] > time.time():
            return jsonify(ok=True, expiresIn=int(entry[1] - time.time()))

        code = f'{secrets.randbelow(1_000_000):06d}'
        CODES[email] = (code, time.time() + TTL)

        try:
            r = requests.post(
                'https://api.resend.com/emails',
                headers={'Authorization': f'Bearer {RESEND_API_KEY}'},
                json={
                    'from': FROM_EMAIL,
                    'to': [email],
                    'subject': 'Your CV download code',
                    'text': (f'Your verification code: {code}\n\n'
                             'Valid for 5 minutes.\n\n-- matflixlab.pl'),
                },
                timeout=10,
            )
        except requests.RequestException as exc:
            CODES.pop(email, None)
            app.logger.error('resend request failed: %s', exc)
            return jsonify(error='Could not send email, try again later'), 502

        # Bez tego user widzi "wysłano" i czeka na maila, który nie przyjdzie.
        if not r.ok:
            CODES.pop(email, None)
            app.logger.error('resend %s: %s', r.status_code, r.text)
            return jsonify(error='Could not send email, try again later'), 502

        return jsonify(ok=True, expiresIn=TTL)

    @app.post('/cv-gate/verify')
    def verify():
        data  = request.get_json(silent=True) or {}
        email = data.get('email', '').strip().lower()
        code  = str(data.get('code', '')).strip()

        entry = CODES.get(email)
        if not entry or entry[1] < time.time():
            CODES.pop(email, None)
            return jsonify(error='Code expired - request a new one'), 404
        if entry[0] != code:
            return jsonify(error='Invalid code'), 401

        del CODES[email]                      # kod jednorazowy
        return send_file(CV_PATH, as_attachment=True,
                         download_name='CV_DevOps-SRE_mjochemski.pdf')

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=8080)
```

Normalizacja `.strip().lower()` jest w obu endpointach — bez niej `Jan@X.pl`
w pierwszym kroku i `jan@x.pl` w drugim to dwa różne klucze i kod się nie zgodzi.

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cv-gate
  namespace: matflixlab
  labels:
    app: cv-gate
spec:
  replicas: 1                       # kody są w pamięci procesu
  selector:
    matchLabels:
      app: cv-gate
  template:
    metadata:
      labels:
        app: cv-gate
      annotations:
        reloader.stakater.com/auto: "true"
    spec:
      containers:
        - name: cv-gate
          image: python:3.12-alpine
          command: ["sh", "-c"]
          args:
            - |
              pip install flask requests --quiet && python /app/app.py
          ports:
            - containerPort: 8080
          env:
            - name: RESEND_API_KEY
              valueFrom:
                secretKeyRef:
                  name: cv-gate-secret
                  key: RESEND_API_KEY
          volumeMounts:
            - name: app
              mountPath: /app
            - name: cv
              mountPath: /cv
              readOnly: true
          resources:
            requests:
              memory: "64Mi"
              cpu: "10m"
            limits:
              memory: "128Mi"
              cpu: "100m"
          readinessProbe:
            httpGet:
              path: /cv-gate/health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
      volumes:
        - name: app
          configMap:
            name: cv-gate-app
        - name: cv
          hostPath:
            path: /home/matflix/matflixlab/cv
            type: DirectoryOrCreate
```

`replicas: 1` jest wymagane, nie kosmetyczne — przy dwóch podach kod
wygenerowany przez jeden nie byłby widoczny dla drugiego.

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cv-gate
  namespace: matflixlab
spec:
  selector:
    app: cv-gate
  ports:
    - port: 8080
      targetPort: 8080
```

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cv-gate
  namespace: matflixlab
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - host: matflixlab.pl
      http:
        paths:
          - path: /cv-gate
            pathType: Prefix
            backend:
              service:
                name: cv-gate
                port:
                  number: 8080
```

### kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - sealed-secret.yaml
  - configmap.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

---

## Etap 3 — sekret i deploy

`secret.yaml` jest w `.gitignore` — commitujemy wyłącznie `sealed-secret.yaml`.

```bash
cd /home/matflix/matflixlab

cat > k8s/apps/cv-gate/secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: cv-gate-secret
  namespace: matflixlab
type: Opaque
stringData:
  RESEND_API_KEY: re_xxxxxxxxxxxxxxxx
EOF

kubeseal --format yaml \
  < k8s/apps/cv-gate/secret.yaml \
  > k8s/apps/cv-gate/sealed-secret.yaml

rm k8s/apps/cv-gate/secret.yaml
```

Dopisz `- apps/cv-gate` do sekcji Apps w [k8s/kustomization.yaml](k8s/kustomization.yaml).
To nie wpływa na klaster — ArgoCD reconciluje katalogi przez ApplicationSet —
ale bez tego lokalne `kustomize build k8s` przestaje odzwierciedlać repo.

```bash
kustomize build k8s > /dev/null     # walidacja przed pushem
git add k8s/apps/cv-gate/ k8s/kustomization.yaml
git commit -m "cv-gate: email OTP gate for CV download"
git push
```

ApplicationSet wykryje nowy katalog sam — **żadnego `kubectl apply`**.

```bash
sudo kubectl get pods -n matflixlab -l app=cv-gate
sudo kubectl logs deployment/cv-gate -n matflixlab --tail=30
```

---

## Etap 4 — frontend (`index.html`)

### 4.1 Drugi przycisk

Obok istniejącego linku w sekcji portfolio
([index.html:930](k8s/apps/landing/html/index.html#L930)), który **zostaje bez zmian**:

```html
<button class="btn" id="cv-gate-open" data-i18n="portfolio.request">
  &#9993; request my resume
</button>
```

### 4.2 Modal

```html
<div id="cv-modal" class="modal" hidden>
  <div class="modal-content">
    <button class="modal-close" aria-label="Close">&times;</button>

    <div id="step-email" class="modal-step">
      <h3 data-i18n="cvgate.title">Request my resume</h3>
      <p data-i18n="cvgate.intro">Enter your email and I'll send you a verification code.</p>
      <input type="email" id="cv-email" placeholder="your@email.com" autocomplete="email" />
      <button id="send-code-btn" data-i18n="cvgate.send">Send code</button>
      <div id="email-error" class="error" role="alert"></div>
    </div>

    <div id="step-code" class="modal-step" hidden>
      <h3 data-i18n="cvgate.codeTitle">Enter the code</h3>
      <p>Code sent to <span id="user-email"></span></p>
      <input type="text" id="cv-code" inputmode="numeric" autocomplete="one-time-code"
             maxlength="6" placeholder="123456" />
      <button id="verify-code-btn" data-i18n="cvgate.verify">Verify &amp; download</button>
      <div id="code-error" class="error" role="alert"></div>
      <p class="timer">Expires in <span id="countdown">5:00</span></p>
    </div>
  </div>
</div>
```

`autocomplete="one-time-code"` daje podpowiedź kodu z maila na iOS/Androidzie.

### 4.3 JS

```javascript
(() => {
  const modal = document.getElementById('cv-modal');
  const $ = id => document.getElementById(id);
  let timer = null;

  function reset() {
    clearInterval(timer);                  // bez tego dwa timery po ponownym otwarciu
    timer = null;
    $('step-email').hidden = false;
    $('step-code').hidden  = true;
    $('cv-email').value = '';
    $('cv-code').value  = '';
    $('email-error').textContent = '';
    $('code-error').textContent  = '';
    $('verify-code-btn').disabled = false;
  }
  const close = () => { modal.hidden = true; reset(); };

  $('cv-gate-open').addEventListener('click', () => { modal.hidden = false; $('cv-email').focus(); });
  modal.querySelector('.modal-close').addEventListener('click', close);
  modal.addEventListener('click', e => { if (e.target === modal) close(); });
  document.addEventListener('keydown', e => { if (e.key === 'Escape' && !modal.hidden) close(); });

  // Blokada podwójnego kliknięcia — inaczej user wysyła dwa żądania pod rząd.
  async function busy(btn, fn) {
    btn.disabled = true;
    const label = btn.textContent;
    btn.textContent = '…';
    try { await fn(); } finally { btn.disabled = false; btn.textContent = label; }
  }

  $('send-code-btn').addEventListener('click', e => busy(e.currentTarget, async () => {
    const email = $('cv-email').value.trim();
    $('email-error').textContent = '';
    try {
      const res = await fetch('/cv-gate/request', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email }),
      });
      const data = await res.json();
      if (!res.ok) { $('email-error').textContent = data.error; return; }

      $('step-email').hidden = true;
      $('step-code').hidden  = false;
      $('user-email').textContent = email;
      $('cv-code').focus();
      countdown(data.expiresIn ?? 300);
    } catch { $('email-error').textContent = 'Network error'; }
  }));

  $('verify-code-btn').addEventListener('click', e => busy(e.currentTarget, async () => {
    const email = $('user-email').textContent;
    const code  = $('cv-code').value.trim();
    $('code-error').textContent = '';
    try {
      const res = await fetch('/cv-gate/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, code }),
      });
      if (!res.ok) { $('code-error').textContent = (await res.json()).error; return; }

      // /verify zwraca bajty PDF-a, nie JSON — pobranie idzie przez blob.
      const url = URL.createObjectURL(await res.blob());
      Object.assign(document.createElement('a'),
        { href: url, download: 'CV_DevOps-SRE_mjochemski.pdf' }).click();
      URL.revokeObjectURL(url);

      if (window.umami) umami.track('cv-request-gated');
      close();
    } catch { $('code-error').textContent = 'Network error'; }
  }));

  function countdown(seconds) {
    clearInterval(timer);
    let left = seconds;
    const render = () => {
      const m = Math.floor(left / 60), s = left % 60;
      $('countdown').textContent = `${m}:${String(s).padStart(2, '0')}`;
    };
    render();                              // od razu, bez sekundy pustki
    timer = setInterval(() => {
      left -= 1;
      if (left <= 0) {                     // dekrementacja przed renderem,
        clearInterval(timer);              // inaczej mignie "0:-1"
        $('countdown').textContent = 'Expired';
        $('verify-code-btn').disabled = true;
        return;
      }
      render();
    }, 1000);
  }
})();
```

Ścieżki są względne (`/cv-gate/...`), więc nie ma hosta do skonfigurowania
ani nagłówków CORS do ustawienia.

### 4.4 CSS

```css
.modal {
  position: fixed; inset: 0; z-index: 9999;
  background: rgba(0,0,0,.7);
  display: flex; align-items: center; justify-content: center;
}
.modal[hidden] { display: none; }          /* [hidden] przegrywa z display:flex */

.modal-content {
  position: relative; background: var(--bg); color: var(--text);
  padding: 2rem; border-radius: 8px; max-width: 400px; width: 90%;
}
.modal-close {
  position: absolute; right: .75rem; top: .5rem;
  background: none; border: none; color: var(--text);
  font-size: 2rem; line-height: 1; cursor: pointer;
}
.modal-step input {
  width: 100%; padding: .75rem; margin: 1rem 0; font-size: 1rem;
  border: 1px solid var(--accent); border-radius: 4px;
  background: var(--bg); color: var(--text);
}
#cv-code { letter-spacing: .4em; text-align: center; font-variant-numeric: tabular-nums; }
.modal-step button {
  width: 100%; padding: .75rem; font-size: 1rem; cursor: pointer;
  background: var(--accent); color: var(--bg); border: none; border-radius: 4px;
}
.modal-step button:disabled { opacity: .6; cursor: not-allowed; }
.error { color: #ff4444; font-size: .9rem; margin-top: .5rem; }
.timer { text-align: center; margin-top: 1rem; color: var(--text-muted); font-size: .85rem; }
```

### 4.5 i18n

Dopisz do obu słowników `TRANSLATIONS`:
`portfolio.request`, `cvgate.title`, `cvgate.intro`, `cvgate.send`,
`cvgate.codeTitle`, `cvgate.verify`.

### 4.6 Deploy

```bash
git add k8s/apps/landing/html/index.html
git commit -m "landing: request my resume button + OTP modal"
git push
```

ArgoCD zsynchronizuje ConfigMap, Reloader zrestartuje poda landingu.

---

## Etap 5 — testy

```bash
# happy path
curl -s -X POST https://matflixlab.pl/cv-gate/request \
  -H 'Content-Type: application/json' -d '{"email":"twoj@email.com"}'
# {"expiresIn":300,"ok":true}   → sprawdź skrzynkę

curl -s -o cv.pdf -w '%{http_code} %{content_type}\n' \
  -X POST https://matflixlab.pl/cv-gate/verify \
  -H 'Content-Type: application/json' -d '{"email":"twoj@email.com","code":"123456"}'
# 200 application/pdf
```

| Test | Oczekiwane |
|---|---|
| zły kod | 401 `Invalid code` |
| kod po 5 min | 404 `Code expired` |
| ten sam kod drugi raz | 404 (skasowany po użyciu) |
| `/request` dwa razy w ciągu 5 min | 200, ale **tylko jeden mail** |
| pusty / błędny email | 400 |
| `Download CV` → `/cv/CV_...pdf` | 200 — stary przycisk działa dalej |
| `/cv-gate/health` | 200, `pending` = liczba aktywnych kodów |

Dostarczalność: wyślij na Gmail i Outlook, sprawdź w nagłówkach
`Authentication-Results` → `spf=pass`, `dkim=pass`.

**Razem ~2,5 h** plus czekanie na weryfikację domeny.

---

## Cutover

Gdy uznasz, że gate działa i chcesz, żeby faktycznie coś zamykał:

1. Usuń `⇓ Download CV` z `index.html`.
2. Usuń wolumen `pdf-dir` (`volumeMounts` **i** `volumes`) z
   [k8s/apps/landing/deployment.yaml](k8s/apps/landing/deployment.yaml) —
   pod `cv-gate` montuje ten katalog niezależnie, więc nic nie traci.
3. `Disallow: /cv/` w `robots.txt`.
4. Google Search Console → Usunięcia, jeśli PDF był zindeksowany
   (sprawdź: `site:matflixlab.pl filetype:pdf`).
5. Weryfikacja: `curl -sI https://matflixlab.pl/cv/CV_DevOps-SRE_mjochemski.pdf` → 404.

Do tego momentu gate jest demem — warto o tym pamiętać, zanim uznasz PDF za chroniony.

---

## Status

- [ ] Etap 1 — Resend: konto, API key, domena `send.matflixlab.pl`, DNS — **do zrobienia przez Ciebie**
- [x] Etap 2 — `k8s/apps/cv-gate/`: configmap, deployment, service, ingress, kustomization
- [x] Etap 3a — `apps/cv-gate` dopisane do [k8s/kustomization.yaml](k8s/kustomization.yaml)
- [ ] Etap 3b — **`kubeseal` na hoście** (patrz niżej) + push
- [x] Etap 4 — `index.html`: przycisk, modal, JS, CSS, klucze i18n EN+PL
- [x] Etap 5a — testy logiki aplikacji: 30/30 OK (walidacja, dedupe, jednorazowość,
      wygasanie, błąd Resend, routing)
- [ ] Etap 5b — testy end-to-end na klastrze + dostarczalność
- [ ] Cutover (później)

### Co musisz zrobić, zanim to pojedzie

`k8s/apps/cv-gate/kustomization.yaml` odwołuje się do `sealed-secret.yaml`,
którego jeszcze nie ma — **dopóki go nie wygenerujesz, `kustomize build k8s`
i sync ArgoCD będą failować.** Kolejność:

```bash
# na matflix-server, w /home/matflix/matflixlab
vim k8s/apps/cv-gate/secret.yaml        # wklej klucz z Resend zamiast re_REPLACE_ME
kubeseal --format yaml \
  < k8s/apps/cv-gate/secret.yaml \
  > k8s/apps/cv-gate/sealed-secret.yaml
rm k8s/apps/cv-gate/secret.yaml

kustomize build k8s > /dev/null         # musi przejsc
git add k8s/apps/cv-gate/ k8s/kustomization.yaml k8s/apps/landing/html/index.html
git commit -m "cv-gate: email OTP gate for CV download"
git push
```

`secret.yaml` jest w `.gitignore` (`k8s/**/secret.yaml`) — sprawdzone, nie trafi do repo.
