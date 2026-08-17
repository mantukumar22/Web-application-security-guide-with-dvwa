# 03 — CSRF (Cross-Site Request Forgery)

## What the feature does
A "change your password" form. Goal: get a logged-in victim to unknowingly submit a password-change request on the attacker's behalf, by visiting a malicious page.

---

## LOW

```php
$pass_new = $_GET[ 'password_new' ];
$pass_conf = $_GET[ 'password_conf' ];
if ( $pass_new == $pass_conf ) {
    $pass_new = md5( $pass_new );
    $query = "UPDATE users SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
}
```

**Exploitation:** No token, no origin check, and it's a **GET** request — meaning the whole attack is just a URL. Host this on any attacker-controlled page:

```html
<!-- attacker.html -->
<img src="http://127.0.0.1:42001/dvwa/vulnerabilities/csrf/?password_new=pwned123&password_conf=pwned123&Change=Change">
```

If a logged-in DVWA victim merely *loads* this page (image tag, no click needed), their browser automatically attaches the DVWA session cookie to the request, and their password is silently changed.

**Why it works:** The browser doesn't know or care which site initiated the request — it just attaches cookies for the target domain (`SameSite` not enforced, no CSRF token, state-changing action allowed via GET which is even cacheable/prefetchable by browsers).

---

## MEDIUM

```php
if ( stripos( $_SERVER[ 'HTTP_REFERER' ], $_SERVER[ 'SERVER_NAME' ] ) !== false ) {
    // process request
}
```

**Why it's still bypassable:** This checks that the `Referer` header **contains** the server's hostname as a substring — not that it *is* the exact origin. An attacker just needs a URL where the substring appears anywhere, e.g.:

```
http://evil.com/127.0.0.1/csrf-attack.html
http://127.0.0.1.evil.com/csrf-attack.html
```

Both contain `127.0.0.1` as a substring and would pass `stripos()`. Additionally, the `Referer` header can be **stripped by the client** (via `<meta name="referrer" content="no-referrer">`, certain browser privacy settings, or simply omitted on some cross-origin requests) — and many servers, if the header is *missing entirely*, fail open rather than closed depending on implementation.

**Root issue:** Referer-substring matching is trivially spoofable and the header itself isn't a reliable trust boundary.

---

## HIGH

```php
if ( ( isset( $_SESSION[ 'session_token' ] ) ) && ( $_SESSION[ 'session_token' ] === $_REQUEST[ 'user_token' ] ) ) {
    // process
}
```

Real per-session **anti-CSRF token** is required, generated server-side and embedded as a hidden field, unpredictable to an off-site attacker.

**Why this is much stronger (and mostly sufficient):** An attacker hosting a form on `evil.com` has no way to read the victim's session-bound token (Same-Origin Policy prevents JS on `evil.com` from reading the DVWA response body to extract it). The theoretical remaining gaps are implementation bugs elsewhere in the app — e.g. **if an XSS vulnerability exists anywhere on the same origin**, the attacker's injected script *can* read the DOM (including the token) and forge the request from within the trusted origin. This is why CSRF tokens and XSS defenses are treated as **complementary**, not substitutes.

---

## IMPOSSIBLE

Adds, on top of the High-level token check:
- **Requires the user's current password** to change it (re-authentication for sensitive actions)
- Proper prepared statements
- Token regenerated per request

```php
$pass_curr = $_SESSION[ 'password' ] === md5($_POST['password_current']);
if ( $pass_curr && $token_valid && $pass_new === $pass_conf ) { ... }
```

**Why this is actually secure:** Even in the (rare, chained) scenario where a token is somehow leaked or an XSS bug exists, requiring **current-password re-entry** means the attacker still needs a secret they can't script around — the victim would have to actively type their real password into the attacker's forged flow, which is a much higher bar than an invisible `<img>` tag. This is defense-in-depth: token (stops CSRF), re-auth (stops session-riding even if token stolen), prepared statements (stops any injection angle).

---

## Root Cause Summary
> State-changing actions (password change, fund transfer, email change, etc.) must never be triggered by an ambient credential (the cookie) alone. They need an unforgeable, per-session proof of *intent* — a CSRF token, `SameSite=Strict/Lax` cookies, and/or re-authentication for sensitive actions.

## Real-World Parallels
- Historic CSRF flaws in home routers to change DNS settings ("DNS rebinding via CSRF")
- OWASP Top 10 (folded into **A01:2021 – Broken Access Control** in modern OWASP editions)
- Any bug bounty program's "no CSRF token on sensitive state-change endpoint" report — common, but severity heavily depends on impact (password change > profile bio change)

## Mitigation Checklist
- [ ] CSRF token, unique per session (ideally per request), validated server-side
- [ ] `SameSite=Strict` or `Lax` on session cookies
- [ ] State-changing actions use POST/PUT/DELETE, never GET
- [ ] Re-authentication (current password / step-up auth) for high-value actions
- [ ] Don't rely on `Referer`/`Origin` headers as the *sole* control (useful as defense-in-depth, not primary)
