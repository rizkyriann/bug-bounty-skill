# USAGE.md — installing and driving the skills with an AI agent

Everything here is Markdown + frontmatter, so "installing" means copying folders to the agent's skill-discovery path. What the agent needs: **shell/tool-use capability** (the skills execute real commands) and a model strong at multi-step autonomous work.

## 1. Install per agent

All paths below are the conventional skill dirs for that agent; create them if missing.

| Agent | Skill path | Notes |
|---|---|---|
| Claude Code (desktop/CLI) | `~/.claude/skills/<skill-name>/SKILL.md` (or project `.claude/skills/`) | Or package as a plugin (`plugin.json`). Skills ≥ 1.0 auto-trigger by description match. |
| opencode | `~/.opencode/skill/<name>/` (or `~/.config/opencode/skill/<name>/`) | Same trigger mechanism (frontmatter `description`). |
| Codex CLI / "omo codex" | `~/.codex/skills/<name>/` or wherever your AGENTS.md-based loader expects it | Skills work as plain playbooks; if the loader has no inventory feature, reference the folder in `AGENTS.md`. |
| Gemini CLI / Antigravity | the agent's extensions/skills dir (e.g. `~/.gemini/skills/<name>/`) — consult current docs | Same frontmatter convention. |
| Any generic agent (AGENTS.md/CLAUDE.md loader) | keep the repo anywhere and point the agent at it | Add one line to `AGENTS.md`: `Security skills live in /path/to/bug-bounty-skill — read core/<skill>/SKILL.md and follow it when the user asks for pentest work.` |

**Layered install (recommended — the full set is large):**

```bash
# copy the 10 core skills first
cp -r bug-bounty-skill/core/*  <agent-skills-dir>/
# then the playbook families you actually need:
cp -r bug-bounty-skill/playbooks/pentest-*  bug-bounty-skill/playbooks/vuln-*  <agent-skills-dir>/
# optional on demand:
cp -r bug-bounty-skill/optional/<skill> <agent-skills-dir>/
```

(Windows: `xcopy /E /I` or `Copy-Item -Recurse` works the same way; or run the copies inside WSL if skills live there.)

Also make `TOOLS.md` available to the agent (keep the project directory intact, or copy it into the agent's workspace) so the tool fallback chain works. The curated bundle is self-contained; its original source collection is not needed at runtime.

## 2. What to say (prompt templates)

Skills auto-trigger on intent words (`pentest`, `security test`, `audit`, `vulnerabilities`, `bug bounty`). Always include **target + scope + authorization + constraints + output location** — the core skills deliberately refuse or ask first if authorization is unclear.

**A. Full engagement (has test accounts, authorized):**

```text
Security-test https://staging.example.com — bug bounty engagement (HackerOne program #1234, in-scope: *.example.com, allowed env: staging).
Test accounts: qa-a@example.com / <pass> (org 1111), qa-b@example.com / <pass> (org 2222).
Out of scope: /billing/*, /uploads/*. No request flooding, no real email to users.
Use bug-bounty skill set: start from bb-methodology, confirm ownership/scope with recon-scope-triage, then use penetration-testing and the relevant web-app/API skill plus `hunt-*` or `vuln-*` playbooks. Before submitting, run triage-validation, evidence-hygiene, and report-writing.
Deliverables: pentest-report.md + poc-requests.txt in this folder (report format from pentest-report playbook, CVSS).
```

**B. Recon-only first pass (slow, no exploitation):**

```text
Bug bounty recon for *.example.com (in-scope per program #1234). Recon phase only — no exploitation.
First use recon-scope-triage to verify asset ownership. Then use pentest-recon + web2-recon + relevant recon-* playbooks (subdomain enum, JS analysis, secrets, favicon, dorking; skip Shodan if keyless).
Output: recon-notes.md with live hosts, tech stack, endpoints found, and a suggested shortlist for the next phase.
```

**C. Class-focused follow-up:**

```text
Focused test on https://api.example.com (authorized): access-control classes only — vuln-idor, vuln-bfla, vuln-mass-assignment.
Two accounts provided below; run full IDOR matrix (A→B, B→A) and mass-assignment probes. Report findings + replayable PoC.
```

**D. Source-published program:**

```text
Program publishes the app source at <repo>. Review with find-security-vulnerabilities-in-code focusing on the auth layer; escalate nothing to live testing beyond QA environment <url>.
```

## 3. Example session (what to expect)

```text
YOU:      [Template A above]
AGENT:    Starts with bb-methodology and recon-scope-triage. Asks / derives:
          - RoE check: in-scope list present ✓
          - missing: nothing critical; begins.
AGENT:    Recon (reads playbooks/recon-subdomain + recon-js-analysis + pentest-recon):
          > crt.sh, JS bundle harvest, ffuf content discovery → endpoints.md / recon-notes.md
AGENT:    Enumerate & test class by class (dispatches vuln-* playbooks):
          > vuln-jwt: alg=none replay → 200 with admin data  → candidate
          > vuln-idor: matrix A↔B over /v1/invoices/:id → other-tenant body → candidate
          > vuln-xss: reflection check on /search?q=        → candidate
          > nuclei broad sweep → leads only, re-verified by hand
AGENT:    Validation (triage-validation + vuln-exploit-validation):
          > re-run minimal request in clean shell, confirm impact, minimize PoC
AGENT:    Writes:
          - pentest-report.md  (severity table, [EACH FINDING] + curl PoC + impact + remediation)
          - poc-requests.txt   (exact replay requests, tokens as <REDACTED>)
          - evidence scrubbed via evidence-hygiene; report edited via report-writing
          - appendix: tools used, fallbacks used if TOOLS.md chain was triggered, coverage notes
YOU:      Read the report → copy each finding into the platform's report form (H1/Bugcrowd),
          attaching the curl command + response excerpt.
```

Timeline expectation: recon+test on a normal web target runs within a single session (tens of minutes to a few hours); large-surface programs split into sessions (recon first, classes next), and re-runs pick up from the notes.

## 4. Practical guardrails

- One engagement per workspace/folder; keep `poc-requests.txt` and note files out of screenshots when publishing.
- If the agent proposes something outside the stated scope (e.g. brute-forcing the login of an out-of-scope host), refuse and re-anchor the scope in a new prompt.
- Missing tools: agent resolves via `TOOLS.md` chain (check → curl/python fallback → install with your OK → disclose weaker evidence). Approve installs consciously.
- Custom skills edits: after you modify a SKILL.md, re-point the agent at the folder (agents cache skill inventories per session).
- Platform-specific extras: the oMoMo `/mode` slash-commands only exist with the oMoMo plugin host; in other agents state the mode in text (e.g. "bug bounty mode").
