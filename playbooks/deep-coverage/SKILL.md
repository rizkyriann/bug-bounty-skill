---
name: deep-coverage
description: Optional advanced bug-bounty follow-up. Offer only after the standard in-scope workflow has completed and coverage or results remain unsatisfactory; load and run only after the user accepts. Performs structured gap analysis and deeper, hypothesis-driven validation without expanding authorization or scope.
---

# Deep Coverage (Opt-In)

## Purpose

Provide a deliberate second-pass review when the ordinary bug-bounty workflow has completed but the user wants stronger or broader coverage. Apply senior-level security reasoning: reconstruct trust boundaries, challenge assumptions, inspect untested paths, reason about multi-step business logic, and look for safe-to-validate vulnerability chains. The goal is better coverage and evidence—not greater operational impact.

This is a specialized escalation skill, **not a default first-pass workflow**. Never load or execute it automatically at the beginning of an engagement.

## Activation gate

Offer this skill only when all of the following are true:

1. The task is an authorized bug-bounty engagement (or an explicitly authorized assessment whose rules permit the proposed checks).
2. The standard workflow has reached its normal stopping/reporting point, and the user has indicated that coverage or results are not yet satisfactory, or there are material gaps documented in the coverage notes.
3. The current written scope, rules of engagement, permitted accounts, test environment, and relevant limits are known.

The agent should briefly explain why a deeper pass may help and ask whether the user wants to proceed. **Wait for explicit acceptance.** If the user declines, finish with the current evidence and clearly state the coverage limitations. If authorization, scope, or restrictions are unclear, do not offer active testing; ask for clarification or perform only safe local/source analysis already authorized.

## Non-negotiable boundaries

- The original written authorization and scope still govern every action. Deep Coverage never means broader scope, stealth, evasion, persistence, or permission to test adjacent assets.
- Follow program rules, request limits, account restrictions, and third-party policies. Do not run broad/high-rate fuzzing or scanners by default; use only narrowly targeted checks that are permitted.
- Use only accounts and data supplied for testing or owned by the tester. Avoid other users' records; use the minimum controlled test data needed to establish impact.
- No denial of service, flooding, brute force, credential stuffing, destructive writes, persistence, lateral movement, or unnecessary data collection.
- Do not imitate a real threat actor's campaigns, infrastructure, tradecraft, or destructive operations. Apply advanced analytical rigor without adopting criminal operational behavior.
- Stop once evidence is sufficient. Do not intensify exploitation to increase a severity label.

## Deep-coverage workflow

### 1. Reconfirm the engagement boundary

Review the program/engagement scope and RoE, the standard-pass notes, and the user's stated concerns. Record allowed assets, excluded paths or techniques, test identities, request limits, and any prohibited proof methods. Reconfirm before touching a newly discovered asset or a technique with materially higher impact.

### 2. Build a gap map

Summarize what was actually covered, what was not covered, and why. Use a compact matrix such as:

| Surface / trust boundary | Covered checks | Evidence / result | Gap or next hypothesis | Risk / RoE constraint |
|---|---|---|---|---|
| Authentication and recovery | ... | ... | ... | ... |
| Roles, tenants, and object ownership | ... | ... | ... | ... |
| Sensitive business workflows | ... | ... | ... | ... |
| API versions and client/server boundaries | ... | ... | ... | ... |
| Input, file, and integration boundaries | ... | ... | ... | ... |

Do not mark an area tested solely because a scanner or checklist was run. Cite the request, source path, or observation supporting each coverage claim.

### 3. Rank a small number of hypotheses

Select a focused set of the highest-value gaps. Prioritize:

- authorization consistency across roles, tenants, object types, API versions, and related endpoints;
- state transitions and business invariants across multi-step workflows;
- differences between UI validation and server-side enforcement;
- trust boundaries between user-controlled input and sensitive sinks or integrations;
- sibling routes and alternate representations around an already observed anomaly;
- source-code paths, where available and authorized, that explain a reachable behavior.

For each hypothesis, write the expected secure behavior, the smallest safe test, expected evidence for and against, and the point at which testing stops. Prefer one-variable-at-a-time comparisons against a baseline. Load only relevant `hunt-*`, `vuln-*`, technology, or core guidance; do not repeat the entire library indiscriminately.

### 4. Test incrementally

- Start with passive review and existing engagement evidence, then use the lowest-impact active check permitted by the RoE.
- Use controlled accounts and resources to test boundaries. Keep request volume low and within explicit limits.
- Record timestamps, exact requests, relevant response differences, tool/fallback use, and negative results. Redact credentials and personal data.
- Treat scanner output, status-code changes, timing outliers, and isolated errors as leads only. Reproduce and inspect the actual response or state difference before claiming impact.
- If a check could modify real data, send messages, issue transactions, affect availability, or reach third-party systems, do not perform it unless the RoE expressly allows that exact action and safe test data/environment are available. Otherwise stop and ask.

### 5. High-impact finding stop gate

When a candidate could enable remote code execution, command execution, arbitrary file access, account takeover, sensitive-data exposure, destructive state changes, or service disruption:

1. Stop escalation immediately; do not pursue a shell, interactive session, persistence, credential access, filesystem exploration, data extraction, privilege escalation, or lateral movement.
2. Preserve only the minimum evidence already obtained and avoid repeating the action if it could have side effects.
3. Further validation is allowed only if necessary to distinguish a real issue from an anomaly, explicitly permitted by the RoE, and achievable with a harmless, bounded proof using a controlled resource. For suspected code execution, use a benign fixed marker or an approved canary only when the program explicitly permits it; do not run arbitrary commands or retrieve host data.
4. If a safe proof is unavailable, the RoE is ambiguous, or the first signal already establishes reachability, stop and report the candidate as validated to the degree demonstrated—or clearly label it unconfirmed. Ask the user/program owner before any stronger validation.
5. Do not access other users' data or attempt to maximize impact. Describe potential impact as potential, not as observed fact.

### 6. Validate and report

Run `core/triage-validation`, `core/evidence-hygiene`, and `core/report-writing` as appropriate. Present:

- confirmed findings separately from hypotheses and negative checks;
- exact minimal reproduction steps and redacted evidence;
- demonstrated impact separately from plausible worst-case impact;
- severity rationale grounded in the program's rubric;
- remediation, coverage completed, coverage gaps, and explicit stop points;
- any checks intentionally not performed because they could exceed the RoE or cause harm.

Do not describe an untested area as secure, and do not inflate coverage based on the number of tools or payloads used.
