# CV Download — Email Confirmation Gate

## Cel

Dodanie weryfikacji email przed pobraniem CV z landing page:

1. User klika "Download CV"
2. Modal z formularzem — podaje email
3. System generuje 6-cyfrowy kod OTP (ważny 5 min)
4. Kod wysyłany na email
5. User wpisuje kod w tym samym modalu
6. Po poprawnej weryfikacji — CV pobiera się automatycznie

**Technologia:** Cloudflare Workers (serverless) — zero obciążenia hosta, darmowe do 100k req/dzień.

---

## Architektura

```
Landing page (matflixlab.pl)
    │  user klika "Download CV"
    ▼
Modal z formularzem email
    │  POST /cv-gate/request
    ▼
Cloudflare Worker
    │  generuje 6-cyfrowy kod
    │  zapisuje w KV store {email: kod, expires: timestamp+5min}
    │  wysyła email przez MailChannels (darmowe dla Workers)
    ▼
User dostaje email z kodem
    │  wpisuje w modal
    │  POST /cv-gate/verify {email, code}
    ▼
Worker weryfikuje kod
    │  sprawdza KV store
    │  jeśli OK → zwraca signed URL do CV (ważny 10 min)
    │  jeśli błąd → error message
    ▼
Frontend pobiera CV przez signed URL
```

---

## Stack techniczny

| Komponent | Technologia | Koszt |
|---|---|---|
| Backend API | Cloudflare Workers | $0 (<100k req/dzień) |
| Storage (kody OTP) | Cloudflare KV | $0 (<1k writes/dzień) lub $0.50/miesiąc unlimited |
| Email wysyłka | MailChannels (przez Workers) | $0 |
| CV storage | Cloudflare R2 bucket | $0 (10GB free) |
| Frontend | Vanilla JS (landing page) | $0 |

**Razem: $0/miesiąc** przy normalnym ruchu (do 1000 pobrań/dzień).

---

## Etap 1 — Cloudflare Workers: setup projektu

### 1.1 Instalacja Wrangler CLI

```bash
npm install -g wrangler
wrangler login
```

### 1.2 Utworzenie projektu Worker

```bash
mkdir ~/cv-gate-worker && cd ~/cv-gate-worker
wrangler init cv-gate
# wybierz: TypeScript: No, Git: Yes
```

### 1.3 Struktura projektu

```
cv-gate-worker/
├── src/
│   └── index.js          ← główna logika Worker
├── wrangler.toml         ← konfiguracja (KV namespace, routes)
└── package.json
```

---

## Etap 2 — Cloudflare KV: storage dla kodów OTP

### 2.1 Utworzenie KV namespace

```bash
wrangler kv:namespace create CV_CODES
wrangler kv:namespace create CV_CODES --preview  # dla testów lokalnych
```

Output:
```
Created namespace with title "cv-gate-CV_CODES"
{ binding = "CV_CODES", id = "abc123..." }
```

### 2.2 Dodanie do wrangler.toml

```toml
name = "cv-gate"
main = "src/index.js"
compatibility_date = "2024-01-01"

[[kv_namespaces]]
binding = "CV_CODES"
id = "abc123..."  # z poprzedniego kroku

[[kv_namespaces]]
binding = "CV_CODES"
id = "preview_id..."
preview_id = "preview_id..."
```

---

## Etap 3 — Worker: kod backendu

### 3.1 src/index.js

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    
    // CORS headers
    const corsHeaders = {
      'Access-Control-Allow-Origin': 'https://matflixlab.pl',
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };
    
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }
    
    // POST /request — generuj i wyślij kod
    if (url.pathname === '/request' && request.method === 'POST') {
      const { email } = await request.json();
      
      // Walidacja email
      if (!email || !email.match(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)) {
        return jsonResponse({ error: 'Invalid email' }, 400, corsHeaders);
      }
      
      // Generuj 6-cyfrowy kod
      const code = Math.floor(100000 + Math.random() * 900000).toString();
      
      // Zapisz w KV (TTL 5 min = 300s)
      await env.CV_CODES.put(email, code, { expirationTtl: 300 });
      
      // Wyślij email przez MailChannels
      await sendEmail(email, code);
      
      return jsonResponse({ 
        success: true, 
        message: 'Code sent to email',
        expiresIn: 300 
      }, 200, corsHeaders);
    }
    
    // POST /verify — weryfikuj kod
    if (url.pathname === '/verify' && request.method === 'POST') {
      const { email, code } = await request.json();
      
      // Pobierz kod z KV
      const storedCode = await env.CV_CODES.get(email);
      
      if (!storedCode) {
        return jsonResponse({ error: 'Code expired or not found' }, 404, corsHeaders);
      }
      
      if (storedCode !== code) {
        return jsonResponse({ error: 'Invalid code' }, 401, corsHeaders);
      }
      
      // Usuń użyty kod
      await env.CV_CODES.delete(email);
      
      // Wygeneruj signed URL do CV (ważny 10 min)
      const signedUrl = await generateSignedUrl(env);
      
      return jsonResponse({ 
        success: true, 
        downloadUrl: signedUrl 
      }, 200, corsHeaders);
    }
    
    return jsonResponse({ error: 'Not found' }, 404, corsHeaders);
  }
};

// Helper: JSON response z CORS
function jsonResponse(data, status, corsHeaders) {
  return new Response(JSON.stringify(data), {
    status,
    headers: { 'Content-Type': 'application/json', ...corsHeaders }
  });
}

// Helper: wysyłka email przez MailChannels
async function sendEmail(email, code) {
  const emailBody = {
    personalizations: [{
      to: [{ email }]
    }],
    from: { email: 'noreply@matflixlab.pl', name: 'matflixlab CV Gate' },
    subject: 'Your CV download code',
    content: [{
      type: 'text/plain',
      value: `Your verification code: ${code}\n\nValid for 5 minutes.\n\n-- matflixlab.pl`
    }]
  };
  
  await fetch('https://api.mailchannels.net/tx/v1/send', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(emailBody)
  });
}

// Helper: generuj signed URL do CV w R2
async function generateSignedUrl(env) {
  // Opcja A — zwróć bezpośredni link do R2 bucket (jeśli public)
  return 'https://cv.matflixlab.pl/cv.pdf';
  
  // Opcja B — signed URL z R2 (wymaga R2 bucket binding)
  // const object = await env.CV_BUCKET.get('cv.pdf');
  // return await object.generateSignedUrl({ expiresIn: 600 });
}
```

---

## Etap 4 — Cloudflare R2: storage dla CV

### 4.1 Utworzenie R2 bucket

```bash
wrangler r2 bucket create matflixlab-cv
```

### 4.2 Upload CV

```bash
wrangler r2 object put matflixlab-cv/cv.pdf --file=/path/to/cv.pdf
```

### 4.3 Public access (opcjonalnie)

Cloudflare dashboard → R2 → `matflixlab-cv` → Settings → **Public Access** → Enable

URL będzie: `https://pub-xxxxx.r2.dev/cv.pdf`

Alternatywnie: Custom domain `cv.matflixlab.pl` → R2 bucket przez Cloudflare DNS.

---

## Etap 5 — Deploy Worker

```bash
cd ~/cv-gate-worker
wrangler deploy
```

Output:
```
Published cv-gate (1.23s)
  https://cv-gate.your-subdomain.workers.dev
```

### 5.1 Custom domain (opcjonalnie)

Cloudflare dashboard → Workers & Pages → cv-gate → Settings → Triggers → **Custom Domains**

Dodaj: `api.matflixlab.pl` → route: `/cv-gate/*`

Endpoint będzie: `https://api.matflixlab.pl/cv-gate/request`

---

## Etap 6 — Frontend: integracja z landing page

### 6.1 Modal HTML (dodać do index.html)

```html
<!-- CV Download Modal -->
<div id="cv-modal" class="modal" style="display:none;">
  <div class="modal-content">
    <span class="close">&times;</span>
    
    <!-- Step 1: Email input -->
    <div id="step-email" class="modal-step">
      <h3>Download CV</h3>
      <p>Enter your email to receive a verification code</p>
      <input type="email" id="cv-email" placeholder="your@email.com" />
      <button id="send-code-btn">Send Code</button>
      <div id="email-error" class="error"></div>
    </div>
    
    <!-- Step 2: Code input -->
    <div id="step-code" class="modal-step" style="display:none;">
      <h3>Enter Verification Code</h3>
      <p>Code sent to <span id="user-email"></span></p>
      <input type="text" id="cv-code" placeholder="123456" maxlength="6" />
      <button id="verify-code-btn">Verify & Download</button>
      <div id="code-error" class="error"></div>
      <p class="timer">Code expires in <span id="countdown">5:00</span></p>
    </div>
  </div>
</div>
```

### 6.2 JavaScript logika

```javascript
// cv-gate.js

const API_URL = 'https://api.matflixlab.pl/cv-gate';

// Otwórz modal po kliknięciu "Download CV"
document.querySelector('.download-cv-btn').addEventListener('click', (e) => {
  e.preventDefault();
  document.getElementById('cv-modal').style.display = 'block';
});

// Zamknij modal
document.querySelector('.close').addEventListener('click', () => {
  document.getElementById('cv-modal').style.display = 'none';
  resetModal();
});

// Krok 1: Wyślij kod na email
document.getElementById('send-code-btn').addEventListener('click', async () => {
  const email = document.getElementById('cv-email').value;
  
  try {
    const res = await fetch(`${API_URL}/request`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email })
    });
    
    const data = await res.json();
    
    if (!res.ok) {
      document.getElementById('email-error').textContent = data.error;
      return;
    }
    
    // Przejdź do kroku 2
    document.getElementById('step-email').style.display = 'none';
    document.getElementById('step-code').style.display = 'block';
    document.getElementById('user-email').textContent = email;
    startCountdown(300); // 5 min = 300s
    
  } catch (err) {
    document.getElementById('email-error').textContent = 'Network error';
  }
});

// Krok 2: Weryfikuj kod i pobierz CV
document.getElementById('verify-code-btn').addEventListener('click', async () => {
  const email = document.getElementById('cv-email').value;
  const code = document.getElementById('cv-code').value;
  
  try {
    const res = await fetch(`${API_URL}/verify`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, code })
    });
    
    const data = await res.json();
    
    if (!res.ok) {
      document.getElementById('code-error').textContent = data.error;
      return;
    }
    
    // Pobierz CV
    window.location.href = data.downloadUrl;
    
    // Zamknij modal
    document.getElementById('cv-modal').style.display = 'none';
    resetModal();
    
  } catch (err) {
    document.getElementById('code-error').textContent = 'Network error';
  }
});

// Countdown timer
function startCountdown(seconds) {
  const display = document.getElementById('countdown');
  let remaining = seconds;
  
  const timer = setInterval(() => {
    const min = Math.floor(remaining / 60);
    const sec = remaining % 60;
    display.textContent = `${min}:${sec.toString().padStart(2, '0')}`;
    
    if (--remaining < 0) {
      clearInterval(timer);
      display.textContent = 'Expired';
      document.getElementById('verify-code-btn').disabled = true;
    }
  }, 1000);
}

function resetModal() {
  document.getElementById('step-email').style.display = 'block';
  document.getElementById('step-code').style.display = 'none';
  document.getElementById('cv-email').value = '';
  document.getElementById('cv-code').value = '';
  document.getElementById('email-error').textContent = '';
  document.getElementById('code-error').textContent = '';
}
```

### 6.3 CSS dla modalu

```css
/* cv-modal.css */
.modal {
  position: fixed;
  z-index: 9999;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.7);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: var(--bg);
  padding: 2rem;
  border-radius: 8px;
  max-width: 400px;
  width: 90%;
  position: relative;
}

.close {
  position: absolute;
  right: 1rem;
  top: 0.5rem;
  font-size: 2rem;
  cursor: pointer;
  color: var(--text);
}

.modal-step input {
  width: 100%;
  padding: 0.75rem;
  margin: 1rem 0;
  border: 1px solid var(--accent);
  border-radius: 4px;
  background: var(--bg);
  color: var(--text);
  font-size: 1rem;
}

.modal-step button {
  width: 100%;
  padding: 0.75rem;
  background: var(--accent);
  color: var(--bg);
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
}

.error {
  color: #ff4444;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}

.timer {
  text-align: center;
  margin-top: 1rem;
  color: var(--text-muted);
  font-size: 0.9rem;
}
```

---

## Etap 7 — MailChannels: weryfikacja domeny (opcjonalnie)

MailChannels wymaga weryfikacji domeny żeby uniknąć spam filtering.

### 7.1 Dodaj DNS TXT record

Cloudflare DNS → Add record:
```
Type: TXT
Name: _mailchannels
Content: v=mc1 cfid=matflixlab.workers.dev
```

### 7.2 SPF record (opcjonalnie)

```
Type: TXT
Name: @
Content: v=spf1 include:relay.mailchannels.net ~all
```

---

## Etap 8 — Testy

### 8.1 Test lokalny (Wrangler dev)

```bash
cd ~/cv-gate-worker
wrangler dev
```

Endpoint: `http://localhost:8787/request`

```bash
# Test request
curl -X POST http://localhost:8787/request \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com"}'

# Sprawdź KV
wrangler kv:key get --binding=CV_CODES "test@example.com"
```

### 8.2 Test produkcyjny

```bash
curl -X POST https://api.matflixlab.pl/cv-gate/request \
  -H "Content-Type: application/json" \
  -d '{"email":"your@email.com"}'

# Sprawdź email → wpisz kod w modal
```

---

## Podsumowanie kroków implementacji

| Krok | Opis | Czas |
|---|---|---|
| 1 | Setup Wrangler CLI + projekt Worker | 10 min |
| 2 | Cloudflare KV namespace | 5 min |
| 3 | Worker backend kod | 30 min |
| 4 | R2 bucket + upload CV | 10 min |
| 5 | Deploy Worker + custom domain | 10 min |
| 6 | Frontend modal + JS logika | 45 min |
| 7 | MailChannels DNS weryfikacja | 5 min |
| 8 | Testy end-to-end | 15 min |
| **Razem** | | **~2h** |

---

## Wartość dla CV/portfolio

```
Implemented serverless email verification gate for CV downloads:
- Cloudflare Workers backend with OTP generation
- KV store for session management (5-min TTL)
- MailChannels integration for zero-cost email delivery
- Frontend modal with countdown timer and error handling
- Zero infrastructure overhead, scales automatically
```

---

## Status

- [ ] Etap 1 — Wrangler setup
- [ ] Etap 2 — KV namespace
- [ ] Etap 3 — Worker backend
- [ ] Etap 4 — R2 bucket + CV upload
- [ ] Etap 5 — Deploy Worker
- [ ] Etap 6 — Frontend integracja
- [ ] Etap 7 — MailChannels DNS
- [ ] Etap 8 — Testy
