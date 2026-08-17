# 06 — Insecure CAPTCHA

## What the feature does
A "change password" flow that adds a Google reCAPTCHA step, intended to prove a human (not a script) is submitting the request.

---

## LOW

```php
if ( $_POST[ 'g-recaptcha-response' ] == null ) { die("captcha required"); }
// ... CAPTCHA verified via a server call ...
if ( $recaptcha_response_ok ) {
    $pass_new = $_POST[ 'password_new' ];
    $pass_conf = $_POST[ 'password_conf' ];
    if ( $pass_new == $pass_conf ) { updatePassword(); }
}
```

The Low-level flaw: **the password-change logic and the CAPTCHA check happen in the same request/step, but the app doesn't validate that the CAPTCHA step actually gated this specific action** — critically, if you replay/forge the `password_new`/`password_conf`/`Change` parameters directly to the processing endpoint (bypassing the form/CAPTCHA UI entirely), many DVWA builds' Low level process the password change **without ever re-verifying the CAPTCHA token server-side against Google**, or only checks that *some* field is present, not that it's a real, freshly-solved token.

**Exploitation (Burp):** Capture a legitimate password-change request that included a CAPTCHA solve, then **replay the same request repeatedly** (or forge a new one with a stale/fake `g-recaptcha-response` value) directly to the server-side handler — no browser, no actual CAPTCHA solving required, since the server-side code doesn't robustly re-validate the token against Google's `siteverify` API each time.

**Why it works:** CAPTCHA validation was implemented as "does this field exist / did the widget render," not "did Google's server cryptographically confirm this exact token, tied to this session, was solved once and only once."

---

## MEDIUM

Adds a bit more structure but the fundamental flaw remains: **the multi-step flow (step 1: solve CAPTCHA → step 2: submit new password) is split into two separate HTTP requests, and the server trusts a client-supplied "step" flag / hidden field rather than tracking server-side state.**

```php
$step = $_POST[ 'step' ];
if ( $step == '2' ) {
    // process password change — assumes step 1 (captcha) already happened
}
```

**Why it's still bypassable:** An attacker skips straight to `step=2` in a forged POST request, since nothing server-side actually confirms step 1 (the CAPTCHA) was completed for *this* request chain — the "step" is just a client-supplied value with no cryptographic/session binding to a verified CAPTCHA solve.

```bash
curl -X POST http://127.0.0.1:42001/dvwa/vulnerabilities/captcha/ \
  -b "PHPSESSID=<victim_or_own_session>; security=medium" \
  -d "step=2&password_new=pwned123&password_conf=pwned123&Change=Change"
```

**Root issue:** Multi-step "human verification" flows must track completion **server-side, tied to the session**, not via a value the client can simply set/skip.

---

## HIGH

Server-side, the CAPTCHA answer/token is verified against Google's API **and** tied server-side to session state before allowing the password-change branch to execute — closer to correct, but DVWA's High level historically still has edge cases (e.g., verifying the token but not fully invalidating it after single-use, or ordering issues in when the check happens vs. when the update query runs).

**Lesson regardless of exact bypass specifics:** the closer you get to "verify server-side, single-use, session-bound, before the sensitive action, with no client-controlled step/flag," the harder this becomes to bypass.

---

## IMPOSSIBLE

- CAPTCHA is verified server-side against Google's `siteverify` endpoint on **every** submission
- Current password required (re-authentication) in addition to CAPTCHA — same principle as CSRF's Impossible level
- No client-controlled "step" bypass — server tracks what's actually been verified
- Prepared statements

**Why this is actually secure:** The action requires **two independent, non-bypassable proofs**: (1) a fresh, single-use, server-verified CAPTCHA token, and (2) the user's actual current password. Neither can be skipped by forging a parameter, because neither check depends on trusting anything the client claims about *its own prior state* — both are independently verified against ground truth (Google's API; the stored password hash) at the moment of the sensitive action.

---

## Root Cause Summary
> "Security theater" checks (a CAPTCHA widget rendered in the UI, a `step` flag in a hidden field) are not security controls unless the server independently verifies the underlying claim on every sensitive request. Never trust client-asserted state about what already happened.

## Real-World Parallels
- Broken multi-step checkout/payment flows where "payment confirmed" is a client-side flag
- OAuth/2FA flows that can be skipped by jumping straight to the "already verified" callback URL
- OWASP Top 10: **A01:2021 – Broken Access Control** (missing server-side enforcement of business/process flow)

## Mitigation Checklist
- [ ] Verify CAPTCHA server-side against the provider's API on every submission, never trust a client-sent "solved" flag
- [ ] Track multi-step flow state server-side (session), never via client-supplied step parameters
- [ ] Make CAPTCHA tokens single-use and session-bound
- [ ] Pair CAPTCHA with re-authentication for sensitive actions
- [ ] Test every multi-step flow by trying to jump directly to the final step
