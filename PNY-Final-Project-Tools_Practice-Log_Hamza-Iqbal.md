# PNY Final Project Tools — Practice Log

**Student:** Hamza Iqbal · **Lab:** OWASP Juice Shop (own lab only) · **Date:** 17 Nov 2026
**Target:** `http://192.168.10.98:3000` — Windows IP changed from `192.168.10.210` to `192.168.10.98` during the session; curl to the new IP returned **200**.
**Status at 20:42 PKT, 17 Nov 2026:** Practice complete — all 17 tools covered, 3 Juice Shop challenges solved (Error Handling, Password Strength, Login Admin); GitHub portfolio push approved by the user at 20:42 PKT (checklist + hardening guide + practice log).

> Living log — update after each tool. Screenshots/outputs quoted here are from the 17 Nov 2026 Kali session in the "PNY Final Project Tools" chat.

## 1. Install check (Kali)

First check: 10 tools already present — whatweb, nmap, ffuf, gobuster, sqlmap, commix, hydra, nikto, nuclei, burpsuite.

Installed 17 Nov 2026:
- `apt install -y sublist3r zaproxy seclists` — sublist3r, **ZAP 2.17.0**, seclists (767 MB, 8 min 23 s)
- `pip3 install arjun --break-system-packages` — **arjun 2.2.7**
- `git clone https://github.com/s0md3v/XSStrike.git` + requirements — **XSStrike v3.1.5**, verified with `python3 xsstrike.py --help`
- `apt install -y docker.io` — docker.io 28.5.2 installed in Kali, but the Juice Shop image pull failed (IPv6 route unreachable: `dial tcp [2600:1f18:…]:443: network is unreachable`). Not needed — Juice Shop runs on Windows via Docker Desktop instead.

Still to do: **Wappalyzer** (Firefox extension only). **Docker** is needed only on Windows, where Juice Shop already runs. **Sublist3r** is for domains, not an IP target — no work for it in this lab.

## 2. P1 — Map the Target

### WhatWeb
`whatweb http://192.168.10.98:3000` → 200 OK, HTML5, Script[module], Title[OWASP Juice Shop], X-Frame-Options SAMEORIGIN. No HSTS or CSP seen — matches the WebScan finding (HSTS High, CSP Medium missing).

### Nmap
`nmap -sV -p 3000 192.168.10.98` → host DESKTOP-6DAJUGO up, port 3000 open. Service shown as "ppp?" because no Server banner was returned — normal; the fingerprint shows plain HTTP 200.

### Gobuster + ffuf (cross-checked, same 20 results)
`gobuster dir -u http://192.168.10.98:3000 -w /usr/share/seclists/Discovery/Web-Content/common.txt --exclude-length 9984`
`ffuf -u http://192.168.10.98:3000/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 9984` — 42 s, 176 req/s.

Juice Shop is an Angular SPA: every wrong URL returns 200 with the same 9,984-byte index.html, so the length had to be excluded or everything looks "found".

Real findings:
- `/ftp` (200) — the P5 finding (F5, High): unauthenticated listing
- `/robots.txt`, `/.well-known/security.txt`, `/security.txt` (200)
- `/promotion`, `/assets`, `/media`, `/video` (200/301)
- `/api`, `/rest`, `/profile`, `/redirect`, `/restaurants`, `/restore`, `/restricted` (500 — route exists, needs proper input/auth)

### Arjun
`arjun -u "http://192.168.10.98:3000/rest/products/search?q=test"` → parameter detected: **q**. (First run without `q=test` was skipped — the endpoint errors without a baseline value.)

## 3. P2 — Injection Hunter

### sqlmap — scanner missed it
`sqlmap -u "http://192.168.10.98:3000/rest/products/search?q=test" --batch` → "GET parameter 'q' does not seem to be injectable", "all tested parameters do not appear to be injectable". Cause: its probes got **40× HTTP 500**, so it could not tell true from false.

### Manual SQLi — confirmed (F1, Critical)
- `q=test` → 1 product id
- `q=' or 1=1--` → 0 (wrong form — the search query has parentheses, so this payload breaks it)
- `q=')) OR 1=1--` → **56** product ids

**Lesson: a scanner's "clean" report is not proof — verify manually.**

### XSStrike — no reflection (expected)
`python3 xsstrike.py -u "http://192.168.10.98:3000/rest/products/search?q=test"` → "No reflection found", WAF offline. Correct: this endpoint returns JSON and does not reflect input into HTML. Juice Shop's XSS is in the browser/DOM (P2's `<iframe src="javascript:alert('xss')">` in the search box), not on the API.

## 4. Nikto — complete 17 Nov 2026, 19:39 PKT

`nikto -h http://192.168.10.98:3000` → 8,916 requests, 28 items reported.

Real findings:
- **4 security headers missing:** Content-Security-Policy, Strict-Transport-Security (both already in WebScan), plus permissions-policy and referrer-policy (new)
- `robots.txt` lists `/ftp/` returning 200; `/ftp/` flagged interesting — P5 finding confirmed again
- No Server banner — matches WhatWeb/Nmap

Likely **false positives** (SPA returns 200 for everything): `.bash_history`, `.htpasswd`, `.mysql_history`, `.sh_history`, `.sqlite_history`, `*.json` (`/users.json`, `/login.json` …), `JAMonAdmin.jsp`. **Verified 19:44 PKT:** `curl -s http://192.168.10.98:3000/.bash_history | head -c 150` returned the Juice Shop HTML (`<!doctype html>`) — false positive **confirmed**.

## 5. Nuclei — complete 17 Nov 2026, 19:44 PKT

`nuclei -u http://192.168.10.98:3000` — first run installed templates v10.5.2; 11,330 templates, 3 min, **10 matches, all [info]** — no medium/high/critical.

- `/api-docs/swagger.yaml` exposed (API docs)
- `x-recruiting` header (`/#/jobs`)
- **8 security headers missing:** content-security-policy, strict-transport-security (both already in WebScan), permissions-policy, referrer-policy, x-permitted-cross-domain-policies, cross-origin-embedder-policy, cross-origin-opener-policy, cross-origin-resource-policy

Note: the target went unresponsive 30 times mid-scan (load from 9,757 connections) — scan still completed.

## 6. Burp Suite vs ZAP — 17 Nov 2026, 19:45–19:50 PKT

- `burpsuite` in Kali opens **Burp Suite Professional** with a licence-key screen. "Don't have a license key?" opens the PortSwigger Pro page in Firefox — no Community Edition option in the wizard. **User chose ZAP instead.**
- `zaproxy` → **ZAP 2.17.0** launched; session dialog (chose not to persist); Manage Add-ons shows all needed add-ons installed (Active scanner rules 78.0.0, Ajax Spider, Fuzzer, DOM XSS Active scanner…). **Setup:** close Add-ons window → Firefox manual proxy `127.0.0.1:8080` → browse `http://192.168.10.98:3000` → traffic should appear in ZAP History. Juice Shop is HTTP, so no CA certificate needed.
- Proxy was already configured in Firefox (127.0.0.1:8080) — only the "Also use this proxy for HTTPS" tick was missing.
- First browse → **"The proxy server is refusing connections"**. Cause: ZAP wasn't running — `ss -tlnp | grep 8080` returned nothing.
- Fix: relaunched `zaproxy` (session: do not persist → Start) → `ss` showed `LISTEN [::ffff:127.0.0.1]:8080` (java, pid 40245) — proxy up. Next: reload Juice Shop in Firefox and confirm traffic in ZAP History.
- Manual test: History ID 63 (`/rest/products/search`, 304) → Open/Resend with Request Editor. The opened request had `q=` empty (homepage load, not the `test` search) — payload still applies. First Send failed: ZAP "Failed to parse the header" — the request line had split into two lines (`1=1--` / `HTTP/1.1`); fixed with Backspace to rejoin. Resent `q='))%20OR%201=1--` → **HTTP 200 OK, 34,158-byte JSON, all products returned (first: id 1 Apple Juice)** — **SQLi confirmed via ZAP too** (20:24 PKT).
### Wappalyzer (fingerprinting) — install DONE 20:38 PKT
- Firefox extension: official Wappalyzer (120,166 users) from addons.mozilla.org — "Wappalyzer is ready to use". Note: `firefox <url>` as root fails ("Running Firefox as root in a regular user's session is not supported") — open the URL in Firefox's address bar instead, or run `su joe -c "firefox <url>"`.
- ZAP had stopped again (proxy refused connections at 20:39); relaunched with `zaproxy` and the shop reloaded.
- **Bonus: 3 Juice Shop challenges solved by our tool runs** (shown as green banners on reload): Error Handling (500s from sqlmap/Commix), Password Strength (Hydra admin login without SQLi), Login Admin (admin login).
- Result (20:42 PKT): **Angular 22.2.2, Zone.js, Node.js, TypeScript, Font Awesome, Google Font API** — confirms the P1 fingerprinting (Angular SPA on Node.js). **All 17 tools now covered — practice complete.**

### Hydra (login brute-force) — DONE 20:30 PKT
- `printf 'password\n123456\nadmin\nadmin123\n' > /tmp/pw.txt`, then `hydra -l admin@juice-sh.op -P /tmp/pw.txt 192.168.10.98 -s 3000 http-post-form "/rest/user/login:email=^USER^&password=^PASS^:Invalid email or password"`.
- **Found: `admin@juice-sh.op / admin123`** (4 tries). Note: Hydra printed many `[ERROR] received HTTP error code 401` lines — Juice Shop returns 401 on failed logins, and http-post-form expects 200; that is Hydra noise, not a failed test — the credential was still found.

### Commix (command injection) — DONE 20:29 PKT
- First two runs aborted: OS question → N, 500 question → N ("Skipping further testing"). Fix: `commix -u "http://192.168.10.98:3000/rest/products/search?q=test" --batch --os=unix --ignore-code=500` (flags answer both prompts).
- Full run tested: heuristic (basic) "might not be injectable", classic results-based, dynamic code evaluation, time-based (false positive detected), blind time-based, tempfile-based, file-based. Final: **"GET parameter 'q' does not seem to be injectable. All tested parameters do not appear to be injectable."** — correct and expected: `q` goes to a SQL query, not a shell command.

## 7. Final status — all done, 17 Nov 2026

| Tool | Status |
|---|---|
| Nuclei | Done — 10 matches, all info (swagger.yaml + 8 missing headers) |
| Burp Suite | Skipped — Kali opens the Pro licence screen; user chose ZAP |
| ZAP (zaproxy 2.17.0) | Done — proxy working, SQLi confirmed via manual resend (200 OK, 34,158-byte JSON, all products, 20:24 PKT) |
| Commix | Done — `q` not injectable (expected: `q` goes to a SQL query, not a shell command) |
| Hydra | Done — `admin@juice-sh.op / admin123` found in 4 tries (20:30 PKT) |
| Wappalyzer | Done — Angular 22.2.2, Zone.js, Node.js, TypeScript, Font Awesome, Google Font API (20:42 PKT) |
| Sublist3r | Installed; no work on an IP target |
| GitHub portfolio (guide § "To Secure a Job") | Ready — access-control checklist (P3) + hardening guide (P4) + this practice log; push approved 20:42 PKT |

## 8. Lessons worth keeping for Sir / the portfolio

1. Scanner "not injectable" ≠ safe — sqlmap missed the SQLi that a manual payload proved (56 vs 1).
2. SPA targets return 200 for every path — exclude the index length (9,984) in Gobuster/ffuf, and verify Nikto hits by checking the actual body.
3. A target's IP can change between sessions — re-check with `ipconfig` on Windows before assuming the old IP (`.210` → `.98`).
