---
name: find-security-vulnerabilities-in-code
description: Find security vulnerabilities in a codebase or repository with any AI agent — a white-box security review where the agent reads the source, reasons about the actual data flow, sinks, and authorization model, then attempts real exploitation of what it finds (live request where a running instance exists) so every reported issue has a working proof-of-concept instead of a noisy static-analysis alert. Covers injection, XSS, SSRF, broken access control and IDOR, insecure deserialization, secrets in code, unsafe dependencies, and business-logic flaws. Use when the user asks to security-scan, security-review, or audit their code, repo, or pull request for vulnerabilities.
license: Apache-2.0
---

# Find security vulnerabilities in code (agent-driven)

White-box security review. You read the source yourself, build a model of routes, sinks, and authorization checks, then attempt real exploitation against a running instance when one exists. Findings come with a proof-of-concept, so the output is a short list of proven issues rather than the hundreds of "potential" hits a pattern-matching scanner produces. The general entry point is **penetration-testing**; this is the source-focused workflow.

## Run it

You need no special engine — read the code and reason. Workflow:

1. **Scope it.** For a large repo, start with the risky subtree (auth, admin, payments, file upload, anything parsing user input) rather than the whole tree. For a branch/PR review, read the diff first (`git diff <base>...HEAD --name-only` then full content of changed files); a whole-repo read is wasteful.
2. **Map the surface.** Find routes/files served (`grep`/explore), auth entry points (login, token verification, middleware), DB/data access patterns (ORM calls, raw query strings), external requests (`fetch`, `requests`, `http` calls, shellouts), file upload/download handlers, serialization/deserialization sites, template rendering, and use of `eval`/`exec`/`subprocess`/`system`.
3. **Build the authorization model.** Which requests carry the user identity (JWT? session cookie? basic api key)? Where is the tenant/owner check done — middleware, per-query filter, or nowhere? Any query that filters by object id without also filtering by tenant/owner is a BOLA/IDOR candidate.
4. **Trace data flow.** From attacker-controlled input (body, query, path segment, header, filename, even data fetched from a third party) follow it to sinks (SQL, filesystem, templates, shell, URLs the server fetches). A sink with no escaping/parametrization between it and user input is a finding candidate.
5. **Confirm exploitability.** Ideally run a small reproduction against a running instance (the user's dev server, staging, or a local run you start yourself if the user allows). If nothing is running, do a careful manual trace and then mark the finding `unconfirmed` — say so, and describe the exploit path precisely enough that the user can verify quickly. This is the difference between "this looks unsafe" and a validated finding.
6. **Report.** One Markdown file (`code-review-report.md` or user's choice); per finding: title, severity, class (CWE if useful), file and line for sink and attacker entry, short reasoning for the data flow, PoC (HTTP request / code input), impact, and root-cause remediation. End with coverage notes (paths read, paths skipped and why). Ask the user before running anything that modifies their data.

Complementary tooling you can *also* run to speed the low-hanging fruit (still verify each hit manually):

- Secrets in history/code: `gitleaks detect --source .` or `trufflehog` — verify and report only what is actually live.
- Dependencies (SCA): `cargo audit` / `pip audit` / `npm audit` / `trivy fs --scanners vuln .` — useful for known-CVE dependencies.
- Broad SAST greps: `semgrep --config auto` or targeted `grep -rn "eval(\|exec(\|md5(\|SUID"` — treat as leads, not findings.

## What to look for (fast triage order by impact)

1. **Broken access control / IDOR (A01)** — object id taken from request but authorized by client: any endpoint that queries `db.get(id)` without an ownership filter. Profit for concrete proof.
2. **Injection (A03/A05)** — SQL string concatenation, shell executives, template injection, path concatenation for file paths (`../../` traversal), URL fetching of user-supplied URLs (SSRF).
3. **Auth flaws (A07)** — token comparison with `==` (timing-clean comparison), JWT verification missing exp/issuer/audience, tokens valid forever, password hashes with no salt, `Math.random()` used for tokens/ids.
4. **Crypto (A04)** — secret keys in source/config/env committed to repo, weak hashes (MD5/SHA1 for passwords), no TLS checks, custom crypto.
5. **Deserialization / integrity (A08)** — `yaml.load` without `SafeLoader`, `pickle.loads` on user data, accepting unsigned JARs/updates.
6. **Misconfig (A02)** — debug routes shipped, verbose error bodies, CSRF not enforced on state-changing routes, `Access-Control-Allow-Origin: *` with credentials, admin panels with no auth.
7. **Business logic (A06)** — missing quantity/price checks, no idempotency on payment mutation, race-prone check-then-set patterns (then two-step updates).
8. **Supply chain (A03)** — pin-less dependencies with wildcard versions, dependencies that pull install-time scripts, missing build provenance (report as hardening advice, not confirmed vuln).

## When to escalate to a full audit

This skill is a manual, single-operator review sized for speed. For a broader program-source review, use `bb-methodology` to set coverage goals and load the matching `hunt-*` or `vuln-*` playbooks. For source patterns that need a targeted hunting-class playbook, use `vuln-prototype-pollution`, `vuln-deserialization`, `vuln-ssti`, and similar — match the sink class and validate only against a local fixture or program-authorized environment.

## Complementary tooling note

This is exploit-validated review, not an exhaustive inventory. Keep a dependency scanner (SCA) and secret scanning in place for complete coverage of known-CVE dependencies and committed credentials; use this skill for the logic, authorization, and injection bugs those tools structurally cannot find.

## Fix and verify

For fixes to code you own, patch the root cause (the shared authorization helper, not just one route; ORM parameterization, not one string), run the project's tests, and re-run the finding's PoC only in an authorized local or test environment. Update the report with the verification result.
