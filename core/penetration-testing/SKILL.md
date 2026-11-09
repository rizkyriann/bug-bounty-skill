---
name: penetration-testing
description: Entry point for penetration testing with any AI agent — run the test yourself (the agent is the pentest operator) against a web app, API, codebase, repository, URL, domain, or IP, using the tools installed on the box (nmap, ffuf, httpx, curl, nuclei, sqlmap, nikto, and other common Kali-style tools). Exploits and proves vulnerabilities (OWASP Top 10 and beyond — injection, XSS, SSRF, auth/access-control flaws, IDOR, business logic) and returns validated findings with replayable proof-of-concept requests collected into a single Markdown report. Use when the user asks to pentest, hack, security-scan, security-audit, or find vulnerabilities in an app, API, website, or repo. Routes to the more specific skills when the target is narrow.
license: Apache-2.0
---

# Run a penetration test (agent-driven)

You — the AI agent — are the pentest operator. There is no external scan engine and no orchestrating product: you scope the test, enumerate the target, test each vulnerability class with the tools available on the machine, validate every finding by replaying the request yourself, and write the report.

## Tool availability (check before any playbook)

Never assume a tool exists. Gates to run before executing any playbook command that names a binary:

1. `command -v <tool>` (Windows: `where`/`Get-Command`).
2. Missing → fall back to the documented curl/python equivalent for that technique (equivalent coverage must then be disclosed as weaker evidence in the report's coverage notes).
3. Only with the user's permission, install per the cheatsheet in `../../TOOLS.md` (apt / go / pipx / self-contained release binaries; on Windows prefer WSL for the full Kali set).

`../../TOOLS.md` holds the authoritative tool → fallback mapping and per-OS install commands.

## Operating model

- **Your brain, their hands.** Reason about the target and decide what to test next; use tools only as executors. Commonly useful Kali-family tools: `nmap`, `whatweb`/`wappalyzer`, `wafw00f`, `httpx`, `subfinder`/`amass`, `ffuf`/`dirsearch`, `arjun`, `nuclei`, `nikto`, `sqlmap`, `hydra`, `wfuzz`, `gau`/`waybackurls`. Check what exists (`command -v <tool>`). If a tool is missing, either do the same job with `curl`/`python` one-liners or install it if the user allows (e.g. `apt`, `pipx`, `go install`). Do not stall brainstorming for a missing tool — almost everything can be done with `curl` and a script.
- **Every finding is proven.** Store raw, replayable requests (exact `curl` commands) for each finding. A finding the agent cannot replay is unproven — label it "unconfirmed" or drop it.
- **Pacing.** Long task: work in phases and keep scratch notes (`recon.md`, `findings/`, `poc-requests.txt`) as you go, so progress survives context limits and can be compiled into the report at the end.
- **One report.** Final deliverable is a single Markdown file (`pentest-report.md` or a user-chosen name): severity summary, per-finding PoC + impact + remediation, coverage notes, appendix. See the target-specific skills for the exact structure.

## Route to the right skill

| Target / situation | Skill to load |
|---|---|
| Live web app, staging site, dev server | **web-app-penetration-testing** |
| REST / GraphQL / gRPC API | **api-security-testing** |
| Program-published source code / authorized repository review | **find-security-vulnerabilities-in-code** |
| "Test me against the OWASP Top 10" assessment | **owasp-top-10-testing** |
| Start/switch target or choose the next bug-bounty phase | **bb-methodology** |
| Validate a candidate / prepare evidence / write a submission | **triage-validation**, **evidence-hygiene**, **report-writing** |

Load the relevant local skill and follow it. This skill is the general entry point and routes within this project.

## Companion skill library (OMOP playbook collection)

A granular playbook library lives at `../playbooks/` in this project. Use these skills' own names as needed — they are per-phase, per-class, per-tool, and per-technology playbooks with concrete command sequences:

| Need | Library family | Examples |
|---|---|---|
| Recon / enum phases | `recon-*`, `pentest-recon`, `pentest-enum` | `recon-subdomain`, `recon-shodan`, `recon-secrets` |
| Attack-class depth | `vuln-*` | `vuln-idor`, `vuln-sqli`, `vuln-ssrf`, `vuln-jwt`, `vuln-http-smuggling` |
| Tool-specific usage | `tool-*` | `tool-nmap`, `tool-sqlmap`, `tool-caido`, `tool-dalfox` |
| Tech stack specifics | `tech-*`, `hunt-*` | `tech-stack-fingerprint`, `hunt-nextjs`, `hunt-nodejs` (optional) |
| Payload catalog | `payload-*` | `payload-ssti`, `payload-command-injection` |

`bb-methodology` is the default bug-bounty workflow. `pentest-mode` and `pentest-workflow` are optional conveniences; do not rely on host-specific slash commands or out-of-scope modes. `pentest-report` provides a CVSS-based template; use the severity and CVSS version required by the program.

## General workflow (applies to any target)

1. **Authorize & scope.** Confirm the target is owned or explicitly authorized (ask when it looks like third-party infrastructure). Record scope, out-of-scope paths, environment (prefer staging), wall-clock limits, and any rate limits/WAF.
2. **Recon.** Inventory technologies (`whatweb`, headers, JS bundles, `/robots.txt`), surface, ports (`nmap` when an IP/host is in scope), and the defense stack (`wafw00f`). Then pick the target-specific skill's enumeration phase.
3. **Test.** Run the target-specific skill's attack classes. Spend most effort on authorization-shaped bugs (IDOR, privilege escalation, tenant isolation) — they are where real impact lives.
4. **Validate.** Replay each candidate finding cleanly, confirm impact in the response body (not just an anomaly in status codes), and minimize the PoC.
5. **Report.** One Markdown report, findings ranked by proven impact, each with the exact replayable request (tokens redacted as `<REDACTED>`), impact, and root-cause remediation. State what was *not* tested honestly — partial coverage is not a clean bill of health.
6. **Offer next steps.** Retest the original PoCs only while authorized; use `triage-validation`, `evidence-hygiene`, and `report-writing` before any submission.

## Safety

Only test targets the user owns or is explicitly authorized to test. Never destroy data, flood targets, or out-of-scope hit. Keep credentials out of reports, shell history, and committed files (raw PoCs stay in a git-ignored scratch file or are redacted when shared).
