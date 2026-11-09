# `osint-methodology` skill

The "how to think" reference for external red-team OSINT and bug-bounty reconnaissance.

| Field | Value |
|---|---|
| Name | `osint-methodology` |
| Version | 2.1 |
| Lines | ~1,700 |
| Top-level sections | 33 |
| Subsections | ~125 |
| Companion skill | [`offensive-osint`](../../optional/offensive-osint/) |

## When this skill triggers

Auto-triggers on prompts containing any of ~55 trigger phrases. Common ones:

- `external recon`, `external red team`, `bug bounty recon`, `attack surface management`, `ASM`, `perimeter recon`
- `OSINT methodology`, `recon methodology`, `target reconnaissance`, `asset discovery`, `attack path`
- `identity fabric`, `SSO discovery`, `IdP fingerprinting`, `M365 enumeration`
- `phishing infrastructure`, `pretext development`, `bug bounty submission`, `responsible disclosure`
- `client report`, `exec summary`, `risk translation`
- `confidence upgrade`, `time budget`, `engagement profile`, `asset triage`
- `detection-aware probing`, `back-off strategy`, `persona rotation`
- `WAF bypass`, `CDN bypass`, `origin discovery`
- `vulnerability prioritization`, `CVE prioritization`, `EPSS`, `CISA KEV`
- `threat actor investigation`, `attribution`

Full trigger list in the SKILL.md frontmatter.

## What's in it

See `SKILL.md` for the complete methodology and the linked companion playbook for operational techniques.

Highlights:

- **§7 — 5-stage recon pipeline** + priority order + time budgeting (1h / 4h / 1d / 1w profiles)
- **§8 — Asset graph discipline** with 29 typed asset types + 23 typed edges + per-asset-type triage
- **§9 — Findings rubric** anchored on examples (CRITICAL → INFO + escalation rules)
- **§11 — Identity fabric mapping** (Entra, Okta, ADFS, Google, SAML, AWS, M365 deep)
- **§22 — Breach × identity correlation** (HudsonRock + HIBP + DeHashed + IntelX → SSO_EXPOSURE finding)
- **§27 — WAF/CDN bypass + origin discovery** (8 techniques)
- **§28 — Vulnerability prioritization** (CVE × EPSS × KEV × Metasploit rubric)
- **§29 — Phishing infrastructure & pretext development**
- **§30 — Bug bounty submission templates** (HackerOne, Bugcrowd, Intigriti, etc.)
- **§31 — Client deliverable templates** (exec summary + risk translation matrix + reporting cadence)

## Loading

Use `USAGE.md` at the project root for agent-specific installation instructions. Keep this skill's `SKILL.md` with its companion `offensive-osint` skill when using OSINT-specific workflows.

This bundled `SKILL.md` is the content to load. No external documentation or source repository is required at runtime.

## Self-test

No automated self-test suite is included in this curated bundle; check the skill's scope and tool-availability guidance before use.

## License

The source collection identifies this skill as MIT-licensed; preserve its attribution when redistributing.
