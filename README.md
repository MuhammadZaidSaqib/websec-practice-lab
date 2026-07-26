# websec-practice-lab
 
A personal lab for practicing web application security testing — applying
concepts from TryHackMe and PortSwigger's Web Security Academy against
real, self-built applications instead of pre-made lab boxes.
 
## Why this repo exists
 
Learning security concepts in isolated labs is one thing; finding and fixing
real bugs in a real codebase is another. This repo documents that process:
setting up a proper attacker/target lab, testing methodically, and writing
up findings the way a real security report would look — including tests
that turned up **no** vulnerability, since knowing what's *not* broken is
just as much a part of the process as finding what is.
 
## Lab setup
 
- **Target**: applications running locally (Windows host / local VM)
- **Attacker**: Kali Linux (VMware), isolated virtual network
- **Tooling**: Burp Suite Community Edition, nmap, manual source review
## Findings
 
| # | Target | Vulnerability | Class | Severity | Status |
|---|--------|---------------|-------|----------|--------|
| 1 | FalconStrix | Unauthenticated debug endpoints exposing internal telemetry | CWE-306 (Missing Authentication) | High | Fixed & Verified |
| 2 | FalconStrix | Auth lockout silently auto-bypassed; unlock endpoint missing role check | CWE-287 (Improper Authentication) / CWE-863 (Incorrect Authorization) | Medium | Fixed & Verified |
| 3 | FalconStrix | Cross-Site Request Forgery on state-changing endpoints (`/logout`, `/api/alerts/manual`) | CWE-352 (Cross-Site Request Forgery) | High | Fixed & Verified |
 
Each finding includes a full write-up (PDF) in [`/reports`](./reports) covering:
summary, technical root cause, reproduction steps with request/response
evidence, impact, CVSS-style severity reasoning, remediation code, and
before/after verification.
 
### Finding 2 in brief
 
A failed-login lockout displayed a "THREAT LEVEL: CRITICAL" screen but
silently cleared itself within seconds due to auto-unlock logic that only
checked open alert counts, not the actual lock cause — combined with an
unlock endpoint missing the role check its own code comments claimed to
enforce. Fixed by making the unlock logic reason-aware and adding a real,
admin-gated manual override.
 
### Finding 3 in brief
 
State-changing routes relied solely on the session cookie for
authorization, with no CSRF token and no `SameSite` cookie attribute.
Proven with two working proof-of-concept pages: a plain auto-submitting
form (`/logout`) and a `text/plain`-encoded form that bypassed the
browser's CORS preflight check (`/api/alerts/manual`). Fixed with
Flask-WTF CSRF tokens across every route/form plus `SameSite=Lax` as a
second layer of defense.
 
## Tested — No Vulnerability Found
 
Not every test turns up a bug. These were tested with real technique and
found to be properly defended; full write-ups are in [`/reports`](./reports)
for reference.
 
| # | Target | Test | Class | Result |
|---|--------|------|-------|--------|
| 1 | FalconStrix | IDOR on `/api/alerts/<id>/resolve` (cross-account resolve) | CWE-639 (Insecure Direct Object Reference) | Not a vulnerability — shared SOC-queue design, UI intent matches API enforcement |
| 2 | FalconStrix | SQL Injection across 4 input vectors (login form, alert message field, alert-ID parameter, username-availability endpoint) | CWE-89 (SQL Injection) | No injection — parameterized queries, strict route typing, and input allow-listing each independently block it |
 
### No-vulnerability tests in brief
