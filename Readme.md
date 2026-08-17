# DVWA Web Application Security Guide

A structured, hands-on repository for learning web application security using **DVWA (Damn Vulnerable Web Application)** — built the way a security professional would document an internal training program: setup → root-cause analysis → exploitation → mitigation, at every difficulty level.

> ⚠️ **Legal / Ethical Notice**
> DVWA is *intentionally* vulnerable. Only run it in an isolated lab (local VM, container, or air-gapped network). Never expose it to the internet or a shared network. Everything in this repo applies **only** to systems you own or are explicitly authorized to test. Using these techniques against systems you don't own is illegal in most jurisdictions (e.g. CFAA in the US, Computer Misuse Act in the UK).

---

## Why This Repo Exists

Most DVWA write-ups just say "type this payload." This repo instead teaches the **underlying vulnerability class** — the insecure code pattern, why the attack works at each security level, and why it stops working at the next level — so the knowledge transfers to real bug bounty targets and real CTFs, not just DVWA.

Every chapter follows the same format:

1. **What the feature does** (functional description)
2. **Low** — the naive/vulnerable implementation, exploited step by step
3. **Medium** — a partial fix, and why it's still bypassable
4. **High** — a much stronger fix, and why it's usually still theoretically flawed
5. **Impossible** — the "correct" secure implementation, and *why* it's actually secure
6. **Root cause summary** — the one-sentence lesson
7. **Real-world / CVE parallels** — where this bug class shows up outside DVWA
8. **Mitigation checklist** — what a developer/reviewer should do

---

## Repo Structure

```
dvwa-security-guide/
├── README.md                          ← you are here
├── docs/
│   ├── 00-setup.md                    ← install & configure DVWA safely
│   ├── 01-brute-force.md
│   ├── 02-command-injection.md
│   ├── 03-csrf.md
│   ├── 04-file-inclusion.md
│   ├── 05-file-upload.md
│   ├── 06-insecure-captcha.md
│   ├── 07-sql-injection.md
│   ├── 08-sql-injection-blind.md
│   ├── 09-weak-session-ids.md
│   ├── 10-xss-dom.md
│   ├── 11-xss-reflected.md
│   ├── 12-xss-stored.md
│   ├── 13-csp-bypass.md
│   ├── 14-javascript-attacks.md
│   └── 15-open-redirect.md
└── bug-bounty/
    ├── 00-methodology.md              ← how these bug classes map to real programs
    ├── 01-recon-and-tooling.md
    └── 02-reporting-and-triage.md
```

---

## Suggested Learning Path

| Stage | Chapters | Focus |
|---|---|---|
| 1. Foundations | Setup, Brute Force, Weak Session IDs | Auth & session mechanics |
| 2. Injection | SQL Injection, SQLi (Blind), Command Injection | Untrusted input reaching interpreters |
| 3. Client-side | XSS (Reflected/Stored/DOM), CSP Bypass, JavaScript | Browser trust model |
| 4. File handling | File Inclusion, File Upload | Path/content trust boundaries |
| 5. Logic flaws | CSRF, Insecure CAPTCHA, Open Redirect | Missing verification of intent/origin |
| 6. Applied | `bug-bounty/` | Turning DVWA skills into real findings |

Work through each module at **Low → Medium → High → Impossible**, in order, without skipping ahead — the point isn't just "get the flag," it's understanding *why each fix works or fails*.

---

## Tooling Used Throughout

- **Burp Suite Community/Pro** — intercepting proxy, repeater, intruder
- **sqlmap** — automated SQL injection
- **curl / httpie** — quick manual requests
- **Browser DevTools** — DOM inspection, console, network tab
- **OWASP ZAP** — alternative to Burp
- Standard Kali Linux toolset (as seen in the reference environment)

## Quick Start

```bash
git clone <this-repo>
cd dvwa-security-guide
cat docs/00-setup.md   # start here
```

## Legal / Scope Reminder

Techniques in this repo are taught for:
- Personal lab learning (DVWA, PortSwigger Academy, HackTheBox, etc.)
- Authorized penetration testing engagements
- Bug bounty programs **within their published scope and rules of engagement**

See `bug-bounty/00-methodology.md` for how to responsibly transition these skills to real-world authorized testing.
