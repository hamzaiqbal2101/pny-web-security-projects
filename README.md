# PNY Trainings — Web Application Security: Final Projects

Five hands-on lab assessments performed against **OWASP Juice Shop v20.3.0** (an intentionally vulnerable training application) in an isolated local lab, as the final projects of the Web Application Security specialization at **PNY Trainings**.

Each project was executed end-to-end by hand — reconnaissance, exploitation, verification, remediation — and documented in a professional report (PDF + editable DOCX).

## The Projects

| # | Project | Report | Highlights |
|---|---------|--------|------------|
| 1 | **Map the Target** — reconnaissance & attack-surface mapping | [PDF](Project-1_Map-the-Target_Hamza-Iqbal.pdf) · [DOCX](Project-1_Map-the-Target_Hamza-Iqbal.docx) (11 pp.) | ZAP spidering mapped 21 endpoints across `/api/` and `/rest/`; technology fingerprinting (Angular / Node.js / Express / SQLite); unauthenticated version disclosure |
| 2 | **Injection Hunter** — SQL injection & XSS | [PDF](Project-2_Injection-Hunter_Hamza-Iqbal.pdf) · [DOCX](Project-2_Injection-Hunter_Hamza-Iqbal.docx) (7 pp.) | SQLi login bypass (`' or 1=1--`) → full administrator session; DOM-based XSS via iframe `javascript:` payload — both exploited live with fixes and code examples |
| 3 | **Broken Access Control Audit** | [PDF](Project-3_Broken-Access-Control_Hamza-Iqbal.pdf) · [DOCX](Project-3_Broken-Access-Control_Hamza-Iqbal.docx) (11 pp.) | Basket IDOR — another user's basket read & edited by changing a client-held ID; admin configuration API exposed to normal users; includes an 11-point reusable BAC audit checklist |
| 4 | **Harden and Rescan** | [PDF](Project-4_Harden-and-Rescan_Hamza-Iqbal.pdf) · [DOCX](Project-4_Harden-and-Rescan_Hamza-Iqbal.docx) (12 pp.) | ZAP baseline (8 alerts) → nginx reverse-proxy hardening (CSP, CORS restriction, clickjacking & sniffing headers) → rescan: all 4 targeted findings resolved, verified header-by-header |
| 5 | **Mini Penetration Test** — full assessment | [PDF](Project-5_Mini-Penetration-Test_Hamza-Iqbal.pdf) · [DOCX](Project-5_Mini-Penetration-Test_Hamza-Iqbal.docx) (14 pp.) | 7 findings (1 Critical, 4 High, 2 Medium) incl. unauthenticated `/ftp` exposure of a password vault and an exposed `/metrics` endpoint; attack-path analysis, remediation roadmap, and a 10-minute live-demo runbook |

## Tools & Techniques

- **OWASP ZAP** 2.17.0 — spidering, automated & baseline scans, before/after verification
- **Browser DevTools** — session-storage tampering, response-header verification
- **Docker** — target hosting and the nginx hardening proxy
- Manual exploitation: SQL injection, DOM XSS, IDOR, forced browsing, sensitive-data exposure
- Reporting aligned with the **OWASP Web Security Testing Guide**, findings classified by severity and CWE

## Scope & Ethics

All testing was performed **only** against a deliberately vulnerable application running in my own local lab. No external system was touched. (Unauthorized testing of others' systems is a criminal offence under Pakistan's PECA 2016 — these projects are lab work, by design.)

## Author

**Hamza Iqbal** — Cybersecurity Engineer · Python security tooling · CLLMSE / CLLMSP certified
[GitHub](https://github.com/hamzaiqbal2101) · [LinkedIn](https://www.linkedin.com/in/hamza-iqbal-299528367) · [Certifications](https://github.com/hamzaiqbal2101/CERTIFICATIONS)

*Part of my security portfolio — see also: [portscanner](https://github.com/hamzaiqbal2101/portscanner) · [file-encryptor](https://github.com/hamzaiqbal2101/file-encryptor) · [password-strength-checker](https://github.com/hamzaiqbal2101/password-strength-checker) · [subdomain-enumerator](https://github.com/hamzaiqbal2101/subdomain-enumerator) · [file-integrity-checker](https://github.com/hamzaiqbal2101/file-integrity-checker)*
