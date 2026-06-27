# TryHackMe - Static Malware Analysis Writeup

## Overview

Today we have another **TryHackMe Defensive Security** challenge where we are provided with a malware sample and supporting artifacts. Our objective is to investigate the incident by performing **network traffic analysis** and **static malware analysis**, answer the challenge questions, and understand the complete infection chain.

---

# Scenario

The scenario provided is:

> Jim from the Finance department received an email appearing to come from the company's system administrator requesting him to execute a PowerShell script to apply critical security updates. Trusting the email, Jim executed the script. Shortly afterward, unusual network activity and suspicious system behavior were detected. We have been provided with the malware artifacts to determine exactly what happened, identify the impact, and understand how the attacker established control over the compromised system.

---

# Environment

For this investigation the malware was analyzed inside **Flare-VM**, which provides a safe environment with malware analysis tools pre-installed.

Tools used during this investigation:

* Wireshark
* Flare-VM
* dnSpy
* Ghidra
* Python
* PowerShell
* Hash utilities

---

# Initial Investigation

The downloaded ZIP archive was extracted inside the isolated malware analysis environment.

The provided artifacts mainly consisted of:

* PowerShell script
* Network capture (.pcap)
* Second-stage payload (amd.bin)

Since the challenge focuses on both network traffic and malware behavior, the first step was to inspect the packet capture.

---

# PCAP Analysis

Opening the PCAP inside **Wireshark**, the first step was filtering HTTP traffic.

```
http
```

Almost immediately an outbound HTTP request appears.

---

## Q1. What external domain was contacted during script execution?

The very first HTTP request shows the victim downloading an external file.

```
GET /amd.bin HTTP/1.1

Host:
api-edgecloud.xyz
```

Full URI:

```
http://api-edgecloud.xyz/amd.bin
```

### Answer

```
api-edgecloud.xyz
```

### Analysis

The PowerShell script contacts the external domain **api-edgecloud.xyz** to retrieve a second-stage payload named **amd.bin**.

Instead of embedding malware directly inside the script, the attacker separates the infection into multiple stages, making the initial script smaller and harder to detect.

---

# Static PowerShell Analysis

Opening the PowerShell script reveals several suspicious behaviors.

The script performs four major actions:

1. Downloads an encrypted payload
2. Decrypts it
3. Saves it to disk
4. Executes it

Rather than downloading a PE executable directly, it downloads an encrypted blob named:

```
amd.bin
```

The downloaded content is decrypted in memory before being written to disk.

---

## Q2. What encryption algorithm was used?

Looking deeper into the script reveals a manual implementation of the **RC4 stream cipher**.

The script contains:

* Key Scheduling Algorithm (KSA)
* Pseudo Random Generation Algorithm (PRGA)

These are the two fundamental components of RC4.

The script:

* Builds a permutation array
* Generates the RC4 keystream
* XORs every byte of the downloaded payload
* Produces the decrypted executable

### Answer

```
RC4
```

### Why RC4?

Attackers commonly use RC4 because it is:

* lightweight
* simple to implement
* fast
* capable of hiding payloads from simple antivirus signatures

Although RC4 is considered cryptographically weak today, it is still frequently encountered in malware.

---

## Q3. What key was used?

The PowerShell script stores the key as multiple concatenated strings.

```
$k = [System.Text.Encoding]::UTF8.GetBytes(

('X9vT3pL'+'2QwE'+'8xR6'+'ZkYhC4'+'s')

)
```

After concatenation the RC4 key becomes:

```
X9vT3pL2QwE8xR6ZkYhC4s
```

### Answer

```
X9vT3pL2QwE8xR6ZkYhC4s
```

---

# Payload Drop

Once RC4 decryption finishes, the script writes the executable to the victim's temporary directory.

```
$env:TEMP\amdfendrsr.exe
```

The filename is intentionally chosen to resemble an AMD-related executable.

This naming technique attempts to blend in with legitimate software and reduce suspicion.

The script then launches the executable using:

```
Start-Process
```

This completes the first stage of the infection.

---

## Q4. What was the timestamp of the server response?

Looking at the HTTP response containing **amd.bin**, the server returns the following timestamp.

```
Fri, 10 Apr 2026 05:28:23 GMT
```

### Answer

```
Fri, 10 Apr 2026 05:28:23 GMT
```

---

# Decrypting the Second Stage

The downloaded file itself is not executable.

Instead it contains encrypted hexadecimal data.

A small Python script was written to reproduce the RC4 algorithm and decrypt the payload.

```python
from pathlib import Path
import hashlib

hex_data = Path("amd.bin").read_text().strip()
enc = bytes.fromhex(hex_data)

key = b"X9vT3pL2QwE8xR6ZkYhC4s"

def rc4(key,data):
    s=list(range(256))
    j=0

    for i in range(256):
        j=(j+s[i]+key[i%len(key)])%256
        s[i],s[j]=s[j],s[i]

    i=j=0
    out=bytearray()

    for b in data:
        i=(i+1)%256
        j=(j+s[i])%256
        s[i],s[j]=s[j],s[i]
        out.append(b ^ s[(s[i]+s[j])%256])

    return bytes(out)

dec=rc4(key,enc)

Path("amdfendrsr.exe").write_bytes(dec)

print(hashlib.sha256(dec).hexdigest())
```

The script successfully reconstructs the malware executable.

---

## Q5. SHA-256 of the decrypted payload

After decryption, the resulting executable has the following SHA-256 hash.

```
e3d39d42df63c6874780737244370ba517820f598fd2443e47ff6580f10c17cb
```

### Answer

```
e3d39d42df63c6874780737244370ba517820f598fd2443e47ff6580f10c17cb
```

---

# C2 Communication

Continuing the Wireshark analysis reveals another HTTP request after the malware executes.

```
GET /images?guid=...
```

Host:

```
34.174.57.99
```

Full URI:

```
http://34.174.57.99/images?guid=UVRNZUdTS0ozRzRaUGNlaGhLRUd0aXl2R3Zub2N2YW5UZTh3ZmorMEx1WXBPdEIvL1BSSzZ1RW5oN0EvMTIxRUJ3Z3NQZk5Yb2d0VUYxOTV0MFZ1SUZDZ1cwcnRHYzlCYlFLK1NzTWd1NVE9
```

Unlike the first request, this communication is no longer downloading malware.

Instead, this is the malware communicating with its Command and Control (C2) server.

---

## Q6. What remote URL did the client use?

### Answer

```
http://34.174.57.99/images?guid=UVRNZUdTS0ozRzRaUGNlaGhLRUd0aXl2R3Zub2N2YW5UZTh3ZmorMEx1WXBPdEIvL1BSSzZ1RW5oN0EvMTIxRUJ3Z3NQZk5Yb2d0VUYxOTV0MFZ1SUZDZ1cwcnRHYzlCYlFLK1NzTWd1NVE9
```

---

# Static Analysis of the Executable

The decrypted executable is a **.NET application**, making decompilation relatively straightforward.

The executable was opened using **dnSpy**.

The application's entry point is:

```
TrevorC2Client.Main()
```

This immediately identifies the malware as a **TrevorC2** client.

TrevorC2 is a PowerShell/.NET based Command and Control framework that disguises C2 traffic as legitimate HTTP requests.

Instead of obvious malware traffic, TrevorC2 hides commands inside normal-looking web requests, making network detection much more difficult.

---

## Q7. Which encryption algorithm and key does the client use?

Inspecting the TrevorC2 source code inside dnSpy reveals the client encrypts communications using **AES**.

Encryption Key:

```
M4squ3r4d3Th3P4ck3tSt34lthM0d31337
```

Algorithm:

```
AES
```

### Answer

```
AES

Key:
M4squ3r4d3Th3P4ck3tSt34lthM0d31337
```

---

# Q8. Decrypting the Attacker Commands

Once the AES configuration was identified inside dnSpy, the encrypted commands exchanged with the Command and Control server could be analyzed.

Using **Ghidra**, the encrypted data and malware logic were further inspected to understand how TrevorC2 processes incoming commands.

After decrypting the attacker instructions, the hidden flag can be recovered from the malware communication.

---

# Indicators of Compromise (IOCs)

| Type                     | Value                                                            |
| ------------------------ | ---------------------------------------------------------------- |
| Domain                   | api-edgecloud.xyz                                                |
| Downloaded File          | amd.bin                                                          |
| Dropped Executable       | amdfendrsr.exe                                                   |
| RC4 Key                  | X9vT3pL2QwE8xR6ZkYhC4s                                           |
| SHA-256                  | e3d39d42df63c6874780737244370ba517820f598fd2443e47ff6580f10c17cb |
| C2 IP                    | 34.174.57.99                                                     |
| C2 URI                   | /images?guid=...                                                 |
| Malware Family           | TrevorC2 Client                                                  |
| Communication Encryption | AES                                                              |

---

# Attack Flow

```
Phishing Email
        │
        ▼
PowerShell Script Executed
        │
        ▼
Download amd.bin
(api-edgecloud.xyz)
        │
        ▼
RC4 Decryption
        │
        ▼
Drop amdfendrsr.exe
        │
        ▼
Execute Payload
        │
        ▼
TrevorC2 Client Starts
        │
        ▼
AES Encrypted HTTP Communication
        │
        ▼
Command & Control Server
```

---

# MITRE ATT&CK Mapping

| Technique                         | ID        |
| --------------------------------- | --------- |
| Spearphishing Attachment          | T1566.001 |
| PowerShell                        | T1059.001 |
| Ingress Tool Transfer             | T1105     |
| Command and Scripting Interpreter | T1059     |
| Deobfuscate/Decode Files          | T1140     |
| Encrypted Channel                 | T1573     |
| Application Layer Protocol        | T1071.001 |

---

# Lessons Learned

This investigation demonstrates a classic **multi-stage malware infection**. Rather than downloading an executable directly, the attacker used a PowerShell script to retrieve an RC4-encrypted payload from an external domain. After decryption, the payload was written to disk as **amdfendrsr.exe** and executed. Static analysis showed that the payload was a **TrevorC2** client, which established encrypted HTTP communication using **AES** to receive attacker commands.

The layered design of the malware highlights how modern threats combine scripting, encryption, staged payload delivery, and covert C2 communications to evade detection. By combining **network traffic analysis**, **PowerShell code review**, and **.NET reverse engineering**, the complete infection chain could be reconstructed and all key indicators of compromise identified.

---

# Conclusion

This investigation successfully reconstructed the entire malware execution chain, beginning with a phishing email and ending with an active TrevorC2 Command and Control session. Network analysis identified the external infrastructure, PowerShell analysis revealed the RC4 decryption routine, Python was used to recover the second-stage payload, and dnSpy confirmed the malware's AES-based C2 implementation. Together, these techniques provided a complete understanding of the malware's behavior and the methods used by the attacker to gain and maintain control over the compromised system.
