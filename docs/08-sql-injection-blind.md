# 08 — SQL Injection (Blind)

## What the feature does
Same "look up user by ID" concept, but the response **only ever says "User ID exists" or "User ID is MISSING from the database"** — no data is ever echoed back directly. This forces inference-based extraction.

---

## LOW

```php
$id = $_REQUEST[ 'id' ];
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
if ( mysqli_num_rows($result) ) { echo "User exists"; } else { echo "User missing"; }
```

**Exploitation — Boolean-based blind:**
```sql
1' AND '1'='1     -->  "User exists"   (true condition)
1' AND '1'='2     -->  "User missing"  (false condition)

-- Extract data one character at a time
1' AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='5
1' AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a
-- ... iterate through charset until true, then move to position 2, etc.
```

**Exploitation — Time-based blind** (useful when even the true/false response is identical):
```sql
1' AND IF(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='5', SLEEP(5), 0)-- -
```
If the response takes ~5 seconds, the guessed character is correct.

**Automated (sqlmap handles both techniques automatically):**
```bash
sqlmap -u "http://127.0.0.1:42001/dvwa/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" \
  --cookie="PHPSESSID=<id>; security=low" \
  --technique=BT --dump -T users
```

**Why it works:** Even without any data being echoed, the *existence of any observable difference* (response text, timing, HTTP status) tied to a true/false condition is enough to extract data one bit/character at a time via automated iteration.

---

## MEDIUM

Same pattern as classic SQLi Medium: `mysqli_real_escape_string()` applied, query still built via concatenation, and input often expected via a dropdown-restricted `id`.

**Why it's still bypassable:** Same root cause as Chapter 07 Medium — the query is `WHERE user_id = '$id'` (quoted, escaped) so a raw `'` is blocked, **but** true blind injection frequently doesn't even require breaking out of quotes when the app logic can be probed differently, and more importantly: if any parameter or endpoint (session-tracked `id`, cookie-based `id`, or another field) is unquoted or improperly escaped, or if second-order injection exists (unescaped when the value is *reused* in a later query), the same boolean/time-based technique applies. Additionally, escaping doesn't stop an attacker from using `sqlmap`'s automated tamper scripts, which try many known escaping-bypass patterns automatically.

**Root issue:** Same as classic SQLi — escaping is not equivalent to parameterization, and blind injection just needs *one* observable signal, which is very hard to fully suppress once concatenation exists anywhere in the flow.

---

## HIGH

Similar to classic SQLi High: session-tracked ID flow, escaping instead of parameterization, `LIMIT 1` added.

**Why it's still bypassable:** The observable signal (exists/missing text, or timing) still exists as long as the query is influenced by unescaped/unparameterized attacker input at any point in the flow (e.g., where the session `id` value is first set). Blind injection is often *easier* to miss in code review than classic SQLi precisely because the response looks so restrained/safe — but "the response doesn't show data" is not the same as "the query is safe."

---

## IMPOSSIBLE

```php
$id = $_GET[ 'id' ];
$stmt = $GLOBALS["___mysqli_ston"]->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (?) AND user_id = (?) LIMIT 1;' );
$stmt->bind_param( 'ss', $id, $id );
$stmt->execute();
```

**Why this is actually secure:** Identical reasoning to Chapter 07 — parameterized queries mean the input can never alter the query's logical structure, so there is no boolean condition to manipulate and no true/false signal to extract data through, regardless of how the response is displayed (verbose or restrained).

---

## Root Cause Summary
> Blind SQL injection proves that "the app doesn't show data back to me" is not a security control. Any observable difference tied to attacker-influenced query logic (text, timing, status code, response size) can be automated into a full data-extraction oracle. The fix is identical to classic SQLi: parameterized queries.

## Real-World Parallels
- Time-based blind SQLi is a common technique against production APIs that return generic `200 OK` / `400 Bad Request` with no error detail — attackers still extract full databases via timing alone
- sqlmap's default technique set (`B`,`E`,`U`,`S`,`T`,`Q`) automatically tries boolean, error-based, union, stacked, time-based, and inline-query techniques against every injectable parameter it finds

## Mitigation Checklist
- [ ] Parameterized queries (same fix as classic SQLi — this is the same root vulnerability, different exfiltration technique)
- [ ] Rate-limit/anomaly-detect on requests with abnormal timing patterns (defense-in-depth, not a fix)
- [ ] Uniform response times and generic error responses reduce oracle strength but don't eliminate the vulnerability
- [ ] Log and alert on `SLEEP()`/`WAITFOR`/`BENCHMARK()`-style patterns in inputs (detective control only)
