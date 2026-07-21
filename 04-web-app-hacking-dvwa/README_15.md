# Lab 04 – Web Application Hacking with DVWA

## Overview

This lab sets up *DVWA (Damn Vulnerable Web Application)* on my Ubuntu Server and uses it to practice three of the most common web application vulnerabilities: Command Injection, SQL Injection, and Cross-Site Scripting (XSS).

DVWA is a PHP web application built to be insecure — every vulnerability is intentional, making it the standard practice tool for learning web app security in a safe, legal environment. Think of it as a punching bag specifically designed to be hit.

This lab had a significant troubleshooting component — a MySQL 8.4 compatibility issue with DVWA's setup script required reading Apache error logs, identifying the root cause, and editing DVWA's source code directly to fix it. That troubleshooting process ended up being one of the most valuable parts of the lab.

---

## 🎯 Objectives

- Install and configure DVWA on the Ubuntu Server VM
- Troubleshoot a MySQL 8.4 compatibility issue by reading error logs and editing source code
- Exploit Command Injection to run arbitrary OS commands through a web form
- Exploit SQL Injection to dump the entire user database and extract password hashes
- Execute Reflected XSS and Stored XSS attacks
- Understand the real-world impact of each vulnerability

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| DVWA | Deliberately vulnerable web application (target) |
| Apache 2.4.66 | Web server hosting DVWA |
| MySQL 8.4 | Database backend |
| PHP 8.5.4 | Server-side scripting |
| Firefox (Kali) | Browser used to interact with DVWA |

---

## 🌐 Lab Setup

```
[ Kali Linux VM ]  →  Firefox  →  [ Ubuntu Server: 192.168.50.47 ]
   (Attacker)                         Apache + MySQL + DVWA
```

Both VMs on Bridged network adapter, same subnet.

---

## ⚙️ Step 1 – Verify Prerequisites

SSH'd into Ubuntu Server from Kali and confirmed Apache and MySQL were running from the previous lab:

```bash
sudo systemctl status apache2
sudo systemctl status mysql
```

Both showed *active (running)*:

![Apache and MySQL confirmed running, DVWA git clone in progress](./screenshots/01-apache-mysql-running-dvwa-cloned.png)

---

## ⚙️ Step 2 – Install DVWA

Cloned DVWA directly from GitHub into Apache's web root:

```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data /var/www/html/DVWA/
cd /var/www/html/DVWA/config
sudo cp config.inc.php.dist config.inc.php
```

![DVWA cloned successfully, permissions set, config copied](./screenshots/02-apache-mysql-status-config-copied.png)

---

## ⚙️ Step 3 – Configure DVWA

Opened the config file in nano:

```bash
sudo nano /var/www/html/DVWA/config/config.inc.php
```

The default credentials were already correct — database set to `dvwa`, user `dvwa`, password `p@ssw0rd`. Changed the security level from `impossible` to `low` so vulnerabilities could actually be exploited:

![DVWA config file showing database settings and security level](./screenshots/03-dvwa-config-file-nano.png)

---

## ⚙️ Step 4 – Set Up the MySQL Database

Created the DVWA database and user in MySQL:

```bash
sudo mysql -u root
```

```sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

![MySQL database setup commands and Query OK responses](./screenshots/04-mysql-database-setup.png)

---

## ⚙️ Step 5 – DVWA Setup Page

Opened Firefox in Kali and navigated to `http://192.168.50.47/DVWA/setup.php`:

![DVWA setup page showing configuration check](./screenshots/05-dvwa-setup-page.png)

![DVWA setup page bottom showing database config and Create button](./screenshots/06-dvwa-setup-page-bottom.png)

Several items showed red — reCAPTCHA missing, mod_rewrite not enabled, PHP display_errors disabled. None of these affect the core vulnerability exercises (SQL Injection, Command Injection, XSS) so they can be safely ignored.

---

## 🐛 Troubleshooting – MySQL 8.4 Compatibility Issue

Clicking **"Create / Reset Database"** returned a generic error page. Checked Apache error logs to find the real cause:

```bash
sudo tail -f /var/log/apache2/error.log
```

The log revealed:
```
PHP Fatal error: Uncaught mysqli_sql_exception: You have an error in 
your SQL syntax near 'IF NOT EXISTS role VARCHAR(20) DEFAULT 'user''
```

**Root cause:** MySQL 8.4 removed support for `ADD COLUMN IF NOT EXISTS` in ALTER TABLE statements — a syntax that older DVWA versions used in their setup script.

**Fix:** edited DVWA's MySQL setup file directly to remove the unsupported syntax:

```bash
sudo grep -n "IF NOT EXISTS" /var/www/html/DVWA/dvwa/includes/DBMS/MySQL.php
```

Found two instances on different lines. Changed both from:
```php
ADD COLUMN IF NOT EXISTS role VARCHAR(20)
ADD COLUMN IF NOT EXISTS account_enabled TINYINT(1)
```

To:
```php
ADD COLUMN role VARCHAR(20)
ADD COLUMN account_enabled TINYINT(1)
```

After saving, the database created successfully and DVWA redirected to the login page.

**Key lesson:** when a web app throws a generic error, the real information is always in the server logs — not the browser. `tail -f /var/log/apache2/error.log` is now a reflex.

---

## ✅ Step 6 – Login to DVWA

Logged in with default credentials `admin` / `password`:

![DVWA login page](./screenshots/07-dvwa-login-page.png)

![DVWA dashboard showing all vulnerability modules](./screenshots/08-dvwa-dashboard.png)

The left menu shows every vulnerability module available: Brute Force, Command Injection, CSRF, File Inclusion, SQL Injection, XSS, and more.

---

## 🔬 Exercise 1 – Command Injection

Clicked **Command Injection** — the app presents a "Ping a device" form that takes an IP address and pings it. The developer didn't sanitize the input, so additional commands can be chained onto it using `;`.

**Legitimate use:**
```
192.168.50.77
```
Returns normal ping output — 4 packets, 0% loss.

**Injecting `whoami`:**
```
192.168.50.77; whoami
```
Returns `www-data` at the bottom of the ping output — the web server ran our command and revealed what user it's running as.

**Injecting `id`:**
```
192.168.50.77; id
```
Returns `uid=33(www-data) gid=33(www-data) groups=33(www-data)` — confirming the web server's privilege level.

**Injecting `cat /etc/passwd`:**
```
192.168.50.77; cat /etc/passwd
```
Dumped the entire system user file — every account on the server, their home directories, and shell assignments:

![Command injection dumping /etc/passwd showing all system users](./screenshots/10-command-injection-etc-passwd.png)

**Injecting `uname -a`:**
```
192.168.50.77; uname -a
```
Revealed the full OS and kernel version: `Linux ubuntu-server-01 7.0.0-27-generic #27-Ubuntu SMP`:

![Command injection revealing OS and kernel version](./screenshots/11-command-injection-uname.png)

**Why this matters:** with command injection an attacker can read config files, discover other services, create reverse shells, and move laterally through the network. The `;` is all it takes when input isn't sanitized.

**Attempted `/etc/shadow`:** tried to read the password hash file but it returned no output — `www-data` doesn't have permission to read that file. This shows why running web servers as low-privilege users is important — it limits the damage even when injection is possible.

---

## 🔬 Exercise 2 – SQL Injection

Clicked **SQL Injection** — the app presents a User ID lookup form that queries the database.

**Normal query:**
```
1
```
Returns: `First name: admin, Surname: admin` — working as intended.

**Breaking the query:**
```
1' OR '1'='1
```
Returns ALL users in the database — the injected condition `'1'='1'` is always true so the WHERE clause matches every record:

![SQL injection dumping all users with OR 1=1](./screenshots/12-sql-injection-all-users.png)

**Extracting password hashes with UNION SELECT:**
```
1' UNION SELECT user, password FROM users-- -
```
Pulled every username and MD5 password hash from the users table:

![SQL injection UNION SELECT dumping usernames and password hashes](./screenshots/13-sql-injection-password-hashes.png)

The hashes are weak MD5 — crackable instantly against rainbow tables:

| Username | Hash | Plaintext |
|----------|------|-----------|
| admin | `5f4dcc3b5aa765d61d8327deb882cf99` | `password` |
| gordonb | `e99a18c428cb38d5f260853678922e03` | `abc123` |
| pablo | `0d107d09f5bbe40cade3de5c71e9e9b7` | `letmein` |
| smithy | `5f4dcc3b5aa765d61d8327deb882cf99` | `password` |

**This is how real data breaches happen** — one unsanitized input field, one UNION SELECT, and every user's credentials are exposed.

---

## 🔬 Exercise 3 – Cross-Site Scripting (XSS)

**Reflected XSS:**

Clicked **XSS (Reflected)** and typed into the name field:
```html
<script>alert('XSS!')</script>
```

The browser executed the JavaScript and popped an alert box showing "XSS!" from `192.168.50.47`:

![Reflected XSS alert box firing from injected script](./screenshots/14-xss-reflected-alert.png)

**What this means:** the server took the input, embedded it directly into the HTML response without sanitizing it, and the browser executed it as code. In a real attack, the script could steal session cookies, log keystrokes, or redirect to a phishing page.

**Stored XSS:**

Clicked **XSS (Stored)** and submitted a script in the guestbook message field. The alert fired immediately on page load — and continued firing on every subsequent page refresh because the script was saved to the database and re-executed for every visitor.

**Reflected vs Stored XSS:** reflected XSS requires tricking a specific user into clicking a crafted link. Stored XSS plants the payload once and it attacks every user who visits the page automatically — significantly more dangerous.

---

## 📝 What I Learned

| Vulnerability | Root Cause | Real World Impact |
|--------------|-----------|------------------|
| Command Injection | Unsanitized user input passed to OS shell | Attacker runs arbitrary commands on the server |
| SQL Injection | Unsanitized input embedded in SQL queries | Full database dump, credential theft, authentication bypass |
| Reflected XSS | Unsanitized input reflected in HTML response | Session hijacking, phishing, credential theft |
| Stored XSS | Malicious input saved to database and re-rendered | Persistent attack affecting every visitor |

**The common thread:** every single one of these attacks exists because a developer trusted user input without sanitizing it. Input validation and parameterized queries would prevent all of them.

---

## 🐛 Troubleshooting Log

| Problem | Cause | Fix |
|---------|-------|-----|
| "Problem with this site" on Create Database | MySQL 8.4 removed `ADD COLUMN IF NOT EXISTS` syntax | Read Apache error logs, edited DVWA's MySQL.php to remove unsupported syntax |
| git pull "dubious ownership" error | sudo vs labadmin git config mismatch | Ran `sudo git config --global --add safe.directory` |
| Empty nano editor | Opened wrong path / wrong file | Used full absolute path to open correct config file |

---

## ⏭️ Next Steps

- Revisit DVWA on Medium and High security levels — same attacks need to be adapted
- Try SQL Injection (Blind) module
- Explore Burp Suite for intercepting and modifying HTTP requests
- Move on to [Lab 03 – Active Directory](../03-active-directory/README.md)
