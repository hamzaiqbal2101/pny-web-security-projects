# Hardening Guide — Security Headers via Reverse Proxy

**Student:** Hamza Iqbal · **Source:** PNY Trainings, Web Application Security — Project 4 (Harden and Rescan)
**Target:** OWASP Juice Shop v20.3.0, local Docker lab — baseline `http://localhost:3000`, hardened `http://localhost:4000`
**Lab only:** built and tested against the student's own Juice Shop instance. Never apply or test these changes on a system you do not own — unauthorized access is an offence under PECA 2016.

## Idea in one line

Leave the application untouched. Put an **nginx reverse proxy** in front of it and enforce the security-header policy at the edge — uniformly, on every response (static pages, API responses, errors), the way production systems do it.

## Before — baseline scan (ZAP Automated Scan, Dev Standard policy)

8 alerts on `http://localhost:3000` — 0 High, 4 Medium, 3 Low, 1 Informational. Four were selected for remediation (all response-header issues a proxy can fix uniformly):

| # | Alert | Risk | Disposition |
|---|-------|------|-------------|
| 1 | Content Security Policy (CSP) Header Not Set (Systemic) | Medium | Targeted — fixed |
| 2 | Cross-Domain Misconfiguration (Systemic) | Medium | Targeted — fixed |
| 3 | Missing Anti-clickjacking Header | Medium | Targeted — fixed |
| 4 | Session ID in URL Rewrite (Systemic) | Medium | Residual — application design |
| 5 | Private IP Disclosure | Low | Residual — response body content |
| 6 | Timestamp Disclosure — Unix | Low | Residual — informational leakage |
| 7 | X-Content-Type-Options Header Missing | Low | Targeted — fixed |
| 8 | Modern Web Application (Systemic) | Informational | Not a vulnerability |

Each targeted finding was verified by hand first (Chrome DevTools → Network → document request → Response Headers): no CSP header at all; `Access-Control-Allow-Origin: *`; X-Frame-Options and X-Content-Type-Options present on the static document only — the application's API responses (most of its traffic) did not include them.

## The fix — nginx reverse proxy

An nginx container (`harden-proxy`) in front of the unchanged application. Clients reach the shop at `http://localhost:4000`; nginx forwards to the app on port 3000 and rewrites response headers on the way back.

```nginx
map $http_upgrade $connection_upgrade { default upgrade; '' close; }

server {
    listen 80;
    server_tokens off;

    location / {
        proxy_pass http://host.docker.internal:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_hide_header X-Frame-Options;
        proxy_hide_header X-Content-Type-Options;
        proxy_hide_header Access-Control-Allow-Origin;

        add_header Content-Security-Policy "default-src 'self'; script-src 'self'
            'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';
            img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self'
            ws: wss:; frame-src 'self' https://www.youtube.com;
            frame-ancestors 'none'; base-uri 'self'; form-action 'self'" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Access-Control-Allow-Origin "http://localhost:4000" always;
        add_header Referrer-Policy "no-referrer" always;
    }
}
```

Design notes:

- **Uniformity** — `proxy_hide_header` + `add_header` guarantees exactly one instance of each managed header on every response, with no duplicates.
- **Compatibility** — the CSP allows `'unsafe-inline'` / `'unsafe-eval'` for scripts. Juice Shop's Angular frontend requires them to function; this is a deliberate, documented trade-off, not an oversight.
- **Clickjacking** — `frame-ancestors 'none'` in the CSP plus `X-Frame-Options: SAMEORIGIN` protect modern and legacy browsers respectively.
- **Disclosure** — the CORS wildcard is replaced by the single legitimate origin; `server_tokens off` stops nginx leaking its version (the Server header now reads simply `nginx`).

## After — verification and rescan

Verified by hand through the proxy (DevTools, document request to `http://localhost:4000`): CSP present with the full policy; `Access-Control-Allow-Origin: http://localhost:4000` (wildcard gone); X-Frame-Options and X-Content-Type-Options present exactly once each; Referrer-Policy `no-referrer` added; Server header reduced to `nginx`.

ZAP was relaunched in a fresh session and the identical Automated Scan was run against `http://localhost:4000`:

| Baseline alert (:3000) | Risk | After rescan (:4000) | Verdict |
|---|---|---|---|
| CSP Header Not Set | Medium | Gone — replaced by CSP policy-quality notices (below) | RESOLVED |
| Cross-Domain Misconfiguration | Medium | Gone | RESOLVED |
| Missing Anti-clickjacking Header | Medium | Gone | RESOLVED |
| X-Content-Type-Options Missing | Low | Gone | RESOLVED |
| Session ID in URL Rewrite | Medium | Still present (socket.io session ids in URLs) | RESIDUAL |
| Private IP Disclosure | Low | Still present (internal addresses inside the app configuration body) | RESIDUAL |
| Timestamp Disclosure — Unix | Low | Still present | RESIDUAL |
| Modern Web Application | Info | Still present | NOT A FLAW |

**4 of 4 targeted findings resolved.** Application functionality through the proxy was unaffected (shop, basket, websockets all verified working).

### The rescan's new CSP notices are progress, not regressions

The rescan lists four CSP notices that did not exist at baseline (CSP Wildcard Directive, script-src unsafe-eval, script-src unsafe-inline, style-src unsafe-inline — all Medium). At baseline there was no policy for ZAP to grade at all, and "no policy" was itself the Medium finding. The scanner now finds a policy and critiques its strictness. The critique is accurate — the policy deliberately permits inline/eval script for Angular compatibility — and it defines the next hardening step precisely: eliminate the application's reliance on inline script (nonce- or hash-based CSP) and on eval, then remove both keywords.

## Residual risk — what a proxy cannot fix

- **Session ID in URL Rewrite (Medium):** the realtime channel places session identifiers in request URLs, where they leak via logs and Referer headers. Requires an application-level change (cookie/header-based session transport).
- **Private IP Disclosure (Low):** internal addresses appear inside the application's configuration response — the same exposure reported as Finding 1 of Project 3. Fix at the source by removing internal data from the client-facing configuration.
- **Policy tuning:** schedule the move to a nonce-based CSP without `'unsafe-inline'` / `'unsafe-eval'`.
- **Deployment:** the proxy centralizes header policy; the same pattern should also terminate TLS and add Strict-Transport-Security before any real deployment.

Full evidence, screenshots and the alert-by-alert comparison: see `Project-4_Harden-and-Rescan_Hamza-Iqbal.pdf` in this repo.
