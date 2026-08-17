# Reporting & Triage

A great finding with a bad report gets ignored, downgraded, or duplicated against. A good report is what actually gets paid and fixed.

## 1. Report Structure That Triagers Actually Want

```markdown
# Title
[Vulnerability Class] in [Component/Endpoint] leads to [Impact]
e.g. "Reflected XSS in /search?q= leads to Session Hijacking (Account Takeover)"

## Summary
One or two sentences: what's broken, and what's the worst realistic outcome.

## Severity
Your assessed CVSS score/vector, and plain-language justification.

## Steps to Reproduce
1. Numbered, exact steps — assume the reader has zero context
2. Include exact URLs, exact parameters, exact payloads
3. Include authentication context needed (which role/account)

## Proof of Concept
- Screenshot(s) or screen recording
- Raw HTTP request/response (from Burp) — this is often more useful than a screenshot
- A working curl command if applicable, so the triager can reproduce in 30 seconds

## Impact
Concretely explain what an attacker gains: data exposed, actions performed,
users affected. Avoid "this could potentially maybe" — either demonstrate impact
or clearly label it as theoretical.

## Suggested Fix
Point to the specific class of fix (e.g. "use parameterized queries," "add
output encoding via htmlspecialchars," "validate redirect destination against
an allowlist") — shows you understand the root cause, not just the payload.
```

## 2. Severity Justification Example (using this repo's chapters)

> **Title:** SQL Injection in `/vulnerabilities/sqli/?id=` leads to Full Database Compromise
> **CVSS 3.1:** 9.8 Critical — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
> **Justification:** Unauthenticated, network-exploitable, no user interaction required, complete read/write access to the `users` table including password hashes was demonstrated (see PoC).

## 3. Writing a Reproducible PoC

- Prefer a **raw HTTP request** (copy from Burp) over a description — the triager can paste it straight into Repeater.
- If you used `sqlmap`/automated tooling, still provide **one manual, minimal reproduction** — triagers are often wary of "just run this tool" reports without a distilled manual PoC.
- For XSS: use a **harmless, clearly-non-malicious PoC payload** (`alert(document.domain)`), never anything that could be mistaken for an actual attack against real users/data (no real cookie exfiltration against production, no destructive actions).
- For anything with real user impact (account takeover, data deletion), **stop before causing actual harm** — demonstrate the vulnerability exists via a benign proof, don't complete a full destructive exploit against real data unless the program's rules explicitly permit it.

## 4. Common Reasons Reports Get Downgraded or Rejected

| Reason | How to avoid it |
|---|---|
| Out of scope | Re-read program scope/RoE before testing, not after finding something |
| Duplicate | Search existing disclosed reports (Hacktivity) for the same class on the same target first, when visible |
| No real impact demonstrated | Show, don't just claim — a working PoC beats a theoretical description |
| Missing reproduction steps | Assume zero context; a triager with no time should be able to follow it exactly |
| Best-practice suggestion, not a vulnerability | Distinguish "this violates security best practice" from "this is exploitable" — programs generally reward the latter |
| Self-XSS / requires unrealistic victim interaction | If exploitation requires the victim to paste something into their own console, it's typically not a valid finding |

## 5. Communication Etiquette

- Be professional and patient — triage queues are often backlogged; following up politely after a reasonable window (check the program's stated SLA) is fine, repeated pinging is not.
- If a program disputes severity, respond with **evidence and reasoning**, not frustration — cite CVSS vectors and concrete impact.
- Never publicly disclose a finding before the program's disclosure policy permits it (usually after a fix is shipped, or a set embargo period).
- Never re-test or continue probing after a report is submitted unless requested — avoid unnecessary continued access to production systems/data.

## 6. From DVWA to First Real Bounty — a Practical Checklist

- [ ] Complete every chapter in `docs/` at all four difficulty levels
- [ ] Complete PortSwigger Web Security Academy's labs for each matching vulnerability class
- [ ] Pick a VDP (no bounty) program first and find/report one real, low-risk finding end-to-end
- [ ] Write the report using the structure above *before* submitting — self-review it as if you were the triager
- [ ] Iterate based on feedback; treat every rejection as calibration data, not a personal failure
