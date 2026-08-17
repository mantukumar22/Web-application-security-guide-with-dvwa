# 01 — Brute Force

## What the feature does
A login form (`username` + `password`) that checks credentials against the `users` table. Goal: recover valid credentials (default target is usually `admin`) without knowing the password in advance.

---

## LOW

**Code pattern:** Form submits via GET, no rate limiting, no lockout, no CAPTCHA, generic-but-consistent error message.

```php
$user = $_GET[ 'username' ];
$pass = $_GET[ 'password' ];
$pass = md5( $pass );
$query = "SELECT * FROM users WHERE user = '$user' AND password = '$pass';";
```

**Exploitation (Hydra):**

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  127.0.0.1 -s 42001 http-get-form \
  "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:F=Username and/or password incorrect" \
  -H "Cookie: PHPSESSID=<your_session>; security=low"
```

**Exploitation (Burp Intruder):** Capture the GET request → send to Intruder → mark `password` as payload position → attack type "Sniper" → load wordlist → look for a different response length/status on success.

**Why it works:** GET requests are trivial to script; there is no per-attempt cost (no delay, no CAPTCHA, no account lockout), so an attacker can send thousands of guesses per minute limited only by network/server throughput.

---

## MEDIUM

**Code pattern:** Adds `sleep(2)` after a failed login attempt.

```php
if( ( isset( $_GET[ 'Login' ] ) ) && ( sqli_result... ) ) {
   ...
} else {
   sleep( 2 );
   echo "<pre>Username and/or password incorrect.</pre>";
}
```

**Why it's still bypassable:** A fixed sleep only *slows*, it doesn't *stop*, brute forcing — and it's trivially defeated with **parallelism**. Run Hydra with many concurrent threads (`-t 64`) and the wall-clock cost per thread doesn't matter; aggregate throughput stays high. It also does nothing against a **small, targeted wordlist** (e.g. top 25 passwords) which succeeds in seconds even single-threaded.

```bash
hydra -l admin -P top25.txt -t 32 127.0.0.1 -s 42001 http-get-form "..."
```

**Root issue:** Rate-limiting on time-delay alone, with no cap on concurrent attempts and no tracking of failed-attempt count per account/IP.

---

## HIGH

**Code pattern:** Adds a CSRF token (`user_token`) to the form, delivered via a hidden field, and requires it to match a session-stored value — turning the login into a POST with anti-CSRF/anti-automation friction, plus the same `sleep(2)`.

```php
$user_token = $_SESSION[ 'session_token' ];
if ( isset( $_POST[ 'user_token' ] ) && $user_token === $_POST[ 'user_token' ] ) { ... }
```

**Why this raises the bar (but isn't perfect):** A naive brute-force script now needs to **first GET the login page, scrape the fresh token, then POST it** — this defeats simple one-shot brute-force tools but not a scripted attacker who parses the token per request (Burp Intruder supports this via "Pitchfork" + a macro/session handling rule that re-fetches the token each request). It's still theoretically brute-forceable, just meaningfully slower to build tooling for and easier to detect (token-fetch pattern is noisy in logs).

**Root issue:** Anti-automation via friction, not via actual attempt-limiting. No account lockout, no IP throttling, no anomaly detection.

---

## IMPOSSIBLE

**Code pattern:** Adds real defenses:
- Prepared statements (fixes any SQLi angle too)
- **Account lockout after N failed attempts**, tracked in a `login_attempts` / `failed_login` table keyed by user + IP, with a time-based lockout window
- CSRF token required
- `sleep(2)` still present (defense in depth)

```php
// pseudocode of the real logic
if ( too_many_failed_attempts_for( $user, $ip ) ) {
    show_locked_out_message();
    exit;
}
```

**Why this is actually secure:** Brute force fundamentally requires *many attempts*. Impossible-level DVWA caps attempts **per account and per IP within a time window**, so the attacker either gets locked out (denial, but no credential leak) or must slow to a rate that's operationally useless. Combined with prepared statements (no SQLi shortcut) and CSRF tokens (no scripted replay without a real session), the only viable path left is a targeted, extremely-low-and-slow attack — which is why real-world defenses add MFA and anomaly/IP-reputation detection on top of lockouts.

---

## Root Cause Summary
> Authentication endpoints without rate-limiting, lockout, or MFA reduce "guess the password" to a pure throughput problem — and computers have a lot of throughput.

## Real-World Parallels
- Credential stuffing breaches (reused passwords + no rate limiting = automated account takeover at scale)
- OWASP Top 10: **A07:2021 – Identification and Authentication Failures**
- CVEs where login endpoints lacked rate limiting are a recurring bug bounty category (often triaged as "Low" severity alone, but chained with leaked credential lists it becomes critical)

## Mitigation Checklist
- [ ] Rate-limit by IP **and** by account
- [ ] Account lockout / exponential backoff after N failures
- [ ] CAPTCHA after a threshold of failures
- [ ] MFA for sensitive accounts
- [ ] Generic error messages (don't leak "user exists" vs "wrong password")
- [ ] Log and alert on failed-login spikes
- [ ] Prefer POST over GET for credentials (avoid logging passwords in server/proxy access logs and browser history)
