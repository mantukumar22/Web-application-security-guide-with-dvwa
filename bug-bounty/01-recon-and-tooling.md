# Recon & Tooling

## 1. Passive Recon (no direct interaction with the target's servers beyond normal traffic)

```bash
# Subdomain enumeration
subfinder -d target.com -o subs.txt
amass enum -passive -d target.com

# Certificate transparency logs
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u

# Wayback / historical URLs (finds old, forgotten endpoints)
waybackurls target.com | tee wayback.txt

# Google/Shodan dorking (info-gathering only — respect ToS)
site:target.com inurl:admin
site:target.com filetype:pdf
```

## 2. Active Recon (only within authorized scope, per program RoE)

```bash
# Live host check
cat subs.txt | httpx -silent -o live.txt

# Port scanning (only if explicitly permitted by program scope)
nmap -sV -T3 -p- target.com

# Tech stack fingerprinting
whatweb target.com
wappalyzer-cli target.com

# Directory/endpoint discovery
ffuf -u https://target.com/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403

# JS file analysis for hidden endpoints/API keys
katana -u https://target.com -jc | grep -E "\.js$" > jsfiles.txt
for f in $(cat jsfiles.txt); do curl -s $f | grep -oE "\"(/[a-zA-Z0-9_/-]+)\"" ; done
```

## 3. Core Manual Toolchain

| Tool | Purpose |
|---|---|
| **Burp Suite** | Intercepting proxy, Repeater (manual request tampering), Intruder (fuzzing), Extensions (e.g. Turbo Intruder for race conditions) |
| **sqlmap** | Automated SQLi detection/exploitation |
| **ffuf / feroxbuster** | Directory & parameter fuzzing |
| **Nuclei** | Template-based known-vulnerability scanning |
| **Postman/Insomnia** | API testing, especially for REST/GraphQL |
| **jwt_tool** | JWT manipulation (alg confusion, weak secret brute force) |
| **Autorize (Burp extension)** | Automated authorization/IDOR testing across roles |

## 4. Systematic Parameter Testing Checklist

For every input point discovered (query params, JSON body fields, headers, cookies, file uploads, GraphQL variables):

- [ ] Try classic SQLi probes (`'`, `"`, `1=1`, `SLEEP(5)`)
- [ ] Try XSS probes (`<script>`, `<svg onload=...>`, context-specific breakouts)
- [ ] Try path traversal (`../../../../etc/passwd`, encoded variants `%2e%2e%2f`)
- [ ] Try command injection metacharacters (`;`, `|`, `` ` ``, `$()`)
- [ ] Test IDOR — change any ID/reference to another user's resource (see below)
- [ ] Test for SSRF in any parameter that accepts a URL
- [ ] Test HTTP method tampering (does `GET` work where only `POST` should?)
- [ ] Test for mass assignment (extra JSON fields like `"isAdmin": true` in a profile update)
- [ ] Check for verbose errors leaking stack traces/paths/versions

## 5. IDOR (Insecure Direct Object Reference) — Not a DVWA Module, But Essential

The single most common bug bounty finding class. Test by simply **changing an ID and checking if you get someone else's data**:

```bash
# As user A, request user B's resource
curl -H "Authorization: Bearer <userA_token>" https://target.com/api/orders/1002
# If 1002 belongs to user B and this returns their data → IDOR
```

Test with:
- Sequential IDs (increment/decrement)
- UUIDs (still test — sometimes predictable or leaked elsewhere in the app)
- Different HTTP methods (`GET` might be protected, `PUT`/`DELETE` on the same resource might not be)

## 6. Rate-Limiting & Race Condition Testing

```bash
# Burp's "Turbo Intruder" or simple parallel curl to test race conditions
# (e.g. redeeming a discount code twice, double-spending a wallet balance)
for i in {1..20}; do curl -s -X POST https://target.com/api/redeem -d "code=DISCOUNT10" & done; wait
```

## 7. Keep Detailed Notes

Use a structured note-taking tool (Obsidian, Notion, or plain markdown per-target) to log:
- Every endpoint discovered and its auth requirements
- Every parameter tested and result
- Screenshots/request-response pairs for anything suspicious, even if not yet confirmed

This becomes the raw material for your report (`02-reporting-and-triage.md`).
