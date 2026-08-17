# Bug Bounty Methodology — From DVWA to Real Programs

This chapter bridges DVWA's controlled, labeled vulnerability classes to unlabeled, authorized real-world targets.

## 1. Legal Foundation First

Never test anything you don't have explicit written authorization to test.

- **Bug bounty programs** (HackerOne, Bugcrowd, Intigriti, YesWeHack, or company-run programs) publish a **scope** and **rules of engagement (RoE)** — read them fully before touching anything.
- Stay strictly within listed domains/apps/IP ranges. Out-of-scope findings are usually not eligible and testing them may itself be unauthorized access.
- Respect explicit exclusions (e.g. "no automated scanning," "no DoS testing," "no social engineering").
- If unsure whether something is in scope, ask the program via their official contact channel before testing.

## 2. Mapping DVWA Skills to Bug Bounty Categories

| DVWA Module | Real-World Bug Bounty Category | Typical Severity |
|---|---|---|
| SQL Injection / Blind SQLi | SQLi | Critical (data breach) |
| Command Injection | RCE | Critical |
| File Upload | Arbitrary File Upload → RCE | Critical |
| File Inclusion | LFI/RFI, sometimes → RCE | High–Critical |
| XSS (Reflected/Stored/DOM) | XSS | Low–High depending on impact/auth context |
| CSRF | CSRF | Low–High depending on action sensitivity |
| Weak Session IDs | Session prediction/hijacking | High |
| Insecure CAPTCHA | Business logic / flow bypass | Low–Medium (context-dependent) |
| Open Redirect | Open Redirect | Low, higher if chained (OAuth) |
| CSP Bypass | Security misconfiguration | Info–Medium |

## 3. General Testing Workflow

1. **Recon** — map the attack surface (see `01-recon-and-tooling.md`)
2. **Understand the app's logic** — what does each endpoint/role/feature *intend* to do?
3. **Look for the trust boundary violations** — every vuln class above is really "input trusted where it shouldn't be" or "action allowed without proper verification"
4. **Test systematically per input** — every parameter, header, cookie, and file upload field is a potential injection point
5. **Chain findings** — a "low severity" open redirect + a "low severity" CSRF might combine into account takeover; think about impact chains, not isolated bugs
6. **Verify impact, don't just prove existence** — a working PoC that demonstrates real impact (data read, account takeover, RCE) is far more valuable and credible than "alert(1)"

## 4. Severity Thinking (CVSS-informed, but think practically)

Ask:
- **Confidentiality** — can this read data it shouldn't?
- **Integrity** — can this modify data/state it shouldn't?
- **Availability** — can this deny service?
- **Privilege required** — does exploitation need auth? Admin? None?
- **User interaction** — does a victim need to click something, or is it zero-click?
- **Scope** — does it affect only the vulnerable component, or escalate beyond it (e.g. XSS → session hijack → account takeover)?

A reflected XSS with no cookie protection and admin-panel reach is very different in severity from the same bug on a static marketing page with no session data.

## 5. Building a Personal Practice Pipeline

1. Master each DVWA module at all four levels (this repo)
2. Move to **PortSwigger Web Security Academy** (free, excellent, maps almost 1:1 to real bug classes with modern frameworks)
3. Practice on **OWASP Juice Shop**, **bWAPP**, **HackTheBox** web challenges
4. Read public disclosed reports on **HackerOne Hacktivity** to see how real findings are written up
5. Start with programs offering a **VDP (Vulnerability Disclosure Program)** — no bounty, lower stakes, still legal and good practice
6. Move to paid bounty programs once comfortable with methodology and reporting quality

See `02-reporting-and-triage.md` for how to write a report a triager will actually act on quickly.
