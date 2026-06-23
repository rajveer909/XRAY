## What is XRAY

XRAY is a custom-built, automated web security auditing tool designed to identify misconfigurations and missing defensive mechanisms in web applications. Built with a fast, modern Python backend, XRAY allows security researchers and developers to quickly assess a target's security posture through passive scanning techniques.

Currently in its first phase, XRAY focuses on non-destructive reconnaissance, specifically analyzing HTTP Security Headers (like HSTS, CSP, and X-Frame-Options) to ensure modern defensive web practices are actively enforced.

Key Features:

Passive Header Auditing: Rapidly checks for missing or misconfigured security headers.

Built-in SSRF Protection: Strict input validation and sanitization at the API level to prevent Server-Side Request Forgery against internal or local networks.

Modern Tech Stack: Powered by a high-performance backend.

Disclaimer: XRAY is built for educational purposes and defensive security auditing. Only scan targets you have explicit permission to test.

## Project Structure

```
secscanner/
├── backend/
│   ├── main.py            # FastAPI app: SSRF-safe validation + header scan logic
│   └── requirements.txt
└── frontend/
    └── index.html         # Terminal-styled UI (Tailwind CDN, vanilla JS)
```

## How It Works

1. The frontend sends the target URL to `POST /api/scan/headers`.
2. The backend validates the URL:
   - Only `http`/`https` schemes allowed.
   - Blocked hostnames (`localhost`, etc.) rejected immediately.
   - The hostname is DNS-resolved and **every** resolved IP is checked against
     private / loopback / link-local / reserved / multicast / unspecified
     ranges (this also blocks things like the `169.254.169.254` cloud
     metadata address and obfuscated IP literals, since they resolve into
     the same blocked ranges).
   - Redirects are followed manually (not by the HTTP client) so every hop
     is re-validated — this stops an attacker from pointing a public URL at
     one that 302-redirects into your internal network.
3. If the URL passes validation, the backend fetches it and checks for:
   `Strict-Transport-Security`, `Content-Security-Policy`, `X-Frame-Options`,
   `X-Content-Type-Options`.
4. A risk level (`Low` / `Medium` / `High` / `Critical`) is computed from the
   ratio of missing to total headers, and a structured JSON report is returned.

## Running Locally

### 1. Backend

```bash
cd secscanner/backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

You should see the API come up at `http://127.0.0.1:8000`. Confirm with:

```bash
curl http://127.0.0.1:8000/
# {"status":"ok","service":"Security Header Scanner API","phase":1}
```

Interactive API docs (Swagger UI) are auto-generated at
`http://127.0.0.1:8000/docs`.

### 2. Frontend

No build step needed — just open the file directly in a browser:

```bash
cd secscanner/frontend
open index.html      # macOS
# or just double-click index.html / drag it into your browser
```

It's hardcoded to call the API at `http://127.0.0.1:8000`. If you change the
backend's port, update the `API_BASE` constant near the top of the `<script>`
block in `index.html`.

### 3. Try it

Scan a real public site, e.g. `https://github.com`, and confirm:
- Local/private targets (`127.0.0.1`, `192.168.1.5`, `169.254.169.254`,
  `localhost`) are rejected with a 400 error.
- Public sites return a populated report with present/missing headers and a
  risk level.

## Notes & Next Steps

- CORS is wide open (`allow_origins=["*"]`) for local dev convenience —
  restrict this before deploying anywhere public.
- This phase only checks 4 headers; later phases could add `Referrer-Policy`,
  `Permissions-Policy`, cookie flag checks, TLS/cert inspection, etc.
- Always get explicit authorization before scanning any target you don't own.
