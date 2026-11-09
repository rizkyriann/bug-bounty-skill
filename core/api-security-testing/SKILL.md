---
name: api-security-testing
description: Security-test a REST, GraphQL, or gRPC API with any AI agent — enumerate endpoints from an OpenAPI/GraphQL schema (or by crawling) using common Kali-style pentest tools (curl, ffuf, httpx, arjun, nuclei, sqlmap), then actually exploit the API-specific vulnerability classes in the OWASP API Security Top 10 (2023) — broken object-level authorization (BOLA/IDOR), broken object property level authorization (excessive data exposure and mass assignment), broken function-level authorization, unrestricted resource consumption, SSRF, injection, and auth/token flaws. Every finding must be validated with a working, replayable proof-of-concept request and collected into a single Markdown report. Use when the user asks to pentest, security-test, audit, or find vulnerabilities in an API, endpoint, or backend service.
license: Apache-2.0
---

# Security-test an API (agent-driven, tool-agnostic)

You (the AI agent) are the pentest operator. There is no external scan engine: you reason about the target, drive the tools yourself through bash, verify every finding by replaying requests, and write the report. Use any tool that is installed — most are common on Kali Linux (`curl`, `httpx`, `ffuf`, `dirsearch`, `arjun`, `nuclei`, `sqlmap`, `wafw00f`, `jq`, `hydra`, `ffuf`, `postman-to-openapi`). If a tool is missing, either do the same job with `curl`/`python` one-liners or install it if the user allows (e.g. `apt`, `go install`, `pipx`).

This workflow targets API specifics: no rendered surface to crawl, bugs that are authorization-shaped rather than injection-shaped, and behavior that changes per token. The coverage checklist is the current [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/). For full web apps (rendered UIs, XSS, click-jacking, TNAS), use the web-app skill instead.

## 1. Gather inputs before testing

APIs are near-impossible to test blind. Collect first, and ask the user for anything missing — never fabricate tokens and never test a target the user does not own or is not authorized to test.

| Input | Why it matters |
|---|---|
| **Schema** — OpenAPI/Swagger file, Postman collection, GraphQL endpoint (with introspection on if available), or a gRPC `.proto` | Turns guesswork into full endpoint enumeration. Biggest single win in coverage. |
| **Two sets of credentials/tokens**, ideally in different tenants | BOLA/IDOR (API1:2023, still the #1 API risk) can only be *proven* by accessing tenant A's objects with tenant B's token. |
| **A low-privilege and a high-privilege token** | Required to prove broken function-level authorization (API5:2023 — a `user` calling admin-only routes). |
| **Example object IDs** (user ids, order ids, org ids) | Lets you test ID tampering immediately instead of hunting for valid identifiers. |
| **Out-of-scope routes** | Payments, mass notification, destructive admin endpoints — skip those, and state the exclusion in the report. |
| **Known rate limits / WAF in front** | Adapt request pacing and avoid burning time on throttled probes; if a WAF exists, check `wafw00f <base-url>` and note bypass attempts separately from findings. |

Get the user's explicit authorization for the target and save tokens in a scratch file (e.g. `.env` or `tokens.txt`, git-ignored) rather than pasting them into every command; avoid relying on shell history for secrets.

## 2. Enumerate the attack surface

Work from schema to surface, and keep everything you learn in a scratch file (e.g. `endpoints.md`) — you'll cite it in the report.

- **OpenAPI/Swagger present:** parse it directly. With `yq`/`jq`: `jq -r '.paths | keys[]' openapi.yaml` (convert YAML to JSON with `yq -o=json` first if needed). Extract method + path + parameters + required auth into your endpoint list. If only a Postman collection exists, `postman-to-openapi` can convert it, or read `item[].request` structure directly with `jq`.
- **No schema:** reconstruct the surface.
  - Passive: fetch the base URL and any JS bundles, grep for `fetch(`/`axios` paths, `/api/`, `/v1/`, `swagger`, `openapi.json`, `/graphql`, `.proto`.
  - Active: `ffuf -w <wordlist> -u https://api.target.com/FUZZ -mc 200,301,401,403 -H "Authorization: Bearer $TOKEN"` against common dirs (`swagger-ui`, `api-docs`, `openapi.json`, `graphql`, `actuator`, `debug`, `admin`, ...); `arjun -u <url> --headers "Authorization: Bearer $TOKEN"` for hidden query/JSON parameters; directory discovery with `dirsearch` if preferred.
- **GraphQL:** probe `POST /graphql` with an introspection query:
  ```bash
  curl -s https://api.target.com/graphql -H 'Content-Type: application/json' \
    -d '{"query":"{ __schema { types { name } } }"}'
  ```
  If introspection is blocked, try field-suggestion probing (send a wrong field; the error often leaks the real one) and classic GET `/graphql?query=...`. Dump the full schema when introspection works and enumerate mutations/queries + their argument types — that is your endpoint list.
- **gRPC:** target the endpoint; reflect on the `.proto` the user gave you (grpcurl with `-proto`/`-plaintext` if installed: `grpcurl -proto service.proto -plaintext host:port list`).
- **Baseline behavior:** with a valid token, sample a few GETs and note status codes, header shape (`X-RateLimit-*`, `WWW-Authenticate`, verbose error bodies), and response time. Record API versioning (`/v1` vs `/v2`) — old versions often skip new auth checks.

## 3. Test, one attack class at a time

Keep all generated requests in the report as **exact, replayable `curl` commands**. Never paraphrase a request; a finding the agent cannot replay is unproven. Attack sequence:

**BOLA / IDOR (API1) — the highest-value class.** With tenant B's token (or a low-priv token), request tenant A's objects: swap ids/tokens in `GET /orders/501`, `GET /users/11`, etc. Check status *and* body — an empty `200`, a `200` with different fields, or a `403` that leaks existence all count differently. Try id variants: uuid→numeric, sequential ids, email-as-id, case/anagram variations, and parameters like `?user_id=`/`?account_id=` in otherwise-safe endpoints.

**Broken authentication (API2) & token flaws.** Test: JWT alg confusion (`alg:none`, HS256 with the exposed public key as HMAC secret), missing `exp`/issuer validation, signature stripping (remove the third JWT segment), refresh-token reuse across devices, weak api-key entropy, auth on `/logout` only, credential endpoints accepting enumeration (different message or latency for valid vs invalid users). Tools: `jwt-tool` or a local `python -c` decode; verify server behavior with `curl` replays for each tampered token.

**Broken object property level authorization (API3).** Mass assignment: replay `PATCH/POST` with added privilege fields (`"role":"admin"`, `"isAdmin":true`, `"tenantId":…`) and see if the server accepts/ignores/honors them; then re-read the object to see the effect. Over-exposure: compare the list endpoint's fields against the detail endpoint's — lists that include role/email/api-key fields are findings. Contrast high-priv vs low-priv reads of the same object.

**Unrestricted resource consumption (API4).** Carefully and with the user's consent: extremely large page sizes (`?limit=100000`), deep GraphQL nesting/aliases in one request, oversized JSON payloads, slow-but-cheap expensive filters. Do not DoS; a single 10x-limit request that returns 500k rows is a finding, a flood is not. Also test missing pagination in list endpoints.

**Broken function-level authorization (API5).** Call admin/logout-elevated routes with low-priv tokens; try every verb on discovered admin endpoints, and try admin verbs on user endpoints (`POST /users/11/delete` vs `DELETE /users/11`); some frameworks only guard the decorated route.

**Server-side request forgery (API8).** Where the API fetches URLs (webhooks, importers, avatars, "test connection" features), point them at a listener you control or a safe canary (`http://127.0.0.1:PORT`, internal hostnames, metadata endpoints like `169.254.169.254` for cloud targets). SSRF is proven by internal-only responses, timing deltas, or canary callbacks.

**Security misconfiguration (API9).** Enumerate HTTP methods per endpoint (`curl -X OPTIONS`, and TRACE/PATCH/PUT probes); check CORS wildcards (`Origin: evil.com`), missing TLS enforcement (plain HTTP still serving), verbose stack traces on bad input, default/management endpoints (`/swagger`, `/vault`, `/api/admin/docs`), and permissive auth-skip headers (`X-Original-URL`, `X-Forwarded-For`-based allowlists).

**Injection (API6/API7/API10 — SQLi, NoSQLi, unsafe consumption).** Inject markers into every parameter type: `'`, `"`, `1=1`, `{"$gt":""}` (NoSQL), `{{7*7}}` (SSTI), `;sleep 5;` (timing). Use sqlmap only against a test URL with `--batch` and a limited risk/level after you've confirmed injection manually — never let an automated fuzzer rampage over out-of-scope endpoints. Prove with a data-dependent response difference you can show.

**Automated checks:** once the surface is mapped, optionally run `nuclei -u <base-url> -t api/ && nuclei -t exposures/ -silent` as a broad sweep for known misconfig/exposure patterns; treat its output as leads to verify by hand, never as a finding until you replay the request yourself and see the impact.

**Playbook depth:** for API-shaped classes needing deeper technique coverage (schema exposure, mass assignment, JWT confusion, WebSocket/gRPC quirks), load the matching playbook from this project's `../playbooks/` — e.g. `vuln-api-schema-exposure`, `vuln-mass-assignment`, `vuln-jwt`, `vuln-websocket`, `vuln-grpc` (see **penetration-testing** § Companion skill library).

## 4. Verify findings before reporting

For every candidate finding:

1. **Replay the exact request** (in a clean shell, no leftovers) and capture the full response (status, headers, body).
2. **Confirm impact, not just anomaly**: for authorization findings, check the response really contains the *other tenant's* / *elevated* data — not an empty 200 or your own echoed input.
3. **Minimize the PoC**: strip irrelevant headers, reduce to the smallest request that still proves it; that's what goes in the report.
4. **Rate the severity** (critical/high/medium/low/info) with one-sentence justification tied to impact (data breach? privilege escalation? DoS potential? information leak?).
5. Re-check each finding again after any target-side change — a stale finding is worse than none.

## 5. Write the report

One file, `pentest-report.md` (or a filename the user chooses), in the working directory. Structure:

```markdown
# API Penetration Test Report — <target / service name>

- Date: <date>
- Tester: <agent name> (assisted)
- Scope: <endpoints/domains tested> — Out of scope: <list>
- Authorization: confirmed by user on <date>
- Overall risk: <critical|high|medium|low|info + count table>

## Executive summary
<3–6 sentences: what was tested, how the surface was enumerated, headline findings>

## Findings

### [CRITICAL] BOLA — other tenant's orders readable with tenant B token
- OWASP API: API1:2023 — Broken Object Level Authorization
- Evidence:
  ```http
  GET /orders/501 HTTP/1.1
  Host: api.target.com
  Authorization: Bearer <REDACTED_TOKEN_B>
  ```
- Response (excerpt): HTTP 200 — body includes order id 501, tenant org 1111, <data>
- Impact: one sentence
- Remediation: one-three sentences (fix the authz check, don't patch one endpoint)

### [HIGH] <title>
...(repeat per finding)

## Coverage notes
<which OWASP API Top 10 categories were tested and how; anything not tested and why>
## Appendix
<tools used, endpoint list source (schema or fuzzing), requests kept for re-test>
```

Include the *exact* replayable command (with `curl`, token placeholder) per finding — the point is that a human or another agent can re-run it verbatim. Then redact real tokens in the report (`<REDACTED>`) but keep the raw replayable variant in a local `poc-requests.txt` if the user wants it.

## 6. Retest and keep it tested

After fixes ship, re-run the *same* PoC list against the fixed target and mark each finding Fixed / Still-exploitable in a short addition to the report. Suggest wiring a subset of the PoCs into CI (a small script that replays the requests and diffs responses) so new endpoints get checked as they ship — offer to write that script if the user wants it.

## Safety

Only test targets the user owns or is explicitly authorized to test; ask when the target looks like third-party infrastructure. Never destroy data, flood targets, or buy/spend anything. Keep credentials out of reports, shell history, and committed files.
