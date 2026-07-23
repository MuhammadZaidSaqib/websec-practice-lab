websec-practice-lab

A personal lab for practicing web application security testing — applying concepts from TryHackMe and PortSwigger's Web Security Academy against real, self-built applications instead of pre-made lab boxes.

Why this repo exists

Learning security concepts in isolated labs is one thing; finding and fixing real bugs in a real codebase is another. This repo documents that process: setting up a proper attacker/target lab, testing methodically, and writing up findings the way a real security report would look.

Lab setup
Target: applications running locally (Windows host / local VM)
Attacker: Kali Linux (VMware), isolated virtual network
Tooling: Burp Suite Community Edition, nmap, manual source review
Findings
#	Target	Vulnerability	Class	Severity	Status
1	FalconStrix	Unauthenticated debug endpoints exposing internal telemetry	CWE-306 (Missing Authentication)	High	Fixed & Verified
2	FalconStrix	Auth lockout silently auto-bypassed; unlock endpoint missing role check	CWE-287 / CWE-863	Medium	Fixed & Verified

Each finding includes a full write-up (PDF) in /reports covering: summary, technical root cause, reproduction steps with request/response evidence, impact, CVSS-style severity reasoning, remediation code, and before/after verification.

Disclaimer

All testing in this repo was performed exclusively against my own locally-hosted projects, in an isolated lab environment I control. No third-party systems, production services, or other people's applications were tested.

Structure
/reports/          → vulnerability write-ups (PDF)
/FalconStrix/       → target application source (with fixes applied
