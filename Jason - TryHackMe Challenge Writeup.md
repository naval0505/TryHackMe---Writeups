# Jason - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine Name | Jason |
| Difficulty | Easy |
| Operating System | Linux |
| Category | Web Exploitation / Insecure Deserialization |

---

# Introduction

Today we are solving another **TryHackMe Easy** Linux machine named **Jason**.

This machine focuses on a dangerous web application vulnerability known as **Insecure Deserialization** within a Node.js application. Instead of exploiting SQL Injection or File Upload vulnerabilities, the attack abuses the application's trust in serialized objects stored inside client-side cookies. By crafting a malicious serialized object, it becomes possible to execute arbitrary JavaScript on the server and obtain Remote Code Execution (RCE).

The challenge demonstrates why deserializing untrusted user input is considered one of the most critical web application vulnerabilities. A simple cookie modification ultimately leads to complete compromise of the web application and, due to overly permissive sudo privileges, full root access.

---

# Objectives

The objectives for this machine are:

- Enumerate exposed services.
- Analyze the web application.
- Identify the insecure deserialization vulnerability.
- Gain Remote Code Execution.
- Obtain an interactive shell.
- Escalate privileges.
- Capture both flags.

---

# Attack Methodology

The complete attack path followed during this machine is shown below.

```
Reconnaissance
        │
        ▼
Port Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
Cookie Analysis
        │
        ▼
Node.js Deserialization
        │
        ▼
Remote Code Execution
        │
        ▼
Reverse Shell
        │
        ▼
Shell Stabilization
        │
        ▼
Local Enumeration
        │
        ▼
Privilege Escalation
        │
        ▼
Root
```

---

# Initial Reconnaissance

The target machine IP is:

```
10.48.140.31
```

As with every penetration test, the first step is identifying exposed network services.

A full TCP port scan is performed.

```bash
nmap -p- 10.48.140.31
```

Results:

```
22/tcp

80/tcp
```

Only two ports are exposed.

- SSH
- HTTP

Although the attack surface appears minimal, further enumeration is required.

---

# Service Enumeration

A service detection scan is performed.

```bash
nmap -sC -sV 10.48.140.31
```

Results:

```
22/tcp

OpenSSH 8.2p1

80/tcp

HTTP
```

The HTTP page title reveals:

```
Horror LLC
```

No additional directories or technologies are immediately disclosed through the Nmap scripts.

---

# Initial Website Analysis

Browsing the website presents a simple landing page.

The application appears to contain a newsletter subscription form requesting an email address.

Nothing immediately appears vulnerable.

Instead of interacting only through the browser, Burp Suite is used to inspect every request and response exchanged with the server.

---

# Intercepting Requests

Burp Suite reveals an interesting workflow after submitting an email address.

The application performs four distinct actions.

### Step 1

The browser sends a POST request containing the supplied email address.

```
POST

email=test@example.com
```

---

### Step 2

The server responds with a cookie.

Example:

```
Set-Cookie:

session=
eyJlbWFpbCI6InRlc3RAZXhhbXBsZS5jb20ifQ==
```

The value appears to be Base64 encoded.

---

### Step 3

The browser automatically refreshes the page.

A GET request is then sent including the newly issued cookie.

---

### Step 4

The application extracts the email from the cookie and displays:

```
We'll keep you updated at:

test@example.com
```

This confirms that the server trusts information stored inside the client-controlled cookie.

---

# Decoding the Cookie

Decoding the Base64 value reveals a JSON object.

Example:

```json
{
    "email":"test@example.com"
}
```

This immediately raises suspicion.

Instead of storing a random session identifier, the application stores serialized user data directly inside the cookie.

Applications that deserialize client-controlled objects frequently become vulnerable to **Insecure Deserialization**.

---

# Understanding Insecure Deserialization

Serialization converts objects into a transferable format.

```
Object

↓

Serialized Data

↓

Cookie
```

Later the application performs:

```
Cookie

↓

Deserialize

↓

Object
```

The vulnerability occurs because the application blindly trusts user-controlled serialized data.

If dangerous object types are supported during deserialization, arbitrary code execution becomes possible.

---

# Discovering the Vulnerability

Inspecting the application source reveals the use of:

```
node-serialize
```

This Node.js package contains a well-known Remote Code Execution vulnerability.

Instead of storing only data, specially crafted objects containing executable JavaScript functions may also be deserialized.

When the application calls:

```javascript
unserialize()
```

the embedded function executes automatically.

This vulnerability has been publicly documented for several years.

---

# Understanding the Exploit

Normally the application expects:

```json
{
  "email":"user@example.com"
}
```

Instead, an attacker can supply:

```javascript
{
"_$$ND_FUNC$$_function(){ ... }()
}
```

During deserialization:

```
Deserialize

↓

Execute Function

↓

Run System Commands
```

Rather than simply rebuilding an object, the application unintentionally executes attacker-controlled JavaScript.

---

# Crafting the Payload

Initially a simple proof-of-concept payload is used.

```javascript
{
"email":"_$$ND_FUNC$$_function(){ return 2+2; }()"
}
```

Successful execution confirms that arbitrary JavaScript code is being evaluated.

The proof of concept is then replaced with a reverse shell payload.

---

# Building the Reverse Shell

A Node.js reverse shell is generated.

The payload launches:

- net
- child_process
- /bin/sh

before connecting back to the attacker's machine.

A simplified version appears below.

```javascript
require('child_process')
```

The complete serialized payload is then Base64 encoded before replacing the original cookie value.

---

# Injecting the Malicious Cookie

Instead of modifying application parameters, only the session cookie requires replacement.

The malicious request now contains a serialized function rather than a normal email address.

Once the request reaches the vulnerable server, the application performs:

```
Cookie

↓

Deserialize

↓

Execute JavaScript

↓

Spawn Reverse Shell
```

No authentication bypass or upload functionality is required.

The vulnerability exists entirely within cookie processing.

---

# Receiving the Reverse Shell

Before sending the payload, a Netcat listener is started.

```bash
nc -lvnp 4444
```

After the malicious cookie is submitted, the listener immediately receives a connection.

```
connect to

10.48.140.31
```

Checking the current user.

```bash
whoami
```

Output:

```
ubuntu
```

Unlike many web applications that execute as **www-data**, this application runs directly as the **ubuntu** user.

This provides a significantly stronger initial foothold.

---

# Shell Stabilization

To obtain a fully interactive shell, Python PTY is used.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Suspend the session.

```
CTRL + Z
```

Configure the local terminal.

```bash
stty raw -echo
```

Resume the reverse shell.

```bash
fg
```

Finally export a proper terminal.

```bash
export TERM=xterm-256color
```

The reverse shell now behaves similarly to a standard SSH session, making post-exploitation significantly easier.

---

# Initial Enumeration

After stabilizing the shell, basic enumeration is performed.

Current directory:

```
/opt/webapp
```

Listing the application files reveals:

```
index.html

server.js

package.json

package-lock.json

node_modules
```

This confirms that the target is a **Node.js** web application, matching the earlier observations made during the deserialization analysis.

At this stage we have successfully:

- Enumerated exposed services.
- Identified a Node.js web application.
- Intercepted and analyzed session cookies.
- Discovered insecure deserialization using `node-serialize`.
- Crafted a malicious serialized object.
- Achieved Remote Code Execution.
- Obtained a reverse shell as the **ubuntu** user.
- Stabilized the shell.
- Confirmed the application's file structure.

The next phase focuses on local enumeration, privilege escalation through unrestricted sudo access, capturing both user and root flags, MITRE ATT&CK mapping, security weaknesses, and defensive recommendations.

# Privilege Escalation

After obtaining a stable shell as the **ubuntu** user, the remaining objective is escalating privileges to **root**.

The first step is always performing basic local enumeration.

---

# Enumerating the Current User

Checking the current user.

```bash
whoami
```

Output:

```
ubuntu
```

Checking the user's group memberships.

```bash
id
```

Output:

```text
uid=1001(ubuntu)

gid=1002(ubuntu)

groups=

adm
dialout
cdrom
floppy
sudo
audio
dip
video
plugdev
lxd
netdev
```

Several interesting groups are present, including:

```
sudo
```

Membership in the sudo group often indicates potential privilege escalation opportunities.

---

# Enumerating Sudo Permissions

The next logical step is checking the user's sudo privileges.

```bash
sudo -l
```

Output:

```text
User ubuntu may run the following commands:

(ALL : ALL) ALL

(ALL) NOPASSWD: ALL
```

This immediately reveals the privilege escalation path.

The **ubuntu** user can execute **any command as root without supplying a password**.

---

# Understanding the Misconfiguration

Normally, privileged commands require authentication.

```
User

↓

sudo

↓

Password Prompt

↓

Root
```

On this machine:

```
User

↓

sudo

↓

Root
```

No password verification occurs.

This represents one of the most severe privilege escalation misconfigurations possible on Linux systems.

---

# Escalating to Root

Obtaining a root shell is straightforward.

```bash
sudo su
```

Immediately after execution, the shell changes.

Checking the current user.

```bash
whoami
```

Output:

```
root
```

The machine is now fully compromised.

---

# User Flag

Although root access has already been obtained, the user flag still needs to be collected.

Navigating to the home directory.

```bash
cd /home
```

Listing users.

```bash
ls
```

Output:

```
dylan

ubuntu
```

The user flag belongs to:

```
dylan
```

Navigating into the directory.

```bash
cd /home/dylan
```

Reading the flag.

```bash
cat user.txt
```

Output:

```
0ba48780dee9f5677a4461f588af217c
```

The first objective has now been completed.

---

# Root Flag

Since root access is already available, retrieving the final flag is straightforward.

Navigate to the root directory.

```bash
cd /root
```

Read the flag.

```bash
cat root.txt
```

Output:

```
2cd5a9fd3a0024bfa98d01d69241760e
```

The machine has now been fully compromised.

---

# Complete Attack Path

```
Internet
      │
      ▼
Nmap Enumeration
      │
      ▼
HTTP Service
      │
      ▼
Newsletter Form
      │
      ▼
Cookie Analysis
      │
      ▼
Base64 JSON Object
      │
      ▼
node-serialize
      │
      ▼
Insecure Deserialization
      │
      ▼
Remote Code Execution
      │
      ▼
Reverse Shell
      │
      ▼
ubuntu
      │
      ▼
sudo -l
      │
      ▼
NOPASSWD: ALL
      │
      ▼
Root Shell
      │
      ▼
User Flag
      │
      ▼
Root Flag
```

---

# Security Weaknesses Identified

## 1. Insecure Deserialization

The application trusted serialized objects stored inside a client-controlled cookie.

**Impact**

- Arbitrary JavaScript execution.
- Remote Code Execution.
- Complete server compromise.

**Mitigation**

- Never deserialize untrusted input.
- Replace unsafe serialization libraries.
- Validate and sign serialized data.

---

## 2. Client-Controlled Session Data

The session cookie contained serialized application data rather than a secure session identifier.

**Impact**

- Cookie tampering.
- Data manipulation.
- Code execution when combined with unsafe deserialization.

**Mitigation**

- Store sessions server-side.
- Use signed session identifiers.
- Protect cookies with integrity validation.

---

## 3. Vulnerable Node.js Package

The application relied on the vulnerable **node-serialize** package.

**Impact**

- Known Remote Code Execution vulnerability.
- Public exploit availability.

**Mitigation**

- Remove deprecated libraries.
- Regularly update dependencies.
- Perform dependency vulnerability scanning.

---

## 4. Excessive Sudo Permissions

The **ubuntu** user possessed unrestricted passwordless sudo access.

```
NOPASSWD: ALL
```

**Impact**

- Instant privilege escalation.
- Full system compromise.

**Mitigation**

- Apply the principle of least privilege.
- Limit sudo permissions to required commands.
- Require password authentication for privileged actions.

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Active Scanning | T1595 |
| Exploit Public-Facing Application | T1190 |
| Exploitation for Client Execution | T1203 |
| Command and Scripting Interpreter (Unix Shell) | T1059.004 |
| Ingress Tool Transfer | T1105 |
| Valid Accounts (sudo) | T1078 |
| Abuse Elevation Control Mechanism | T1548 |

---

# Detection Opportunities

Organizations can detect attacks similar to this by monitoring:

- Unexpected modifications to session cookies.
- Base64-encoded cookies containing serialized objects.
- Deserialization exceptions within application logs.
- Execution of child processes from Node.js applications.
- Outbound reverse shell connections.
- Unusual execution of `/bin/sh` by web services.
- Excessive or unexpected use of `sudo`.
- Passwordless privilege escalation events.

Web Application Firewalls (WAFs), Endpoint Detection and Response (EDR), and runtime application monitoring can provide visibility into these behaviors before full compromise occurs.

---

# Security Recommendations

## Eliminate Unsafe Deserialization

Applications should never deserialize data received directly from users without strict validation.

Safer alternatives include:

- JSON parsing without executable objects.
- Server-side session storage.
- Signed session tokens.

---

## Secure Dependency Management

Regularly review third-party packages for known vulnerabilities.

Automated tools such as dependency scanners can identify outdated or vulnerable libraries before deployment.

---

## Apply Least Privilege

Users should only receive permissions required for their operational responsibilities.

Granting unrestricted:

```
NOPASSWD: ALL
```

creates an unnecessary path to complete system compromise.

---

## Monitor Child Process Creation

Node.js applications rarely require spawning operating system shells.

Alerting on unexpected execution of:

- `/bin/sh`
- `bash`
- `nc`
- `python`

from web application processes can quickly identify exploitation attempts.

---

## Harden Session Management

Applications should:

- Store sessions server-side.
- Sign cookies using strong cryptographic secrets.
- Validate all session data.
- Reject modified session values.

---

# Lessons Learned

The **Jason** machine demonstrates how a seemingly harmless session cookie can become the entry point for complete system compromise. By trusting serialized objects supplied by the client, the application exposed itself to a well-known **Node.js insecure deserialization** vulnerability that allowed arbitrary JavaScript execution and Remote Code Execution.

The challenge also highlights the importance of understanding application logic during web assessments. Careful analysis of HTTP requests, cookies, and client-side behavior revealed the trust relationship between the browser and the server, ultimately leading to successful exploitation without requiring authentication or file uploads.

Finally, the privilege escalation stage reinforces a common Linux security lesson: even if an attacker gains only limited access initially, excessive sudo permissions can immediately transform a low-privileged compromise into full administrative control.

---

# Conclusion

**Jason** is an excellent beginner-friendly TryHackMe machine that introduces the dangers of insecure deserialization within Node.js applications. Through careful analysis of session cookies and application behavior, we identified the vulnerable **node-serialize** library, crafted a malicious serialized object, and achieved Remote Code Execution by injecting JavaScript into a trusted deserialization process.

Following initial access as the **ubuntu** user, local enumeration revealed unrestricted passwordless sudo privileges, allowing immediate escalation to **root** without exploiting additional vulnerabilities. The challenge emphasizes the risks of trusting client-controlled data, using outdated third-party libraries, and granting excessive administrative permissions.

Overall, Jason provides an excellent practical introduction to modern web exploitation, Node.js security, insecure deserialization, and Linux privilege escalation while reinforcing the importance of secure coding practices and the principle of least privilege.
