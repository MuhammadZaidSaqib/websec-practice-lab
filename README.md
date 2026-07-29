
Readme · MD
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
| 4 | FalconStrix | Sessions remained valid after logout (old cookies kept working indefinitely) | CWE-613 (Insufficient Session Expiration) | High | Fixed & Verified |
 
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
 
### Finding 4 in brief
 
Clicking "Sign Out" only cleared the session cookie on the client — it
never invalidated anything server-side, since Flask's default sessions
store all session data inside the signed cookie itself with no
server-side record of active sessions. A cookie captured *before* logout
kept working indefinitely: viewing the dashboard, creating real alerts,
and downloading PDF reports all succeeded using a "logged out" cookie.
Fixed by adding a per-user `session_version` counter in the database,
checked on every request and incremented on logout — instantly
invalidating every previously issued cookie for that user.
 
## Tested — No Vulnerability Found
 
Not every test turns up a bug. These were tested with real technique and
found to be properly defended; full write-ups are in [`/reports`](./reports)
for reference.
 
| # | Target | Test | Class | Result |
|---|--------|------|-------|--------|
| 1 | FalconStrix | IDOR on `/api/alerts/<id>/resolve` (cross-account resolve) | CWE-639 (Insecure Direct Object Reference) | Not a vulnerability — shared SOC-queue design, UI intent matches API enforcement |
| 2 | FalconStrix | SQL Injection across 4 input vectors (login form, alert message field, alert-ID parameter, username-availability endpoint) | CWE-89 (SQL Injection) | No injection — parameterized queries, strict route typing, and input allow-listing each independently block it |
| 3 | FalconStrix | Stored XSS via the manual alert message field, traced across Live Alerts, User Activity, Incident Summary, and Resolved Cases | CWE-79 (Cross-Site Scripting) | No execution — payload consistently HTML-escaped on output across every surface tested |
 
### No-vulnerability tests in brief
 
**IDOR — `/api/alerts/<id>/resolve`:**
- Created alerts under three separate accounts (`zaid`, `humza`, `admin`) and attempted cross-account resolution via Burp.
- Every account could resolve every alert, regardless of who created it.
- Checked the actual UI for all three accounts — the "Solve" button was equally visible and clickable for every alert, for every account.
- Since the UI's own intent already treats alert resolution as a shared, universal action, the API enforcing exactly that is not a mismatch — it's correct behavior for a shared SOC queue.
- Bonus check: requested a non-existent alert ID and confirmed a proper `404`, ruling out missing existence validation too.
- **Conclusion:** Not a vulnerability.
**SQL Injection — 4 input vectors:**
- **Login form:** tried an auth-bypass payload (`administrator'--`) — login failed normally, no bypass.
- **Alert message field:** submitted a logic payload and a lone single quote — both stored and displayed back as exact literal text, no errors, no broken queries.
- **Alert-ID URL parameter:** replaced the numeric ID with a single quote — rejected at Flask's own routing layer (`<int:...>` type conversion) before ever reaching a database query.
- **Username-availability endpoint:** sent a properly URL-encoded logic payload — rejected outright by a strict server-side allow-list on allowed characters.
- Four different input types, three different underlying defense mechanisms, all independently effective.
- **Conclusion:** No injection possible through any of the four tested vectors.
**Stored XSS — manual alert message field:**
- Submitted `<script>alert('XSS-TEST')</script>` as an alert message via "Create Real Alert."
- Traced the same payload across every surface that displays it: Live Alerts (Description column), User Activity panel, Incident Summary panel, and — after resolving the case — the Resolved Cases table.
- No JavaScript execution occurred anywhere; every surface rendered the payload as plain, literal text.
- Confirmed via the raw API response that the payload is stored unmodified in the database, meaning escaping happens consistently at output/render time across the app, not just in one place.
- **Conclusion:** Not a vulnerability.
## Functional Bugs Found & Fixed Along the Way
 
Not everything found during testing was a security issue — a couple of
functional/data bugs turned up during the XSS assessment and were fixed
as part of keeping the app in a working state:
 
- **Missing resolution details for manual alerts:** the Solved Cases table
  showed blank `Resolved At` / `How Resolved` fields for manually created
  alerts, because the lookup query only matched resolution events by
  process ID — manual alerts have no process behind them. Fixed by also
  matching on the alert ID embedded in the resolution event's text.
- **Unclear blank fields for manual alerts:** `Process Name` / `PID` /
  `Process Created At` showed a bare `—` for manual alerts (which
  genuinely have no process), which looked like missing data. Changed to
  a clear `N/A — manual alert` label instead.
- **Broken PDF report generation:** report downloads failed with
  "PDF engine not available," despite ReportLab being installed. Traced
  to a corrupted Pillow (image library) binary dependency; fixed with a
  clean reinstall.
## Disclaimer
 
All testing in this repo was performed exclusively against my own
locally-hosted projects, in an isolated lab environment I control. No
third-party systems, production services, or other people's applications
were tested.
 
## Structure
 
```
/reports/          → vulnerability write-ups (PDF)
/FalconStrix/       → target application source (with fixes applied)
```
 
