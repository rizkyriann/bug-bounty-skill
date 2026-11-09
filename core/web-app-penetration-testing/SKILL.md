---
name: web-app-penetration-testing
description: Pentest a web app or website end to end with any AI agent — black-box testing of a live URL, staging environment, or local dev server. The agent drives the tools installed on the box (curl, ffuf, dirsearch, httpx, wafw00f, nuclei, sqlmap, hydra, wfuzz) and finds and exploits real vulnerabilities (auth bypass, broken access control, IDOR, injection, XSS, SSRF, business logic), proving each one with a working, replayable proof-of-concept request collected into a single Markdown report. Use when the user asks to pentest, hack, security-test, or audit their web app, website, web application, or staging site.
license: Apache-2.0
---

# Pentest a web application (agent-driven)

Black-box (and optionally source-assisted) penetration testing of a running web app. You are the operator: enumerate, attack, validate each finding by replaying the request yourself, and produce one Markdown report. There is no scan engine behind you — the tools (`curl`, `ffuf`, `dirsearch`, `httpx`, `wafw00f`, `nuclei`, `sqlmap`, `hydra`, `wfuzz`, and whatever else the box has) are just your hands. For API-first targets, prefer **api-security-testing**; the general entry point is **penetration-testing**.

## 1. Confirm authorization and scope

Before touching the target, establish:

- **The target is the user's** (or they are explicitly authorized to test it). Never pentest a third-party site on a hunch.
- **Which environment.** Prefer staging over production; you will send real exploit payloads and may create/modify data.
- **Out-of-scope paths** — payment flows, mass-email endpoints, admin destructive actions, third-party SSO providers.
- **Credentials.** Most real vulnerabilities live behind login. Without a test account you only ever see the marketing surface. Ask for at least one account, ideally two in different tenants/privileges.
- **Known constraints** — rate limits, WAF (`wafw00f <base-url>` to check), maintenance windows.

Ask for anything missing rather than guessing. Store credentials in a local git-ignored scratch file, not in commands that persist in shell history.

## 2. Map the application

Keep findings in scratch notes (`recon.md`, `endpoints.md`) — the report will cite them.

- **Technology fingerprint:** `whatweb https://staging.example.com`, response headers, cookies (session cookie names reveal the framework), `/robots.txt`, `/sitemap.xml`, JS bundle contents.
- **Content discovery:** `ffuf -w <wordlist> -u https://staging.example.com/FUZZ -mc 200,301,401,403 -ac` (auto-calibrate against a random baseline path; filter noise). `dirsearch` works too. Look for: admin panels, api docs (`/swagger`, `/api-docs`, `/openapi.json`, `/graphql`), backup files, `/.git/config`, `/.env`, management endpoints (`/actuator`, `/debug`).
- **Endpoint/parameter extraction from JS:** fetch app JS bundles, grep for `fetch(`, `axios`, `/api/`, `/v1/`, field names; pull historical URLs with `waybackurls`/`gau` and check for still-live paths.
- **Hidden parameters:** `arjun -u <url>` on key endpoints.
- **Auth surface:** login, register, password-reset, MFA, logout, session refresh. Note how sessions/tokens are issued and where they are trusted.
- **Baseline:** with a valid session, sample a few authenticated pages and record what is normal — you'll compare anomalies against this.

## 3. Attack, class by class

Prove each finding with an exact, replayable `curl` command. Attack sequence, tuned for web apps:

**Authentication & session (A07).** Test creds: weak/default passwords on an admin panel (`hydra`/manual), username enumeration via different messages/latency/forgot-password behavior, password-reset token reuse/no-expiry/predictability, login after no rate limit (try 25 rapid attempts). Session: cookie flags (`Secure`, `HttpOnly`, `SameSite`), session fixation (set a known cookie then log in — check whether the session id changed), logout not invalidating server-side, tokens valid after logout, JWT alg/none confusion and signature stripping on token-based sessions.

**Access control & IDOR (A01) — the highest-value class.** With account B, request account A's objects: swap ids/uuids in URLs, JSON bodies, and query parameters (`GET /app/orders/501`, `?user_id=`, POST body `{"account_id": 1}`). Try uuid→numeric, sequential ids, and mixing endpoints. With a low-priv account, request admin pages and admin actions directly (`/admin/users` — note a `403` page that leaks data, a `302` that still serves content with the right headers, and route-style variants (`/ADMIN/users`, `/%41dmin`) all count). Horizontal → then vertical escalation.

**Injection (A03/A05).** SQL injection on every parameter, including JSON body fields and headers (`'`, `"`, `1' AND '1'='1`, timing payloads `1' AND SLEEP(5)-- -`); use `sqlmap -u <url> --data <body> --batch -p <param>` only on endpoints where manual testing says yes, with `--risk=1 --level=1 --threads=1` on production-ish targets. Command injection (`;`, `|`, backticks), template injection (`{{7*7}}`, `${7*7}`), XXE on any XML-consuming form. XSS: reflected (scoreboard than `alert(1)` — test event handlers and context: attribute vs body vs JS string), stored (profile fields, filenames, markdown), and DOM-XSS from JS-bundle sinks. Validate by observing reflection in the returned HTML with proper context — a payload echo in JSON-escaped form is not XSS.

**SSRF (A01).** Features fetching URLs (importers, webhooks, avatar/preview, PDF generators): point at `http://127.0.0.1:PORT`, internal hostnames, `169.254.169.254` (cloud metadata), DNS-only canaries. Proof: internal-only responses, timing deltas, or canary callbacks.

**Business logic (A06).** Price/quantity tampering in checkout, negative quantities, race conditions (double-use of a coupon, `curl` parallel replays), workflow skipping (call step-3 endpoint without step-2), currency manipulations, repeated registration with same email, upload overwrites.

**Misconfiguration (A02/A05).** CORS with `Origin: https://evil.com`, verbose stack traces from malformed input, HTTP-methods and verbs tampering, `TRACE`/`OPTIONS` probes, headers like `X-Forwarded-For`, `X-Original-URL` bypassing path-based allowlists, missing rate limits on sensitive actions, debug/backup files found in step 2, no TLS enforcement, missing CSP and security headers (finding: low/info, but note it).

**Automated sweep:** after mapping, run `nuclei -u <base-url> -silent` (templates `exposures/`, `vulnerabilities/`) as a broad net for known issues; treat each hit as a *lead* — verify by replaying the request yourself before adding it to the report.

**Playbook depth:** for a specific class that needs deeper technique coverage than the summary above, load the matching playbook from this project's `../playbooks/` (see **penetration-testing** § Companion skill library) and follow its concrete commands: `vuln-jwt` for token flaws, `vuln-idor` for access control, `vuln-http-smuggling`/`vuln-cache-deception` for request handling, `vuln-race-conditions` for races, `vuln-waf-bypass` when a WAF is in the way.

## 4. Validate findings

1. Replay the exact request in a clean shell; capture full status/headers/body.
2. Confirm *impact*, not just anomaly — a 403 body count differently from a 200 with another user's data; an echo of your own input is not a vuln.
3. Minimize the PoC: strip irrelevant headers, smallest request that still proves it.
4. Rate severity (critical/high/medium/low/info) with one sentence tied to real impact.
5. Recheck after any target-side change; stale findings are worse than none.

## 5. Report

One file, `pentest-report.md`, in the working directory (or a user-chosen name):

```markdown
# Web App Penetration Test Report — <target>

- Date / Environment (staging/prod) / Authorization: confirmed
- Scope: <paths> — Out of scope: <paths>
- Overall risk: <sev distribution table>

## Executive summary
<3–6 sentences>

## Findings
### [SEVERITY] <title>
- Evidence: exact replayable request (curl) + response excerpt (tokens as <REDACTED>)
- Impact / Remediation (root cause, not one patch point)

## Coverage notes
<what was tested per class, what could not be tested and why (e.g. no second account)>
## Appendix
<tools run, wordlists used, unconfirmed observations>
```

For a retest, replay each original PoC against the program-authorized environment and mark each finding Fixed / Still-exploitable in the report. Do not probe a fixed production target unless the program still authorizes testing there.

## Safety

Only test what you are authorized to. Never destroy data, flood with requests, or email real users. No DoS — a single oversized request or one race probe is enough to prove resource/state bugs. Keep credentials out of reports, shell history, and committed files.
