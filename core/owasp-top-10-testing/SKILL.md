---
name: owasp-top-10-testing
description: Test an application against the OWASP Top 10 with any AI agent — the agent as pentest operator attempts real exploits for each category of the current OWASP Top 10:2025 (broken access control including SSRF, security misconfiguration, software supply chain failures, cryptographic failures, injection, insecure design, authentication failures, integrity failures, logging and alerting failures, mishandling of exceptional conditions) using the tools installed on the box, and reports only what it could actually prove, mapped back to the category with a replayable proof-of-concept. Also covers the OWASP API Security Top 10 (2023). Use when the user asks for an OWASP Top 10 assessment, OWASP compliance testing, or a security review mapped to OWASP categories.
license: Apache-2.0
---

# Test against the OWASP Top 10 (agent-driven)

The OWASP Top 10 is a taxonomy of risk categories, not a test suite — "OWASP Top 10 testing" means exercising each category against the real application and reporting what's actually exploitable. You do the exploitation yourself, using the tools on the box; this skill covers going category-by-category and reporting coverage honestly.

**Use the current edition: [OWASP Top 10:2025](https://owasp.org/Top10/)** (8th installment, superseding 2021). Ask the user before targeting an older edition — some compliance checklists still reference 2021, and a report labelled with the wrong edition is misleading. Key differences from 2021: **SSRF is folded into A01**, **A03 Software Supply Chain Failures** expands the old "Vulnerable and Outdated Components", and **A10 Mishandling of Exceptional Conditions** is new; A02 Security Misconfiguration moved 5→2.

## What is and is not testable

Be straight with the user about this — claiming a clean sweep of all ten is misleading.

| Category (2025) | Coverage | How to test |
|---|---|---|
| A01 Broken Access Control (incl. SSRF) | **Strong** | Two accounts (different tenants/privileges) + SSRF candidates; swap ids/tokens, retry admin routes as low-priv user, point URL-fetching features at internal hosts/canaries. |
| A02 Security Misconfiguration | **Strong** | DOM/grep for debug endpoints, verbose errors, permissive CORS, default/weak credentials, exposed admin surfaces, missing headers. |
| A03 Software Supply Chain Failures | **Partial** | Version fingerprinting (`whatweb`, headers), dependency review when source is supplied (`npm audit`/`pip audit`/`trivy fs`). Build/distribution compromise is out of scope for a runtime test — pair with SCA plus build-provenance controls. |
| A04 Cryptographic Failures | **Partial** | Transport config (TLS version/ciphers with `testssl.sh` or `nmap --script ssl-enum-ciphers`), plaintext tokens in responses, weak session ids. At-rest crypto and key management need source review (**find-security-vulnerabilities-in-code**). |
| A05 Injection | **Strong** | SQL/NoSQL/command/template injection and XSS payloads on every parameter (including JSON bodies and headers); `sqlmap --batch --risk=1 --threads=1` only on manually flagged params; verify each hit by replaying. |
| A06 Insecure Design | **Partial** | Business-logic abuse (price/quantity tampering, workflow skipping, race conditions via parallel replays) where reachable; design intent still needs human review and threat modelling. |
| A07 Authentication Failures | **Strong** | Enum/credential stuffing on login+forgot-password shapes, session/token handling, password-reset token flaws, JWT alg/exp/aud checks, MFA bypass attempts. |
| A08 Software or Data Integrity Failures | **Partial** | Insecure deserialization and unsigned-update paths where reachable; CI/CD trust boundaries are not runtime-testable. |
| A09 Security Logging & Alerting Failures | **Not testable from outside** | Requires reviewing the logging and alerting pipeline. State this rather than reporting it as passed. |
| A10 Mishandling of Exceptional Conditions | **Partial** | Probe error handling and fail-open behavior: malformed input, forced errors (truncated request, wrong content-length), race and timeout conditions; report what leaks or bypasses a control; exhaustive coverage of internal error paths needs source review. |

For APIs, run the same exercise against the **OWASP API Security Top 10 (2023)** — API1 BOLA, API3 object property level authorization, API5 function-level authorization — using the **api-security-testing** skill.

## Inputs that maximize coverage

Maximum category coverage comes from having both the source and a running instance, plus credentials at two privilege levels. Collect from the user, and note before you start:

- **Staging instance** (prefer over production) with **account A (user), account B (user, different tenant), and an admin account**. Without the second account, A01 results are structurally incomplete — say so in the report rather than leaving it implied.
- Repo/source for the white-box categories (A03 partial, A04, A06, A08, A10) — those cannot be fully assessed black-box.
- Out-of-scope list (billing, email sending, destructive admin actions).

An auditor-facing PDF is not produced here — the deliverable is the Markdown report below.

## Execute

Walk all ten categories in order, keeping per-category scratch notes. The category-specific testing detail lives in **web-app-penetration-testing** (live app) or **api-security-testing** (API) — follow it, but record findings mapped to their 2025 category id on the way. You may split the work into batches (categories A01–A05, then A06–A10) if context is tight; keep one scratch log so nothing is dropped. Optional broad sweep: `nuclei -u <base-url> -silent` mapped back to categories — leads only, verify each by replay.

## Report honestly

One Markdown file, `owasp-top10-report.md` (or user's choice): group findings by category and state, per category: what was attempted, what was proven, and what could not be assessed (A09 always; A03/A04/A06/A08/A10 partially). Label the report with the edition used (2025). Verify each PoC yourself before it goes to the user. An empty result for a category you could not fully test is not a pass — mark it "not assessed".

## Then fix and re-test

For retests, replay the original PoCs only in an environment and time window still authorized by the program; update the report with Fixed / Still-exploitable status. Use `triage-validation` and `report-writing` before submitting findings.
