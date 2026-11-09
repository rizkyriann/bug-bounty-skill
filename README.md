# bug-bounty-skill

Agent-neutral skills for authorized bug bounty work, primarily web and API targets. The AI agent is the operator; tools such as `curl`, `ffuf`, `nuclei`, and `sqlmap` are optional executors. No external scan engine is required.

This directory is self-contained. Curated skill copies and required supporting files are included here; the source collection is not needed at runtime.

## Structure

- **`core/`** — 10 skills: `penetration-testing`, `web-app-penetration-testing`, `api-security-testing`, `owasp-top-10-testing`, `find-security-vulnerabilities-in-code`, `bb-methodology`, `recon-scope-triage`, `triage-validation`, `evidence-hygiene`, and `report-writing`. Start with `bb-methodology` for the bug-bounty workflow, use `recon-scope-triage` before testing discovered assets, and use `triage-validation` + `evidence-hygiene` + `report-writing` before submission. Load `web-app-penetration-testing` or `api-security-testing` for the target type.
- **`playbooks/`** — 128 focused playbooks: 86 original recon/vulnerability/tool/payload skills plus 42 additional web/API recon and `hunt-*` playbooks. Load only the playbooks relevant to observed technology and in-scope features.
- **`optional/`** — 43 skills for specialized targets, platform-specific workflows, and alternative orchestrators. Use only when the written program scope justifies them. `bug-bounty` and `bb-local-toolkit` are alternative full workflows; do not run them as simultaneous orchestrators with `bb-methodology`.
- **`TOOLS.md`** — tool discovery, installation and curl/Python fallbacks.
- **`USAGE.md`** — installation guidance and prompt templates.
- **`PROMPT-TEMPLATE.txt`** — ready-to-fill English prompt for an authorized HackerOne assessment.

## Report convention

Deliver one Markdown report per engagement (`pentest-report.md` or the program's required format), with findings ranked by demonstrated impact, an exact replayable PoC, response evidence, remediation, tools/fallbacks used, and honest coverage notes. Use the program's requested severity/CVSS version; `playbooks/pentest-report` provides a CVSS-based structure, while `core/report-writing` tailors the narrative to the submission platform. Redact live tokens and unnecessary personal data with `core/evidence-hygiene`.

## Safety and scope

- Test only assets explicitly listed in a program's written scope and follow its rules of engagement.
- Verify that discovered assets belong to the program before testing them; keyword matches and recon-tool output do not establish ownership.
- No flooding, denial-of-service, destructive changes, brute-force attempts, real-user account access, or unnecessary collection of user data. Stop at the minimum non-destructive proof.
- Do not test cloud, identity, mobile, network, or other specialized targets unless that asset class is explicitly in scope.
- Keep credentials out of reports, shell history, and committed files. Redact tokens/cookies and minimize PII in evidence.
- The optional `offensive-osint` dashboard is an unauthenticated local helper with a file-scan endpoint: keep it bound to loopback; never expose it to a LAN/network. The secret scanner prints matched values, so keep its output private.
- These rules take precedence over conflicting examples in any individual playbook.

## Curated additions

- **Core additions (5):** `bb-methodology`, `recon-scope-triage`, `triage-validation`, `evidence-hygiene`, `report-writing`.
- **Playbook additions (42):** `web2-recon`, `osint-methodology`, and 40 `hunt-*` skills for common web/API classes.
- **Optional additions (31):** `bb-local-toolkit`, `bug-bounty`, `bugcrowd-reporting`, `security-arsenal`, `supply-chain-attack-recon`, `offensive-osint`, 16 specialized `hunt-*` skills, and 9 target-specific cloud/mobile/identity/Web3 skills.

The original 86 playbooks and 12 optional skills remain part of the set. `hunt-dispatch` was not imported because it depends on a `/hunt` command provided by another host. `bug-bounty` and `bb-local-toolkit` are retained as optional alternatives, not additional default routers.
