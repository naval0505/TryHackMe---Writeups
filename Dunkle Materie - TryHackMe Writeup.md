# DUNKLEMATERIAL - TryHackMe DFIR Investigation Writeup

## Overview

In this TryHackMe Defensive Security challenge, we investigate a suspected ransomware infection affecting a machine belonging to the Sales department.

The Security Operations Center (SOC) initially detected suspicious outbound network traffic from a host containing customer information. Firewall alerts indicated communications with malicious domains and the transfer of Base64-encoded data over the network.

The Incident Response team collected:

* Process Monitor (Procmon) logs
* Network traffic captures (PCAP)
* System activity artifacts

Upon reviewing the host, analysts discovered that the system had been compromised by ransomware.

The objective of this investigation is to identify Indicators of Compromise (IOCs), determine how the ransomware operated, and ultimately identify the ransomware family responsible for the attack.

---

# Investigation Environment

Tools used during analysis:

* ProcDOT
* Process Monitor (Procmon)
* Wireshark
* VirusTotal

Artifacts provided:

* Procmon CSV log
* Network Traffic PCAP file

---

# Initial Analysis

The first step was importing the provided Procmon CSV file into ProcDOT.

ProcDOT provides a graphical representation of process activity and network communications, making it easier to identify suspicious behavior.

After loading the Procmon CSV and the PCAP file into ProcDOT, I began reviewing the process tree and looking for suspicious executables.

---

# Question 1

## Provide the two PIDs spawned from the malicious executable.

While reviewing the process tree, I identified a suspicious executable named:

```text id="a1"
exploreer.exe
```

The filename closely resembles the legitimate Windows process:

```text id="a2"
explorer.exe
```

This is a common malware technique known as masquerading.

The process spawned two child processes.

### Answer

```text id="a3"
8644,7128
```

---

# Question 2

## Provide the full path where the ransomware initially got executed.

After identifying the malicious process, I investigated its execution path through ProcDOT and Procmon activity.

The executable was launched from the user's temporary directory.

### Answer

```text id="a4"
C:\Users\sales\AppData\Local\Temp\exploreer.exe
```

The use of the Temp directory is a strong indicator of malicious activity because legitimate applications rarely execute ransomware components from temporary folders.

---

# Question 3

## What are the two C2 domains?

Next, I examined network communications associated with the ransomware process.

ProcDOT highlighted outbound connections to suspicious domains.

These domains received HTTP POST requests containing encoded information from the infected host.

### Answer

```text id="a5"
mojobiden.com,paymenthacks.com
```

These domains acted as Command and Control (C2) infrastructure for the ransomware operation.

---

# Question 4

## What are the IP addresses of the malicious domains?

Using DNS resolutions and network activity observed within the investigation tools, the associated IP addresses were identified.

### Answer

```text id="a6"
146.112.61.108

206.188.197.206
```

These IPs can be added to threat intelligence feeds and detection systems for future monitoring.

---

# Question 5

## Provide the User-Agent used to transfer encrypted data.

To determine how the ransomware communicated with its C2 infrastructure, I opened the provided PCAP file in Wireshark.

Filtering on HTTP traffic revealed outbound POST requests sent to the malicious infrastructure.

Reviewing the HTTP headers showed the User-Agent used during communication.

### Answer

```text id="a7"
Firefox/89.0
```

The malware attempted to appear as legitimate browser traffic to avoid detection.

---

# Question 6

## Provide the cloud security service that blocked the malicious domain.

While examining the HTTP communications, the responses indicated intervention from a cloud-based security service.

The service responsible for blocking access to the malicious domain was:

### Answer

```text id="a8"
Cisco Umbrella
```

Cisco Umbrella provides DNS-layer protection and can block communications to known malicious domains.

---

# Question 7

## Provide the name of the bitmap used as the desktop wallpaper.

One of the common behaviors of ransomware is modifying the victim's desktop wallpaper.

This is often done to display ransom instructions or reinforce the attack's impact.

Reviewing ProcDOT activity revealed the bitmap file used during the wallpaper modification process.

### Answer

```text id="a9"
bg.bmp
```

The bitmap file served as the new desktop background after encryption was completed.

---

# Question 8

## Find the PID that attempted to change the wallpaper.

After identifying the wallpaper modification activity, I reviewed the process responsible for writing and applying the bitmap.

### Answer

```text id="a10"
4892
```

This process was responsible for changing the victim's desktop wallpaper.

---

# Question 9

## Registry key path of the mounted drive.

The ransomware mounted an additional drive and assigned it a drive letter.

Reviewing registry modifications revealed the associated key.

### Answer

```text id="a11"
HKLM\SYSTEM\MountedDevices\DosDevices\Z:
```

This registry entry indicates that the ransomware created or interacted with a mounted volume identified as drive Z:.

---

# Question 10

## Identify the ransomware family.

At this stage, several Indicators of Compromise had been collected:

### Domains

```text id="a12"
mojobiden.com
paymenthacks.com
```

### IP Addresses

```text id="a13"
146.112.61.108
206.188.197.206
```

### Malware Executable

```text id="a14"
exploreer.exe
```

Using the collected indicators for external threat intelligence research, particularly VirusTotal and related malware reports, the ransomware family was identified.

### Answer

```text id="a15"
Dharma Ransomware
```

---

# Indicators of Compromise (IOCs)

## Malicious Executable

```text id="a16"
exploreer.exe
```

## Execution Path

```text id="a17"
C:\Users\sales\AppData\Local\Temp\exploreer.exe
```

## Domains

```text id="a18"
mojobiden.com
paymenthacks.com
```

## IP Addresses

```text id="a19"
146.112.61.108
206.188.197.206
```

## User-Agent

```text id="a20"
Firefox/89.0
```

## Registry Artifact

```text id="a21"
HKLM\SYSTEM\MountedDevices\DosDevices\Z:
```

---

# Attack Summary

The ransomware executable masqueraded as a legitimate Windows process by using the filename:

```text id="a22"
exploreer.exe
```

After execution from the Temp directory, the malware:

1. Spawned additional processes.
2. Communicated with remote command-and-control infrastructure.
3. Sent information about the infected system through HTTP POST requests.
4. Mounted an additional drive.
5. Modified the victim's desktop wallpaper.
6. Displayed ransom-related artifacts.
7. Completed the ransomware infection cycle.

The collected indicators and threat intelligence research linked the activity to the Dharma ransomware family.

---

# Conclusion

This investigation demonstrates a typical ransomware incident response workflow using process monitoring and network traffic analysis. By correlating process execution, network communications, registry modifications, and external threat intelligence, it was possible to identify the ransomware family and extract multiple Indicators of Compromise.

The challenge highlights the importance of combining host-based and network-based evidence during DFIR investigations and provides practical experience in ransomware triage and malware attribution.
