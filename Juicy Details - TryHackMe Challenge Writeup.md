# 🛡️ TryHackMe - Juice Shop Log Analysis Writeup

<p align="center">

<img src="https://img.shields.io/badge/TryHackMe-Juice%20Shop-red?style=for-the-badge">
<img src="https://img.shields.io/badge/Difficulty-Easy-success?style=for-the-badge">
<img src="https://img.shields.io/badge/Category-Defensive%20Security-blue?style=for-the-badge">
<img src="https://img.shields.io/badge/Type-Log%20Analysis-orange?style=for-the-badge">
<img src="https://img.shields.io/badge/Focus-SOC%20Investigation-brightgreen?style=for-the-badge">

</p>

---

# Overview

Today's challenge is another **Defensive Security** room from **TryHackMe** where the objective is to investigate server logs after a successful compromise of the Juice Shop application.

Instead of exploiting the target ourselves, we are placed in the role of a **Security Operations Center (SOC) Analyst** responsible for reconstructing the attack timeline using various server log files.

Throughout this investigation we identify:

- Reconnaissance performed by the attacker
- Tools used during the attack
- Web application vulnerabilities exploited
- Brute-force attempts
- SQL Injection attacks
- Sensitive information targeted
- Files accessed through vulnerable endpoints
- Services abused after compromise

This room provides practical experience in web log analysis while teaching how attackers leave traces behind even after successfully compromising a system.

---

# Scenario

You have recently joined one of the world's largest **Juice Shops** as a SOC Analyst.

An attacker has already compromised part of the environment and your responsibility is to investigate the incident.

The IT department has collected all relevant server logs and provided them for forensic analysis.

Your objectives include:

- Identify attacker activity
- Determine attack techniques
- Discover vulnerable endpoints
- Identify sensitive information accessed
- Reconstruct the attack timeline

---

# Investigation Objectives

During this investigation we will answer the following questions:

- What tools were used?
- Which endpoints were attacked?
- Which endpoint suffered brute-force attacks?
- Which endpoint was vulnerable to SQL Injection?
- Which parameter was injectable?
- Which files were targeted?
- Which services were abused?
- What information did the attacker attempt to steal?

---

# Evidence Collection

The challenge provides a ZIP archive containing multiple log files.

Move the archive into the working directory.

```bash
mv /home/kali/Desktop/logs_1618139984797.zip /home/kali/thm/juiceshop
```

Verify its presence.

```bash
ls
```

Output

```text
juiceshop.txt
logs_1618139984797.zip
```

Extract the archive.

```bash
unzip logs_1618139984797.zip
```

Output

```text
Archive: logs_1618139984797.zip

inflating: access.log
inflating: auth.log
inflating: vsftpd.log
```

Three important log files are recovered.

| File | Purpose |
|------|----------|
| access.log | Web server requests |
| auth.log | Authentication activity |
| vsftpd.log | FTP server activity |

Each of these logs provides evidence of a different stage of the intrusion.

---

# Initial Log Review

The investigation begins with **access.log** because it records every HTTP request made against the web application.

Large log files can be reviewed using:

```bash
less access.log
```

or

```bash
cat access.log
```

Searching for suspicious requests immediately reveals several automated tools through their **User-Agent** strings.

---

# Question 1

## What tools did the attacker use?

Carefully reviewing the log entries reveals several well-known offensive security tools.

Examples include:

```text
sqlmap
hydra
feroxbuster
nmap
curl
```

### Answer

```
sqlmap
hydra
feroxbuster
nmap
curl
```

---

## Analysis

The order of tools clearly illustrates the attack progression.

| Tool | Purpose |
|-------|----------|
| Nmap | Host and service discovery |
| Feroxbuster | Directory enumeration |
| Hydra | Credential brute forcing |
| SQLMap | SQL Injection exploitation |
| Curl | Manual HTTP requests |

Rather than randomly attacking the application, the attacker followed a logical penetration testing methodology.

---

# Question 2

## What endpoint was vulnerable to brute-force attacks?

Searching the access log for repeated login attempts quickly identifies the authentication endpoint.

```
/rest/user/login
```

Hydra repeatedly submitted authentication requests against this endpoint until valid credentials were discovered.

### Answer

```
/rest/user/login
```

---

## Analysis

Brute-force attacks can usually be identified by:

- High request volume
- Multiple failed logins
- Repeated POST requests
- Short time intervals
- Same source IP

These indicators clearly appear throughout the log file.

---

# Question 3

## What endpoint was vulnerable to SQL Injection?

Later in the log another suspicious request appears.

```
GET /rest/products/search
```

The query contains a UNION SELECT statement.

Example:

```text
/rest/products/search?q=qwert')) UNION SELECT id,email,password,'4','5','6','7','8','9' FROM Users--
```

The injected SQL syntax confirms successful exploitation of the search functionality.

### Answer

```
/rest/products/search
```

---

## Analysis

Several indicators immediately identify this request as SQL Injection.

```
UNION SELECT

FROM Users

email

password

--
```

These keywords almost never appear during legitimate application usage.

---

# Question 4

## Which parameter was injectable?

Looking closely at the vulnerable request reveals the injected parameter.

```
?q=
```

Everything after **q=** becomes part of the SQL query executed by the backend.

### Answer

```
q
```

---

## Why is this Vulnerable?

Instead of validating user input, the application directly inserts the parameter into the SQL statement.

An attacker can therefore modify the original database query.

Example payload:

```sql
UNION SELECT id,email,password FROM Users--
```

The attacker attempts to retrieve sensitive information directly from the Users table.

---

# Question 5

## Which endpoint did the attacker use to retrieve files?

Directory enumeration eventually discovers another interesting endpoint.

```
/ftp
```

The attacker continues requesting backup files stored inside this directory.

Example requests:

```text
GET /ftp HTTP/1.1
```

followed by

```text
GET /ftp/www-data.bak HTTP/1.1
```

Although access returns **403 Forbidden**, the repeated requests demonstrate the attacker's intention to retrieve backup files from the FTP directory.

### Answer

```
/ftp
```

---

## Analysis

The sequence of requests strongly suggests that the attacker had already identified a potentially sensitive directory before attempting to download backup files.

Observed requests include:

```text
/ftp

/ftp/www-data.bak
```

Backup files frequently contain:

- Passwords
- Database dumps
- Configuration files
- API keys
- Source code
- Credentials

Because of this, attackers commonly search for exposed **.bak**, **.zip**, **.old**, and **.backup** files during post-enumeration activities.

---

# Investigation Progress

At this stage the attack progression is becoming increasingly clear.

```
Nmap Scan
      │
      ▼
Directory Enumeration
      │
      ▼
Brute Force Login
      │
      ▼
SQL Injection
      │
      ▼
Sensitive Data Enumeration
      │
      ▼
Backup File Discovery
```

The next stage of the investigation focuses on determining **what data the attacker successfully accessed**, **whether the brute-force attack succeeded**, and **how the attacker later abused FTP and SSH services after compromising the web application**.

# Part 2 - Detailed Log Analysis

After identifying the initial attack surface and discovering the vulnerable endpoints, the next phase of the investigation focuses on understanding **what actions the attacker performed after gaining access**. By carefully reviewing the web server, FTP, and authentication logs, we can reconstruct the attacker's objectives and determine what sensitive information they attempted to obtain.

---

# Question 6

## What section of the website did the attacker use to scrape user email addresses?

Reviewing the **access.log** reveals a large number of repeated requests targeting the product review endpoint.

Example:

```text
GET /rest/products/1/reviews HTTP/1.1
```

This endpoint returns customer reviews that include user-related information such as email addresses.

The attacker repeatedly accessed this endpoint in a very short period, indicating automated scraping rather than normal browsing activity.

### Answer

```
/rest/products/1/reviews
```

---

## Analysis

Repeated requests to the same endpoint are common indicators of automated collection.

Example log entries:

```text
GET /rest/products/1/reviews

GET /rest/products/2/reviews

GET /rest/products/3/reviews
```

Since review sections often expose usernames or email addresses, attackers frequently scrape them before launching credential attacks.

Possible attacker objectives included:

- Collecting user email addresses
- Building a target list
- Password spraying
- Credential stuffing
- Social engineering

---

# Question 7

## Was the brute-force attack successful?

After reviewing requests against the login endpoint, authentication logs reveal that one login attempt eventually succeeds.

Timestamp:

```text
11/Apr/2021:09:16:31 +0000
```

### Answer

```
Yay

11/Apr/2021:09:16:31 +0000
```

---

## Analysis

The sequence follows a typical brute-force attack pattern.

```
Multiple failed logins
        │
        ▼
Repeated authentication requests
        │
        ▼
Valid credentials discovered
        │
        ▼
Successful login
```

This confirms that Hydra was ultimately able to identify valid credentials.

---

# Question 8

## What information did the attacker attempt to retrieve through SQL Injection?

Later in the logs a UNION-based SQL Injection payload appears.

```text
GET /rest/products/search?q=qwert')) UNION SELECT
id,
email,
password,
'4',
'5',
'6',
'7',
'8',
'9'
FROM Users--
```

The attacker specifically targets the **Users** table.

Two database columns are clearly requested.

```
email

password
```

### Answer

```
email

password
```

---

## Analysis

Rather than dumping the entire database, the attacker selectively targets authentication data.

The SQL payload requests:

```sql
SELECT
id,
email,
password
FROM Users
```

Recovering hashed passwords and associated email addresses provides valuable information for:

- Account takeover
- Password cracking
- Credential reuse
- Privilege escalation
- Future phishing campaigns

---

# Question 9

## Which files did the attacker attempt to download?

After discovering the **/ftp** endpoint, the attacker attempts to retrieve several backup files.

Example requests:

```text
GET /ftp/www-data.bak
```

and

```text
GET /ftp/coupons_2013.md.bak
```

Although the server responds with **403 Forbidden**, the requests clearly indicate which files were being targeted.

### Answer

```
www-data.bak

coupons_2013.md.bak
```

---

## Analysis

Backup files are common targets because administrators frequently leave sensitive copies accessible on production servers.

Examples include:

```
config.php.bak

database.sql

backup.zip

.env.old

www-data.bak
```

These files often expose:

- Passwords
- Database credentials
- API tokens
- Source code
- Customer information

Even unsuccessful requests provide valuable insight into the attacker's intentions.

---

# Question 10

## Which service and account were used to retrieve files?

The **vsftpd.log** provides evidence of FTP activity.

Example:

```text
Sun Apr 11 09:08:34 2021

OK LOGIN

Client "::ffff:192.168.10.5"

anon password "IEUser@"
```

Multiple successful anonymous logins are recorded.

### Answer

| Service | Username |
|----------|----------|
| FTP | anon |

---

## Analysis

The FTP server allowed anonymous authentication.

This enabled the attacker to browse files without requiring valid user credentials.

Relevant log entries:

```text
OK LOGIN

anon

CONNECT
```

Anonymous FTP access significantly increases the risk of information disclosure if sensitive files are accidentally exposed.

---

# Question 11

## Which additional service was successfully accessed?

Later in **auth.log**, another successful authentication event appears.

```text
Accepted password for www-data
```

Example:

```text
Apr 11 09:41:32

Accepted password

for www-data

from 192.168.10.5
```

Immediately afterwards:

```text
session opened for user www-data
```

This confirms successful SSH authentication.

### Answer

| Service | Account |
|----------|----------|
| SSH | www-data |

---

## Analysis

This event represents the final stage of the compromise.

Authentication sequence:

```text
Accepted password

Session opened

www-data
```

Unlike the earlier brute-force attempts, this authentication uses valid credentials, indicating the attacker had already obtained the password through previous stages of the attack.

The likely progression is:

```
Email Enumeration
        │
        ▼
Brute Force Login
        │
        ▼
SQL Injection
        │
        ▼
Credential Discovery
        │
        ▼
FTP Enumeration
        │
        ▼
SSH Login
```

---

# Reconstructed Attack Timeline

The complete intrusion can now be reconstructed chronologically.

| Time | Activity |
|------|----------|
| Initial | Network reconnaissance using Nmap |
| Phase 1 | Directory enumeration with Feroxbuster |
| Phase 2 | User email scraping from product reviews |
| Phase 3 | Hydra brute-force attack against `/rest/user/login` |
| Phase 4 | Successful login obtained |
| Phase 5 | SQL Injection against `/rest/products/search` |
| Phase 6 | Extraction of user emails and passwords |
| Phase 7 | Enumeration of FTP resources |
| Phase 8 | Attempts to download backup files |
| Final | SSH login as `www-data` |

---

At this point the investigation has successfully reconstructed the complete attack sequence from initial reconnaissance through post-compromise access. The final section summarizes the incident with MITRE ATT&CK mappings, Indicators of Compromise (IOCs), detection opportunities, defensive recommendations, and the overall conclusions drawn from the forensic analysis.

# Part 3 - Incident Summary & Defensive Analysis

The investigation is now complete. By correlating the evidence from **access.log**, **auth.log**, and **vsftpd.log**, it is possible to reconstruct the attacker's entire kill chain from reconnaissance to post-compromise access.

Although multiple vulnerabilities existed within the application, the attacker followed a structured penetration testing methodology rather than exploiting random weaknesses. Each successful phase provided additional information that was leveraged during the following stage of the attack.

---

# Complete Attack Chain

```text
Internet
    │
    ▼
Network Reconnaissance
(Nmap)
    │
    ▼
Directory Enumeration
(Feroxbuster)
    │
    ▼
Scraping Product Reviews
(User Email Collection)
    │
    ▼
Hydra Brute Force
(/rest/user/login)
    │
    ▼
Successful Authentication
    │
    ▼
SQL Injection
(/rest/products/search)
    │
    ▼
Extract Users Table
(email + password)
    │
    ▼
FTP Enumeration
(/ftp)
    │
    ▼
Attempted Backup File Downloads
(www-data.bak)
(coupons_2013.md.bak)
    │
    ▼
SSH Login
(www-data)
```

---

# Tools Used by the Attacker

The investigation identified several offensive security tools used throughout the intrusion.

| Tool | Purpose |
|-------|----------|
| Nmap | Network and service enumeration |
| Feroxbuster | Directory and endpoint discovery |
| Hydra | Credential brute-force attack |
| SQLMap | Automated SQL Injection exploitation |
| Curl | Manual HTTP requests and testing |

---

# Vulnerabilities Identified

During the investigation several weaknesses within the environment became evident.

| Vulnerability | Risk |
|--------------|------|
| User enumeration through product reviews | Email harvesting |
| Weak login protection | Successful brute-force attack |
| SQL Injection | Database compromise |
| Anonymous FTP | File disclosure |
| Exposed backup files | Sensitive information leakage |
| Password reuse | SSH compromise |

---

# Sensitive Information Targeted

The attacker specifically attempted to retrieve sensitive information rather than performing random attacks.

The investigation identified attempts to access:

- User email addresses
- User passwords
- Backup files
- FTP resources
- SSH credentials
- Application data

The SQL Injection payload clearly demonstrates the attacker's objective.

```sql
SELECT
id,
email,
password
FROM Users
```

This confirms that credential theft was one of the primary goals of the attack.

---

# Indicators of Compromise (IOCs)

## Source IP

```text
192.168.10.5
```

---

## User Agents Observed

```text
sqlmap

Hydra

Feroxbuster

Nmap

curl
```

---

## Targeted Endpoints

```text
/rest/user/login

/rest/products/search

/rest/products/1/reviews

/ftp
```

---

## Targeted Files

```text
www-data.bak

coupons_2013.md.bak
```

---

## Services Accessed

```text
HTTP

FTP

SSH
```

---

# MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---------|-----------|----|
| Reconnaissance | Active Scanning | T1595 |
| Reconnaissance | Gather Victim Network Information | T1590 |
| Discovery | Network Service Discovery | T1046 |
| Credential Access | Brute Force | T1110 |
| Credential Access | Credentials from Password Stores | T1555 |
| Credential Access | Unsecured Credentials | T1552 |
| Initial Access | Exploit Public-Facing Application | T1190 |
| Collection | Data from Information Repositories | T1213 |
| Collection | Archive Collected Data | T1560 |
| Exfiltration | Exfiltration Over Web Service | T1567 |
| Lateral Movement | Valid Accounts | T1078 |

---

# Detection Opportunities

A SOC team could have detected this intrusion much earlier by monitoring abnormal behavior within the web and authentication logs.

Potential detection opportunities include:

### Excessive Login Failures

Indicators:

- Hundreds of failed POST requests
- Rapid authentication attempts
- Single source IP
- Multiple usernames

Possible detection:

```text
Repeated POST requests to

/rest/user/login
```

---

### SQL Injection Detection

Indicators:

```text
UNION SELECT

FROM Users

--

'
```

These keywords are rarely observed during legitimate web traffic.

A WAF or IDS could easily generate alerts for these payloads.

---

### Directory Enumeration

Indicators:

```text
GET /ftp

GET /robots.txt

GET /admin

GET /backup
```

Large numbers of sequential requests often indicate automated scanning.

---

### Anonymous FTP Access

Indicators:

```text
OK LOGIN

anon

CONNECT
```

Anonymous authentication should generate security alerts in production environments.

---

### SSH Login Monitoring

Indicators:

```text
Accepted password

www-data
```

Application service accounts should rarely authenticate through SSH.

Unexpected interactive logins should immediately trigger investigations.

---

# Defensive Recommendations

Based on the findings, several security improvements should be implemented.

## Web Application Security

- Validate all user input.
- Use prepared SQL statements.
- Deploy a Web Application Firewall (WAF).
- Perform regular vulnerability assessments.

---

## Authentication Security

- Implement account lockout policies.
- Enable rate limiting.
- Enforce strong password policies.
- Require Multi-Factor Authentication (MFA).

---

## FTP Security

- Disable anonymous FTP access.
- Restrict file permissions.
- Remove unnecessary backup files.
- Monitor file downloads.

---

## SSH Security

- Disable password authentication where possible.
- Use SSH key authentication.
- Restrict service account logins.
- Monitor privileged sessions.

---

## Monitoring Improvements

The SOC should continuously monitor for:

- Brute-force attacks
- SQL Injection attempts
- Excessive HTTP requests
- Suspicious User-Agent strings
- Large file downloads
- Anonymous FTP sessions
- Service account logins

---

# Lessons Learned

This room demonstrates how valuable log analysis is during incident response.

Although the attacker successfully compromised multiple services, every stage of the intrusion left identifiable evidence across different log files. By correlating web server logs, FTP logs, and authentication logs, it becomes possible to reconstruct the complete attack timeline, identify exploited vulnerabilities, determine the tools used, and understand exactly what information the attacker attempted to steal.

The challenge also highlights several common web application security issues, including insufficient input validation, weak authentication controls, exposed backup files, anonymous FTP access, and poor monitoring of critical services. Addressing these weaknesses significantly reduces the likelihood of successful compromise.

---

# Conclusion

The Juice Shop Log Analysis room provides an excellent introduction to **SOC investigations and web application forensic analysis**. Rather than focusing on exploitation, the challenge emphasizes understanding attacker behavior through log evidence and teaches how defenders can identify malicious activity after an incident has occurred.

During this investigation, the attacker was observed performing network reconnaissance, directory enumeration, user enumeration, brute-force authentication, SQL Injection, FTP abuse, and SSH access. Each stage of the attack generated artifacts that allowed the complete intrusion to be reconstructed from the available logs.

Overall, this room reinforces the importance of centralized logging, continuous monitoring, secure application development, and proactive threat detection. It serves as a practical exercise for aspiring SOC analysts and incident responders by demonstrating how seemingly isolated log entries can be combined into a comprehensive incident timeline and used to improve an organization's defensive posture.

---

## Investigation Summary

| Category | Result |
|----------|--------|
| Platform | TryHackMe |
| Room Type | Defensive Security |
| Difficulty | Easy |
| Focus | Log Analysis |
| Evidence Reviewed | access.log, auth.log, vsftpd.log |
| Attack Techniques | Brute Force, SQL Injection, Directory Enumeration, FTP Abuse |
| Compromised Services | HTTP, FTP, SSH |
| Outcome | Full Attack Timeline Successfully Reconstructed |

---

<p align="center">

**🛡️ Investigation Complete**

**Status:** Successfully Analyzed ✔️

</p>
