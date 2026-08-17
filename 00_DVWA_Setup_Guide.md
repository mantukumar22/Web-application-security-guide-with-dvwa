# 00 — DVWA Setup Guide

## 1. Lab Isolation (do this first)

DVWA must never touch a production network or the public internet.

- Use a **VM** (VirtualBox/VMware) or **Docker container** set to **NAT or Host-only** networking.
- If using Kali Linux as the attacker box, keep both attacker and target on an isolated virtual network (e.g. VirtualBox "Internal Network" or a Docker bridge network with no external route).
- Snapshot the VM before you start so you can always roll back to a clean state.

## 2. Installation Options

### Option A — Docker (fastest, recommended)

```bash
docker pull vulnerables/web-dvwa
docker run --rm -it -p 80:80 vulnerables/web-dvwa
```

Then browse to `http://127.0.0.1` (or the port you mapped, e.g. `127.0.0.1:42001` if you remapped it).

### Option B — XAMPP / manual LAMP stack

```bash
# On the target VM (never expose to internet)
sudo apt update
sudo apt install apache2 mysql-server php php-mysqli php-gd git -y

cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git dvwa
cd dvwa
sudo cp config/config.inc.php.dist config/config.inc.php
```

Edit `config/config.inc.php`:

```php
$_DVWA[ 'db_server' ] = '127.0.0.1';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ] = 'dvwa';
$_DVWA[ 'db_password' ] = 'p@ssw0rd';
```

Create the DB user in MySQL:

```sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;
```

Start services and browse to the app:

```bash
sudo service mysql start
sudo service apache2 start
```

### Option C — Kali's built-in setup (as in the reference screenshot)

Many Kali security courses ship a preconfigured DVWA reachable at something like `127.0.0.1:42001`. If yours is prebuilt, just log in — skip to step 4.

## 3. First-Time Database Setup

1. Browse to the app root.
2. Log in with the default creds: `admin` / `password`.
3. Click **Setup / Reset DB** in the left nav.
4. Click **Create / Reset Database**. This populates all the vulnerable tables (`users`, `guestbook`, etc.).

## 4. Set the Security Level

DVWA's core teaching mechanic: the **same vulnerable feature** exists at four hardening levels.

Go to **DVWA Security** (left nav) and choose:

| Level | Meaning |
|---|---|
| **Low** | No defenses. Raw, naive code. |
| **Medium** | A partial, commonly-seen (but flawed) fix. |
| **High** | A much stronger fix — usually close to correct, sometimes still bypassable. |
| **Impossible** | The reference-correct implementation. |

Set this to **Low** before starting each chapter in this repo, and re-test at Medium/High/Impossible as instructed.

Also set **PHPIDS** (intrusion detection simulator) to *disabled* for early chapters — some modules use it later to demonstrate WAF bypass concepts.

## 5. Useful Reference Pages Inside DVWA

- **Instructions** — official module descriptions.
- **View Source** button (bottom of each module) — shows the exact PHP for the current security level. **Use this constantly** — this repo's explanations assume you're reading the source alongside the exploit.
- **View Help** — hints/background reading per module.

## 6. Recommended Attacker Toolchain

```bash
# Verify tools available on Kali
which burpsuite sqlmap curl nikto dirb hydra
```

Set your browser (Firefox in the reference screenshot) to proxy through Burp:
`Settings → Network Settings → Manual proxy → 127.0.0.1:8080`, and install/trust Burp's CA cert for HTTPS interception if the app is served over TLS.

## 7. Sanity Check

Before moving to Chapter 1, confirm:
- [ ] DVWA loads and you're logged in as `admin`
- [ ] Database has been created (Setup / Reset DB was run)
- [ ] Security level control is visible and set to **Low**
- [ ] Burp (or ZAP) is intercepting your browser traffic
- [ ] You can see the "View Source" button on any module page

Proceed to **`01-brute-force.md`**.
