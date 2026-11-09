# Tool availability & fallbacks

Playbooks in `playbooks/` name real Kali-family tools in their commands. The agent must never assume a tool exists — always resolve it in this order:

1. **Check** `command -v <tool>` (Windows: `where <tool>` / `Get-Command <tool>`).
2. **Fallback to curl/python** — for most web/API tests the equivalent is ~10 lines of script (see table).
3. **Install with the user's permission** — commands per OS below. Never install silently into bug bounty engagements; state what you are adding.
4. If nothing is feasible (locked-down box), say so in the report's *Methodology* section — do not fake coverage.

## Install cheatsheet per OS

| Pattern | Kali/Debian | macOS (brew) | Windows native | Cross-platform binary |
|---|---|---|---|---|
| nmap | `apt install nmap` | `brew install nmap` | nmap.org installer | — |
| ffuf / dirsearch | `apt install ffuf dirsearch` | `brew install ffuf` | Go release binary | `go install github.com/ffuf/ffuf/v2@latest` ; dirsearch via `pipx install dirsearch` |
| nuclei | `apt install nuclei` (or release) | `brew install nuclei` | release binary | `go install github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest` |
| httpx (projectdiscovery) | release binary / `apt` on newer | `brew install httpx` | release binary | `go install github.com/projectdiscovery/httpx/cmd/httpx@latest` |
| subfinder / amass | `apt install subfinder amass` | `brew install subfinder amass` | release binary | `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` |
| arjun | `pipx install arjun` | `pipx install arjun` | `pipx install arjun` | — |
| wafw00f | `pipx install wafw00f` | `pipx install wafw00f` | `pipx install wafw00f` | — |
| whatweb | `apt install whatweb` | `brew install whatweb` | ruby gem | `gem install whatweb` |
| gobuster | `apt install gobuster` | `brew install gobuster` | release binary | `go install github.com/OJ/gobuster/v3@latest` |
| sqlmap | `apt install sqlmap` | `brew install sqlmap` | `pipx install sqlmap` | — |
| dalfox | release binary | `brew install dalfox` | release binary | `go install github.com/hahwul/dalfox/v2@latest` |
| hydra / john / hashcat | `apt install hydra john hashcat` | `brew install hydra john` | release binaries | — |
| semgrep | `pipx install semgrep` | `pipx install semgrep` | `pipx install semgrep` | — |
| jq | `apt install jq` | `brew install jq` | winget/choco | — |
| interactsh (OOB) | client binary (projectdiscovery) | brew | release binary | `go install github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest` — or use a public canary (webhook.site/requestbin) per program rules |

Python3 + `curl` are assumed present everywhere (Windows 10+ ships curl). `jq` failures: use `python -m json.tool` or a tiny `python -c` filter.

## Fallback mapping (tool → curl/python equivalent)

| Tool | Typically used for | Fallback without it |
|---|---|---|
| ffuf / gobuster / dirsearch | dir/content discovery | `curl -s -o /dev/null -w "%{http_code} %{size_download}\n" "$URL/$path"` in a shell loop over a wordlist; watch status + body-size deltas vs baseline |
| nuclei | template scanning | scan already done by hand per playbook; quick list: `for u in /.env /.git/config /swagger.json /openapi.json /actuator/env; do curl -s -o /dev/null -w "%{http_code} $u\n" "$BASE$u"; done` — treat 200s as *leads only*, verify manually |
| httpx | liveness/tech headers | `curl -sI` per host; extract `Server`, `X-Powered-By` |
| subfinder/amass | subdomain enum | `curl "https://crt.sh/?q=%25.example.com&output=json"` → parse `name_value`; plus DNS: `host`/`nslookup`/`dig` + brute via `/etc/hosts`-safe resolver loop |
| arjun | hidden parameters | `for p in id user admin debug page sort; do curl -s -d "$p=1" ... | cmp with baseline` (GET and POST variants) |
| wafw00f | WAF fingerprint | inspect `Server` header, cookie names (`__cfduid`, `X-Akamai-*`, `incap_ses`), generic 403/blocked-body on an obvious probe |
| sqlmap | SQLi automation | manual payloads from `payload-sqli`: error-based markers, boolean diff, `SLEEP(5)` timing via `curl -w "%{time_total}"` — *only* after manual confirmation |
| dalfox | XSS parameter scanning | manual payloads from `payload-xss` + `curl` reflection check (compare response HTML for unescaped input) |
| hydra/john/hashcat | credential cracking | *skip* — credential stuffing/cracking is usually out of RoE in BB; prefer logic flaws in auth workflow skills |
| interactsh-client | OOB detection | use the platform's own canary (many BB programs provide one) or webhook.site; DNS-based checks via `nslookup -type=txt $(uuid).canary.example` style callbacks if enabled |
| ctf/pwn debugging toolships | n/a in BB | out of scope for this curated set |

## Windows-native notes

- `curl.exe` and PowerShell's `Invoke-WebRequest/Invoke-RestMethod` cover 90% of playbook techniques.
- Go-release binaries are self-contained — download from GitHub releases into a `bin/` dir on PATH.
- For the full Kali toolset, prefer **WSL (Kali/Ubuntu)** in this homelab; everything above installs by `apt`.

## Discipline

- Log in the report's Appendix which tools ran, which were missing, and which fallbacks replaced them — a fallback-covered test is *weaker evidence*, and the coverage notes must say so.
- Playbooks marked "Tool availability" know about this file; the router (**core/penetration-testing**) triggers the check before any playbook runs.
