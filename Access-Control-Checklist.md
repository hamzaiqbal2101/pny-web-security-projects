# Broken Access Control — Reusable Audit Checklist

**Student:** Hamza Iqbal · **Source:** PNY Trainings, Web Application Security — Project 3 (Broken Access Control Audit)
**Tested on:** OWASP Juice Shop v20.3.0, local Docker lab (`http://localhost:3000`)
**Lab only:** this checklist was built and tested against the student's own Juice Shop instance. Never run these tests against a system you do not own — unauthorized access is an offence under PECA 2016.

## Findings it caught on Juice Shop

| # | Finding | Location | Severity | Result |
|---|---------|----------|----------|--------|
| 1A | Forced browsing — administration page | `/#/administration` | — | PASS: 403 returned to non-admin user |
| 1B | Admin configuration API exposed | `GET /rest/admin/application-configuration` | High | FAIL: full config served to any user (CWE-862) |
| 2 | Basket IDOR | `GET /rest/basket/:id` (bid tampering 6 → 7) | High | FAIL: victim's basket viewed and editable (CWE-639) |

The one lesson behind both findings: **authorization must be checked on the server, on every request, against the authenticated user's identity and ownership — never inferred from what the interface chooses to show.** The admin page returned a correct 403 while the API behind it served the full configuration to the same user.

## The checklist

| # | Check | How to test | Result on Juice Shop |
|---|-------|-------------|----------------------|
| 1 | Route & endpoint inventory | List every page, route and API endpoint (recon map, Project 1) — access control cannot be tested on an unknown surface. | Done (Project 1) |
| 2 | Role matrix | For each endpoint, record who should be allowed: anonymous / user / other user / admin. Test against the matrix. | 2 findings |
| 3 | Forced browsing to admin pages | Request admin/hidden URLs directly as a normal user (e.g. `/#/administration`). Expect 401/403. | PASS — 403 |
| 4 | APIs behind protected pages | Call the API endpoints that serve protected pages — UI protection often hides unprotected APIs. | FAIL — Finding 1 |
| 5 | Unauthenticated access | Repeat key requests logged out entirely; admin/data endpoints must refuse. | Version/config data served |
| 6 | Horizontal IDOR — read | Change object ids (basket, order, profile, file) to another user's and attempt to read. | FAIL — Finding 2 |
| 7 | Horizontal IDOR — write | With a foreign id, attempt update/delete/checkout to test integrity, not just viewing. | FAIL — basket editable |
| 8 | Identifier predictability | Check whether ids are sequential integers (trivially enumerable) or unguessable references. | Sequential (6, 7, …) |
| 9 | Vertical escalation | As a normal user, invoke admin-only operations (user management, config, refunds). | Config read succeeded |
| 10 | Server-side enforcement | Confirm checks happen on the server for every request — not only in the frontend, and not only on some routes. | Missing on tested APIs |
| 11 | Error & logging behaviour | Unauthorized attempts should return generic 401/403 and be logged for monitoring. | 403 page OK; API silent 200 |

## Fixes (applied as guidance, from the Project 3 report)

- **Admin APIs:** enforce authentication + an admin role check on the server for every `/rest/admin/*` route; return 401/403 to unauthorized callers.
- **IDOR:** on every request, verify on the server that the requested object belongs to the authenticated user; reject mismatches with 403. Never trust a client-supplied identifier as proof of ownership. Prefer deriving the object from the session token instead of accepting an id at all.

Full evidence, screenshots and proof-of-concept steps: see `Project-3_Broken-Access-Control_Hamza-Iqbal.pdf` in this repo.
