# TryHackMe - Retro Writeup

> **Platform:** TryHackMe
> **Challenge:** Retro
> **Category:** Windows | WordPress | Reverse Shell | Windows Kernel Exploitation | Privilege Escalation

---

# Overview

Today we are back again with another **Hard** rated Windows challenge from **TryHackMe** named **Retro**.

Unlike many Windows machines where exploitation begins through SMB or RDP, this challenge starts with a vulnerable WordPress installation hosted on an IIS server. Careful web enumeration reveals a hidden WordPress directory, where information disclosure eventually leads to valid administrator credentials. After gaining access to the WordPress dashboard, we abuse the Theme Editor to upload a PHP reverse shell and obtain command execution on the server. The final stage involves enumerating the Windows operating system version and exploiting an unpatched kernel vulnerability (**CVE-2017-0213**) to elevate privileges to **NT AUTHORITY\SYSTEM** and recover the root flag.

This room highlights the importance of information gathering, credential discovery, web application exploitation, and Windows privilege escalation.

---

# Target Information

| Information      | Value               |
| ---------------- | ------------------- |
| Machine Name     | Retro               |
| Platform         | TryHackMe           |
| Difficulty       | Hard                |
| Operating System | Windows Server 2016 |
| Target IP        | **10.49.151.7**     |

---

# Initial Reconnaissance

As with every penetration test, the first step is identifying the exposed attack surface.

A full TCP port scan is performed.

```bash
nmap -p- 10.49.151.7
```

The scan reveals only two open ports.

```text
80/tcp    HTTP

3389/tcp  RDP
```

Since only HTTP and RDP are exposed, the web server becomes the initial attack vector.

---

# Service Enumeration

Next, a service and version detection scan is executed.

```bash
nmap -sC -sV 10.49.151.7
```

The results reveal:

| Port | Service                     | Version             |
| ---- | --------------------------- | ------------------- |
| 80   | HTTP                        | Microsoft IIS 10.0  |
| 3389 | Microsoft Terminal Services | Windows Server 2016 |

Additional information gathered from the scan includes:

* IIS Version: **10.0**
* Windows Server 2016
* Computer Name: **RetroWeb**

One interesting observation from the HTTP scan is:

```text
Supported Methods:

OPTIONS

TRACE

GET

HEAD

POST
```

The presence of the **TRACE** method is generally considered insecure because it may assist certain attacks such as Cross Site Tracing (XST), although it is not directly abused in this room.

---

# Initial Web Enumeration

Opening the web application displays the default IIS landing page.

Since no useful information is immediately visible, directory enumeration is performed.

Gobuster is executed using a large wordlist.

```bash
gobuster dir \
-u http://10.49.151.7 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt
```

The scan discovers an interesting directory.

```text
/retro
```

Browsing to this directory reveals an actual web application rather than the default IIS page.

---

# Discovering WordPress

Since the application appears to be a CMS, another directory enumeration is performed specifically against the newly discovered directory.

The scan returns:

```text
/wp-admin

/wp-content

/wp-includes
```

These directories immediately confirm that the application is running **WordPress**.

At this point the attack surface changes completely.

Instead of targeting IIS, the assessment now focuses on WordPress.

---

# Information Disclosure

Rather than immediately searching for public exploits, the website is manually explored.

Reading blog posts and comments eventually reveals an interesting note left by one of the users.

```text
Leaving myself a note here just in case I forget how to spell it:

parzival
```

Although this initially appears harmless, it strongly resembles either a password reminder or passphrase.

Information disclosed through blog comments is frequently overlooked but can become extremely valuable.

The next step is identifying a valid username.

---

# Authenticating to WordPress

After identifying the WordPress login page, several username combinations are tested.

Using:

```text
Username:

wade

Password:

parzival
```

authentication succeeds.

Administrative access to the WordPress dashboard has now been obtained.

No software vulnerability was required.

Instead, a simple information disclosure resulted in valid credentials.

---

# Achieving Remote Code Execution

Inside the WordPress administration panel, the **Appearance → Theme Editor** functionality is available.

Administrators can directly modify theme templates.

A common post-exploitation technique is replacing the **404.php** template with a PHP reverse shell.

Using **revshells.com**, a PHP reverse shell is generated and pasted into the 404 page.

A Netcat listener is started.

```bash
nc -lvnp 4444
```

Triggering the modified page causes the server to execute the malicious PHP code.

Within a few seconds, a reverse shell is established.

```text
Microsoft Windows [Version 10.0.14393]
```

Initial command execution has been achieved.

---

# Improving the Shell

Unlike Linux, Windows does not include utilities such as:

```text
ls

which

python3
```

These commands return:

```text
not recognized as an internal or external command
```

To obtain a more usable interactive shell, the listener is restarted using:

```bash
rlwrap nc -lvnp 4444
```

This provides command history and significantly improves usability when interacting with Windows command shells.

---

# Database Credentials

While exploring the WordPress installation, the configuration file is inspected.

The WordPress configuration reveals:

```php
define('DB_NAME', 'wordpress567');

define('DB_USER', 'wordpressuser567');

define('DB_PASSWORD', 'YSPgW[%C.mQE');
```

Although these database credentials are not directly used for privilege escalation in this challenge, they represent sensitive information that would be valuable during a real-world assessment.

---

# Remote Desktop Access

Earlier during enumeration, valid WordPress credentials were recovered.

Testing the same credentials against RDP succeeds.

```bash
xfreerdp /v:10.49.151.7 /u:wade /p:parzival
```

Credential reuse allows full desktop access.

This highlights another common real-world weakness.

Users frequently reuse passwords across multiple services.

---

# User Flag

Once connected through Remote Desktop, navigating to Wade's desktop reveals the user flag.

```text
3b99fbdc6d430bfb51c72c651a261927
```

The initial objective has now been completed.

---

# Privilege Escalation Enumeration

The next objective is identifying a suitable privilege escalation vector.

Multiple commands are executed to determine the exact operating system version.

```powershell
systeminfo

Get-CimInstance Win32_OperatingSystem

Get-WmiObject Win32_OperatingSystem

[System.Environment]::OSVersion
```

All commands consistently report:

```text
Microsoft Windows Server 2016 Standard

Build 14393

Version 1607
```

Knowing the exact Windows build is essential before attempting kernel exploitation.

---

# Identifying the Vulnerability

Researching Windows Server 2016 Build **14393** reveals that the operating system is vulnerable to:

```text
CVE-2017-0213
```

This vulnerability allows local privilege escalation from a normal user account to **NT AUTHORITY\SYSTEM**.

A public proof-of-concept is available.

```text
https://github.com/SecWiki/windows-kernel-exploits/tree/master/CVE-2017-0213
```

---

# Preparing the Exploit

The exploit archive is downloaded.

```bash
unzip CVE-2017-0213_x64.zip
```

Extracting the archive provides:

```text
CVE-2017-0213_x64.exe
```

The executable is transferred to the target machine using a temporary Python HTTP server.

After downloading the binary onto the Windows host, it is executed.

Within a few seconds, the exploit successfully elevates privileges.

The current user becomes:

```text
NT AUTHORITY\SYSTEM
```

The privilege escalation is complete.

---

# Root Flag

Navigating to the Administrator desktop reveals:

```text
root.txt.txt
```

Reading the file returns:

```text
7958b56956f5d7bd88d1dc6f22d1c4063
```

The machine has now been fully compromised.

---

# Attack Flow

```text
Nmap Enumeration
        │
        ▼
Discover IIS Server
        │
        ▼
Directory Enumeration
        │
        ▼
Identify /retro
        │
        ▼
Fingerprint WordPress
        │
        ▼
Discover Password Hint
        │
        ▼
Recover WordPress Credentials
        │
        ▼
Login to WordPress Dashboard
        │
        ▼
Modify 404.php
        │
        ▼
Upload PHP Reverse Shell
        │
        ▼
Gain Initial Shell
        │
        ▼
Recover Database Credentials
        │
        ▼
Reuse Credentials for RDP
        │
        ▼
Capture User Flag
        │
        ▼
Enumerate Windows Version
        │
        ▼
Identify CVE-2017-0213
        │
        ▼
Execute Kernel Exploit
        │
        ▼
NT AUTHORITY\SYSTEM
        │
        ▼
Capture Root Flag
```

---

# Vulnerabilities Identified

* Information disclosure through WordPress comments.
* Password reuse across WordPress and RDP.
* WordPress administrative access.
* Theme Editor allowing arbitrary PHP code execution.
* Outdated Windows Server vulnerable to **CVE-2017-0213**.
* Lack of timely operating system patching.

---

# Techniques Used

* Nmap Enumeration
* IIS Enumeration
* Gobuster Directory Fuzzing
* WordPress Enumeration
* Credential Discovery
* Information Disclosure
* PHP Reverse Shell
* Windows Command Shell
* RDP Authentication
* Windows Enumeration
* Kernel Exploitation
* CVE-2017-0213
* Windows Privilege Escalation

---

# User Flag

```text
3b99fbdc6d430bfb51c72c651a261927
```

---

# Root Flag

```text
7958b56956f5d7bd88d1dc6f22d1c4063
```

---

# Key Takeaways - Jai Shri Ram

Retro demonstrates how a complete system compromise can begin with something as simple as a password reminder left in a public comment. Rather than exploiting WordPress itself, the attack relied on careful enumeration and information disclosure to obtain valid credentials. Administrative access then allowed code execution through the Theme Editor, a feature that should be restricted or disabled on production systems.

The privilege escalation phase highlights the importance of patch management. By accurately identifying the Windows build and correlating it with known vulnerabilities, it became possible to exploit **CVE-2017-0213** and elevate privileges to **NT AUTHORITY\SYSTEM**. Combined with credential reuse between WordPress and RDP, this machine reinforces several real-world lessons: avoid exposing sensitive information, enforce unique passwords across services, restrict administrative functionality, and keep operating systems fully patched against publicly known vulnerabilities.
