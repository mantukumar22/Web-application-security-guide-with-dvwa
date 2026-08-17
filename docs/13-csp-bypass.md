# 13 — CSP Bypass

## What the feature does
Demonstrates **Content Security Policy (CSP)** — an HTTP response header that restricts which sources of scripts/styles/etc. the browser is allowed to execute — and shows how a misconfigured policy can still be bypassed even though CSP is "enabled."

CSP is a **defense-in-depth** layer against XSS, not a replacement for output encoding.

---

## LOW

**Policy example:**
```
Content-Security-Policy: script-src 'self' https://pastebin.com
```

**Why this is bypassable:** Allowlisting a broad, attacker-writable third-party host (like a paste service or open CDN) as a script source defeats the purpose of CSP — an attacker who can inject a `<script src="...">` tag (via a separate XSS bug, or a page that reflects a `src` parameter) can point it at attacker-controlled content **hosted on the allowlisted domain**:

```html
<script src="https://pastebin.com/raw/attackercontrolledpaste"></script>
```

Since `pastebin.com` is explicitly trusted by the policy, the browser executes it without complaint. This is a very common real-world CSP misconfiguration: trusting a domain that allows arbitrary user-uploaded content (JSONP endpoints, open redirects, paste sites, some CDNs with wildcard user content paths).

---

## MEDIUM

**Policy example:**
```
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

**Why this is bypassable:** `'unsafe-inline'` completely defeats CSP's core protection against XSS — it explicitly permits inline `<script>` tags and inline event handlers (`onerror=`, `onclick=`) to execute, which is exactly what CSP exists to block. If **any** reflected/stored/DOM XSS bug exists anywhere on the page, `'unsafe-inline'` means CSP provides **zero** additional protection against it.

```html
<img src=x onerror=alert(document.cookie)>  <!-- still executes with 'unsafe-inline' -->
```

**Root issue:** `'unsafe-inline'` is often added by developers to make CSP "stop breaking" a legacy app with inline scripts — but doing so removes CSP's main XSS-mitigation value entirely.

---

## HIGH

**Policy example:**
```
Content-Security-Policy: script-src 'self'
```

Much stronger — only same-origin scripts execute, no inline, no external hosts. **Why it can still have gaps:** if the application itself hosts an endpoint that reflects user input as JavaScript (e.g. a JSONP-style callback, or a same-origin page that echoes a `callback=` parameter into a `<script>` context), `'self'` still permits it, because it *is* same-origin:

```
https://target/jsonp?callback=alert(document.cookie)//
```
```html
<script src="https://target/jsonp?callback=alert(document.cookie)//"></script>
```

**Root issue:** `'self'` is strong, but only as strong as the least-trustworthy same-origin endpoint — any user-content-reflecting endpoint on the same domain (uploads, JSONP, open redirects to same-origin script paths) can become a CSP bypass gadget.

---

## IMPOSSIBLE

**Policy example (nonce-based):**
```
Content-Security-Policy: script-src 'nonce-<random-per-request-value>'; object-src 'none'; base-uri 'self';
```

Every legitimate `<script>` tag on the page must carry the exact, freshly-generated nonce for that response:
```html
<script nonce="Xy82ka...">/* legit app code */</script>
```

**Why this is actually secure:** A **per-request random nonce** means an attacker's injected script (even via an otherwise-successful XSS injection point) cannot execute, because they cannot predict or steal the nonce ahead of time to attach to their injected tag — it's regenerated fresh for every single response and never reused. Combined with `object-src 'none'` (blocks Flash/plugin-based bypass vectors) and `base-uri 'self'` (prevents `<base>` tag hijacking of relative URLs), this closes off essentially every classic CSP bypass technique. This is the modern, recommended CSP approach over static allowlists.

---

## Root Cause Summary
> CSP is only as strong as its weakest allowed source. Broad allowlists (`'unsafe-inline'`, trusted-but-attacker-writable domains, or same-origin endpoints that reflect user input into script context) all reopen the exact XSS risk CSP is meant to mitigate. Nonce- or hash-based policies, refreshed per request, are the gold standard.

## Real-World Parallels
- CSP bypass research is an entire subfield — JSONP endpoints on allowlisted CDNs (Google, Angular CDN, etc.) have historically been used as real-world CSP bypass gadgets
- `unsafe-inline` and wildcard (`*`) script-src policies remain extremely common misconfigurations found in bug bounty CSP audits
- OWASP Top 10: **A05:2021 – Security Misconfiguration**

## Mitigation Checklist
- [ ] Never use `'unsafe-inline'` or `'unsafe-eval'` in `script-src`
- [ ] Prefer nonce- or hash-based CSP over static domain allowlists
- [ ] Audit every allowlisted domain for user-content/JSONP/open-redirect endpoints
- [ ] Set `object-src 'none'` and `base-uri 'self'`
- [ ] Treat CSP as defense-in-depth — fix the underlying XSS (output encoding) regardless
- [ ] Use `Content-Security-Policy-Report-Only` first to test policies before enforcing
