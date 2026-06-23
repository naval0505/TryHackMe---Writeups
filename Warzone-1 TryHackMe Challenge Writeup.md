# Warzone 1 - TryHackMe Walkthrough

Today we have another TryHackMe challenge named **Warzone 1**. This room focuses on network traffic analysis, IDS/IPS alerts, malware command-and-control activity, and threat intelligence investigation.

In this scenario, we are acting as a Tier 1 SOC Analyst working for an MSSP. During our shift, an IDS alert is generated indicating potentially malicious traffic and possible malware command-and-control communications.

Our task is to investigate the provided PCAP file, validate the alert, identify the malware involved, trace the attack infrastructure, and recover the artifacts associated with the infection.

---

# Scenario

You work as a Tier 1 Security Analyst (L1) for a Managed Security Service Provider (MSSP).

A network alert has been triggered:

```text
Potentially Bad Traffic
Malware Command and Control Activity Detected
```

The objective is to determine whether the alert is a true positive by analyzing the supplied packet capture file.

---

# Evidence Provided

```text
zone1.pcap
```

The primary tools used during the investigation:

```text
Brim
Wireshark
VirusTotal
CyberChef
```

---

# Initial Investigation

The first step was loading the PCAP file into Brim and reviewing IDS alerts.

Using the query:

```text
alert.category == "Malware Command and Control Detected"
```

A suspicious alert immediately appeared.

---

# Question 1

## What was the alert signature?

Reviewing the IDS alert entry revealed:

```text
ET MALWARE MirrorBlast CnC Activity M3
```

### Answer

```text
ET MALWARE MirrorBlast CnC Activity M3
```

This confirms that the IDS identified activity associated with the MirrorBlast malware family.

---

# Question 2

## What was the source IP address?

The alert showed the infected internal host.

Source IP:

```text
172.16.1.102
```

Defanged format:

```text
172[.]16[.]1[.]102
```

### Answer

```text
172[.]16[.]1[.]102
```

This appears to be the compromised workstation initiating the malicious traffic.

---

# Question 3

## What was the destination IP address?

The IDS alert also revealed the destination.

Destination IP:

```text
169.239.128.11
```

Defanged:

```text
169[.]239[.]128[.]11
```

### Answer

```text
169[.]239[.]128[.]11
```

This is the external system contacted by the malware.

---

# Threat Intelligence Investigation

With the destination IP identified, the next step was to investigate the infrastructure using VirusTotal.

Searching:

```text
169.239.128.11
```

provided additional intelligence.

---

# Question 4

## What threat group is associated with the IP?

Under the Community section of VirusTotal:

```text
Microsoft Themed TA505 malicious domains
```

was referenced.

### Answer

```text
TA505
```

TA505 is a well-known financially motivated threat group frequently associated with malware distribution campaigns.

---

# Question 5

## What malware family was involved?

Further VirusTotal analysis and community comments revealed:

```text
MirrorBlast
```

### Answer

```text
MirrorBlast
```

This matched the original IDS signature observed earlier.

---

# Infrastructure Analysis

Now that the malware family was identified, the investigation moved toward identifying additional infrastructure associated with the campaign.

---

# Question 6

## What was the majority file type listed under Communicating Files?

Reviewing VirusTotal communicating files associated with the malicious domain showed:

```text
Windows Installer
```

### Answer

```text
Windows Installer
```

This indicates the malware was likely distributed through MSI installer packages.

---

# HTTP Traffic Analysis

Next, I pivoted back into the PCAP to inspect the HTTP traffic associated with the suspicious destination.

Filtering for the malicious destination IP:

```text
169.239.128.11
```

revealed beaconing activity.

---

# Question 7

## What was the User-Agent?

Examining the HTTP request:

```text
GET /r?x=...
```

The request contained:

```text
User-Agent: REBOL View 2.7.8.3.1
```

### Answer

```text
REBOL View 2.7.8.3.1
```

This User-Agent is unusual and stands out from normal browser traffic, making it a useful indicator of compromise.

---

# Malware Beacon Analysis

The HTTP request contained:

```text
Host: fidufagios.com
```

and a Base64-encoded parameter:

```text
/r?x=
```

This suggests the malware was transmitting victim information to its command-and-control infrastructure.

Observed attack flow:

```text
Victim Host
      ↓
Beacon Request
      ↓
Malicious Domain
      ↓
Command and Control Infrastructure
```

---

# Question 8

## What other IP addresses were involved?

Using the filter:

```text
(http) && (ip.src == 172.16.1.102)
```

additional external IP addresses were identified.

Defanged format:

```text
185[.]10[.]68[.]235
192[.]36[.]27[.]92
```

### Answer

```text
185[.]10[.]68[.]235,192[.]36[.]27[.]92
```

These IPs appear to be additional infrastructure used during the attack.

---

# Question 9

## What files were downloaded?

Investigating the HTTP sessions associated with the identified IPs revealed MSI downloads.

Recovered filenames:

```text
filter.msi
10opd3r_load.msi
```

### Answer

```text
filter.msi,10opd3r_load.msi
```

This confirms the attacker delivered malware using MSI installation packages.

---

# Question 10

## What files were written after the first MSI download?

Following the TCP stream associated with:

```text
filter.msi
```

revealed filesystem activity.

Recovered paths:

```text
C:\ProgramData\001\arab.bin
C:\ProgramData\001\arab.exe
```

### Answer

```text
C:\ProgramData\001\arab.bin,C:\ProgramData\001\arab.exe
```

The malware drops both a binary payload and an executable into the same directory.

---

# Additional Payload Analysis

The second MSI download followed a similar pattern.

By following the TCP stream and reviewing the payload contents, additional file creation activity was observed.

Searching for:

```text
C:\
```

within the stream quickly revealed the destination paths used by the malware.

This behavior is consistent with staged malware deployment where installer packages extract additional components before execution.

---

# Indicators of Compromise (IOCs)

## Infected Host

```text
172[.]16[.]1[.]102
```

## Command and Control IP

```text
169[.]239[.]128[.]11
```

## Additional Infrastructure

```text
185[.]10[.]68[.]235
192[.]36[.]27[.]92
```

## Malicious Domain

```text
fidufagios.com
```

## Malware Family

```text
MirrorBlast
```

## Threat Group

```text
TA505
```

## User-Agent

```text
REBOL View 2.7.8.3.1
```

## Downloaded Files

```text
filter.msi
10opd3r_load.msi
```

## Dropped Files

```text
arab.bin
arab.exe
```

---

# Attack Chain Reconstruction

```text
Initial Infection
        ↓
MirrorBlast Execution
        ↓
Beacon to C2
        ↓
Communication with fidufagios.com
        ↓
HTTP Requests Generated
        ↓
Additional Infrastructure Contacted
        ↓
MSI Payload Download
        ↓
Files Written to ProgramData
        ↓
Malware Execution
        ↓
IDS Alert Triggered
        ↓
SOC Investigation
```

---

# Lessons Learned

This investigation demonstrates how IDS signatures can quickly identify malware command-and-control activity before significant damage occurs.

Key indicators included:

* Suspicious User-Agent strings
* Known malicious infrastructure
* MSI payload downloads
* Beaconing activity
* Threat intelligence correlation

Combining packet analysis with threat intelligence sources such as VirusTotal provides valuable context and enables rapid validation of security alerts.

---

# Conclusion

Warzone 1 focuses on practical SOC investigation techniques involving IDS alerts, PCAP analysis, malware command-and-control detection, and threat intelligence enrichment.

By analyzing the network traffic, identifying the command-and-control infrastructure, reviewing VirusTotal intelligence, and tracing downloaded payloads, it was possible to confirm the alert as a true positive and reconstruct the malware infection chain. The investigation ultimately linked the activity to the MirrorBlast malware family and TA505-associated infrastructure.
