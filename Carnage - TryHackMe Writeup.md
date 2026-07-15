# Carnage - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Room Name | Carnage |
| Difficulty | Easy |
| Category | Network Forensics / Malware Traffic Analysis |
| Evidence | PCAP (Packet Capture) |
| Primary Tool | Wireshark |

---

# Introduction

Today we are solving another **TryHackMe Defensive Security** challenge named **Carnage**.

Unlike penetration testing rooms where the objective is to compromise a machine, this challenge places us in the role of a **Security Analyst** investigating a malware infection through captured network traffic. Using a provided **PCAP** file, we reconstruct the entire attack chain, identify Indicators of Compromise (IOCs), analyze malware communications, and uncover the attacker's Command and Control (C2) infrastructure.

Throughout this investigation we will use **Wireshark** to inspect HTTP, HTTPS, DNS, SMTP, and TCP traffic while leveraging external threat intelligence sources such as **VirusTotal** to validate malicious infrastructure.

---

# Scenario

The provided scenario states:

> Eric Fischer from the Purchasing Department at Bartell Ltd received an email from a trusted contact containing a Microsoft Word document. After opening the document, the user clicked **Enable Content**, triggering malicious activity. Shortly afterward, the SOC detected suspicious outbound connections from the endpoint. A packet capture was collected from the network sensor and provided for investigation.

Our task is to analyze the PCAP and determine:

- Initial infection activity
- Malware downloads
- C2 infrastructure
- Post-compromise communications
- Additional malicious behavior

---

# Objectives

During this investigation we aim to:

- Identify the initial malicious download.
- Investigate HTTP traffic.
- Analyze TLS connections.
- Identify malicious domains.
- Discover Cobalt Strike infrastructure.
- Build the attack timeline.
- Collect Indicators of Compromise (IOCs).

---

# Investigation Methodology

The investigation follows the workflow below.

```
PCAP Acquisition
        │
        ▼
Wireshark Analysis
        │
        ▼
HTTP Investigation
        │
        ▼
TLS Analysis
        │
        ▼
Certificate Inspection
        │
        ▼
Threat Intelligence
        │
        ▼
Cobalt Strike Identification
        │
        ▼
Attack Timeline
```

---

# Understanding PCAP Analysis

A **Packet Capture (PCAP)** records every packet transmitted across a network.

Unlike endpoint forensics, network captures allow investigators to reconstruct attacker activity even after malware has been removed from the infected system.

Typical artifacts recovered include:

- HTTP Requests
- DNS Queries
- SMTP Traffic
- TLS Certificates
- Malware Downloads
- Command & Control Communications
- Data Exfiltration

Wireshark provides protocol-aware analysis that greatly simplifies identifying these artifacts.

---

# Question 1

## Identify the First Malicious HTTP Connection

The first objective is identifying when the victim initially contacted the malicious infrastructure.

Opening the PCAP inside Wireshark reveals HTTP traffic originating from:

```
10.9.23.102
```

The first suspicious HTTP request appears in **Frame 1735**.

```
GET /incidunt-consequatur/documents.zip HTTP/1.1
```

Destination:

```
85.187.128.24
```

Initially, Wireshark displays timestamps in seconds since capture.

To obtain human-readable timestamps:

```
View

↓

Time Display Format

↓

Date and Time of Day
```

The first malicious HTTP request occurred at:

```
2021-09-24 16:44:38 UTC
```

### Answer

```
2021-09-24 16:44:38 UTC
```

---

# Why This Timestamp Matters

Establishing the first malicious connection allows investigators to:

- Build an attack timeline.
- Correlate endpoint alerts.
- Compare firewall logs.
- Identify patient zero.
- Measure attacker dwell time.

---

# Question 2

## Identify the Downloaded ZIP File

Reviewing the same HTTP request reveals the downloaded object.

```
GET

/documents.zip
```

### Answer

```
documents.zip
```

---

# Analysis

Rather than delivering malware directly, attackers frequently distribute compressed archives.

Advantages include:

- Reduced antivirus detection.
- Smaller download size.
- Easier email delivery.
- Hidden malicious payloads.

---

# Question 3

## Identify the Hosting Domain

The HTTP request contains the Host header.

```
Host:

attirenepal.com
```

### Answer

```
attirenepal.com
```

---

# Why Host Headers Matter

Attackers frequently:

- Rotate IP addresses.
- Share infrastructure.
- Use compromised websites.

The Host header often provides a more reliable Indicator of Compromise than the IP address alone.

---

# Question 4

## Identify the File Stored Inside the Archive

Rather than downloading the ZIP archive, Wireshark allows direct inspection.

Following the HTTP stream reveals the archive contents.

The embedded filename is:

```
chart-1530076591.xls
```

### Answer

```
chart-1530076591.xls
```

---

# Why This Technique Is Useful

Being able to recover filenames directly from captured traffic:

- Preserves forensic integrity.
- Avoids executing malware.
- Speeds incident response.
- Prevents accidental infection.

---

# Question 5

## Identify the Web Server

Inspecting the HTTP response (**Frame 2173**) reveals several response headers.

```
Server:

LiteSpeed
```

### Answer

```
LiteSpeed
```

---

# Understanding HTTP Headers

Response headers frequently disclose valuable information including:

- Web server software.
- Programming language.
- Operating system.
- Frameworks.
- Security configuration.

These details assist investigators in understanding attacker infrastructure.

---

# Question 6

## Identify the Web Server Version

Reviewing the same response reveals:

```
X-Powered-By:

PHP/7.2.34
```

Although the web server is LiteSpeed, the application itself is running:

```
PHP 7.2.34
```

### Answer

```
PHP/7.2.34
```

---

# Why Version Information Matters

Knowing application versions allows investigators to:

- Identify vulnerable software.
- Attribute infrastructure.
- Detect outdated deployments.
- Correlate additional compromises.

---

# Question 7

## Identify Additional Malicious Domains

The challenge indicates additional malware downloads occurred shortly afterward.

Filtering TLS Client Hello traffic during the specified timeframe:

```
tls.handshake.type == 1
```

between:

```
2021-09-24 16:45:11

↓

2021-09-24 16:45:30
```

reveals three suspicious domains.

```
finejewels.com.au

thietbiagt.com

new.americold.com
```

### Answer

```
finejewels.com.au

thietbiagt.com

new.americold.com
```

---

# Why TLS Client Hello Is Useful

Even when HTTPS traffic is encrypted, the TLS handshake frequently reveals:

- Requested domains.
- Certificates.
- Cipher suites.
- Server Name Indication (SNI).

This information is invaluable during encrypted traffic investigations.

---

# Question 8

## Certificate Authority

The first suspicious domain is:

```
finejewels.com.au
```

Following its TCP stream exposes the presented SSL certificate.

The issuing Certificate Authority is:

```
Go Daddy Secure Certificate Authority - G2
```

### Answer

```
Go Daddy Secure Certificate Authority - G2
```

---

# Why Certificates Matter

Certificate analysis helps investigators:

- Validate infrastructure.
- Identify reused certificates.
- Correlate campaigns.
- Detect fake or self-signed certificates.

---

# Question 9

## Identify the Cobalt Strike Servers

One of the strongest indicators of post-compromise activity is **Cobalt Strike Beacon** traffic.

Reviewing:

```
Statistics

↓

Conversations

↓

TCP
```

reveals suspicious communications over ports commonly used by Cobalt Strike.

The identified IP addresses are:

```
185.106.96.158

185.125.204.174
```

Both IP addresses are confirmed through **VirusTotal Community** as known Cobalt Strike Command and Control infrastructure.

### Answer

```
185.106.96.158

185.125.204.174
```

---

# About Cobalt Strike

Originally developed as a legitimate red team platform, **Cobalt Strike** has become one of the most abused post-exploitation frameworks in cybercrime.

Typical capabilities include:

- Beaconing.
- Command execution.
- Credential theft.
- File transfers.
- Lateral movement.
- Process injection.

Its presence almost always indicates successful compromise.

---

# Question 10

## Host Header Used by the First Cobalt Strike Server

VirusTotal Community information reveals additional metadata for:

```
185.106.96.158
```

Among the extracted configuration is the Host header used during beacon communication.

```
ocsp.verisign.com
```

### Answer

```
ocsp.verisign.com
```

Attackers commonly configure Cobalt Strike profiles to impersonate legitimate web services such as certificate validation endpoints. By using trusted-looking Host headers, beacon traffic blends into normal HTTPS activity, making network-based detection significantly more difficult.

---

# Current Investigation Findings

At this stage we have successfully identified:

- The initial malicious HTTP download.
- The exact infection timestamp.
- The downloaded ZIP archive.
- The hosting domain.
- The embedded malicious Excel document.
- The web server technology.
- The application version.
- Three additional malicious download domains.
- The SSL Certificate Authority.
- Two confirmed Cobalt Strike Command and Control servers.
- The Host header used by the first Cobalt Strike beacon.

The remaining investigation focuses on post-infection communications, DNS analysis, additional Cobalt Strike infrastructure, SMTP activity, malware beaconing behavior, complete IOC collection, MITRE ATT&CK mapping, and reconstruction of the full attack timeline.

# Question 11

## Identify the Domain Name for the First Cobalt Strike Server

After identifying the first Cobalt Strike Command and Control (C2) IP address:

```
185.106.96.158
```

additional threat intelligence validation is performed using **VirusTotal**.

The Community section reveals the complete Cobalt Strike beacon configuration, including the associated domain name.

The identified domain is:

```
survmeter.live
```

### Answer

```
survmeter.live
```

---

# Why Threat Intelligence Matters

Although packet captures reveal IP addresses, public threat intelligence platforms often provide additional context such as:

- Associated domains
- Malware families
- Campaign attribution
- Known C2 infrastructure
- Historical observations

This enriches the investigation and improves IOC collection.

---

# Question 12

## Identify the Second Cobalt Strike Domain

Repeating the same process for the second Cobalt Strike server:

```
185.125.204.174
```

VirusTotal Community reports reveal another malicious domain.

```
securitybusinpuff.com
```

### Answer

```
securitybusinpuff.com
```

---

# Cobalt Strike Infrastructure

At this stage two confirmed C2 domains have been identified.

| IP Address | Domain |
|------------|---------|
| 185.106.96.158 | survmeter.live |
| 185.125.204.174 | securitybusinpuff.com |

These domains represent attacker-controlled infrastructure responsible for managing compromised hosts.

---

# Question 13

## Identify the Post-Infection Domain

Once the malware establishes persistence, it begins communicating with another external domain.

Reviewing the HTTP traffic reveals repeated communication with:

```
maldivehost.net
```

### Answer

```
maldivehost.net
```

---

# Why Post-Infection Traffic Is Important

Traffic occurring after the initial compromise often reveals:

- Command and Control
- Beaconing intervals
- Tasking
- Data exfiltration
- Secondary payload delivery

These communications provide insight into the attacker's objectives after gaining access.

---

# Question 14

## First Eleven Characters Sent to the Malicious Domain

Reviewing the first HTTP request directed toward:

```
maldivehost.net
```

reveals the following URI.

```
http://maldivehost.net/zLIisQRWZI9/OQsaDixzHTgtfjMcGypGenpldWF5eWV9f3k=
```

The first eleven characters transmitted are:

```
zLIisQRWZI9
```

### Answer

```
zLIisQRWZI9
```

This string likely represents an encoded identifier or beacon value used by the malware.

---

# Question 15

## First Packet Length Sent to the C2 Server

Inspecting the initial beacon packet reveals its total packet length.

```
281 bytes
```

### Answer

```
281
```

---

# Why Packet Sizes Matter

Many malware families generate:

- Fixed beacon sizes
- Predictable packet lengths
- Consistent timing intervals

These characteristics can be used for behavioral detection even when payload contents are encrypted.

---

# Question 16

## Identify the Server Header

Reviewing the HTTP response from the malicious server reveals:

```
Server:

Apache/2.4.49 (cPanel)

OpenSSL/1.1.1l

mod_bwlimited/1.4
```

### Answer

```
Apache/2.4.49 (cPanel) OpenSSL/1.1.1l mod_bwlimited/1.4
```

---

# Infrastructure Fingerprinting

HTTP response headers frequently expose:

- Web server software
- Control panels
- SSL libraries
- Installed modules

Such information can assist investigators in identifying shared attacker infrastructure.

---

# Question 17

## DNS Query for External IP Check

Many malware families determine the victim's external IP address before beginning Command and Control communications.

Filtering DNS traffic:

```
dns.qry.name contains "api"
```

reveals the first request occurring at:

```
2021-09-24 17:00:04 UTC
```

### Answer

```
2021-09-24 17:00:04 UTC
```

---

# Question 18

## Domain Used for IP Discovery

The DNS request resolves:

```
api.ipify.org
```

### Answer

```
api.ipify.org
```

---

# Why Malware Checks External IP Addresses

Attackers frequently determine the victim's public IP address to:

- Identify NAT environments.
- Geolocate victims.
- Track infected hosts.
- Avoid sandbox environments.
- Generate victim identifiers.

This behavior is extremely common across numerous malware families.

---

# Question 19

## First MAIL FROM Address

Toward the end of the capture, SMTP traffic becomes visible.

Inspecting the SMTP authentication sequence reveals:

```
Username:

ZmFyc2hpbkBtYWlsZmEuY29t
```

The value is Base64 encoded.

Decoding produces:

```
farshin@mailfa.com
```

### Answer

```
farshin@mailfa.com
```

---

# Understanding SMTP Analysis

SMTP traffic can reveal:

- Email accounts.
- Credentials.
- Spam campaigns.
- Malware propagation.
- Data exfiltration.

Even encoded usernames often provide valuable attribution evidence.

---

# Question 20

## Total SMTP Packets

Reviewing the SMTP conversation statistics shows:

```
1439
```

packets exchanged during the mail session.

### Answer

```
1439
```

This confirms substantial SMTP activity, suggesting either spam distribution or outbound email communication initiated by the malware.

---

# Complete Attack Timeline

Combining all recovered evidence allows reconstruction of the compromise.

```
Malicious Email Delivered
          │
          ▼
Victim Opens Word Document
          │
          ▼
Enable Content Clicked
          │
          ▼
HTTP Download
(documents.zip)
          │
          ▼
Malicious Excel File
          │
          ▼
Additional Payload Downloads
          │
          ▼
TLS Connections
          │
          ▼
Cobalt Strike Beacon
          │
          ▼
Command & Control
          │
          ▼
Post-Infection Traffic
          │
          ▼
External IP Discovery
(api.ipify.org)
          │
          ▼
SMTP Activity
```

---

# Indicators of Compromise (IOCs)

| Indicator | Value |
|-----------|-------|
| Initial Domain | attirenepal.com |
| Downloaded File | documents.zip |
| Embedded File | chart-1530076591.xls |
| Additional Domains | finejewels.com.au, thietbiagt.com, new.americold.com |
| C2 IP | 185.106.96.158 |
| C2 Domain | survmeter.live |
| C2 IP | 185.125.204.174 |
| C2 Domain | securitybusinpuff.com |
| Post-Infection Domain | maldivehost.net |
| External IP Service | api.ipify.org |
| SMTP Address | farshin@mailfa.com |

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| Phishing Attachment | T1566.001 |
| User Execution | T1204 |
| Ingress Tool Transfer | T1105 |
| Command and Control | T1071 |
| Application Layer Protocol | T1071.001 |
| Beaconing | T1071 |
| Data Encoding | T1132 |
| Exfiltration Over Web Protocol | T1041 |
| Email Collection / Abuse | T1114 |

---

# Initial Incident Assessment

The network evidence confirms a successful malware infection initiated through a phishing email containing a malicious Microsoft Word document. After the victim enabled document content, the compromised host downloaded a ZIP archive containing a malicious Excel file from **attirenepal.com**.

Following execution, the malware established multiple outbound HTTPS connections, contacted additional malicious domains, and initiated communication with two confirmed **Cobalt Strike** Command and Control servers. Subsequent traffic showed beaconing behavior, external IP discovery through **api.ipify.org**, and SMTP communications that may indicate spam activity or additional malware operations.

Overall, the captured traffic demonstrates a complete infection lifecycle progressing from initial access through post-exploitation and command-and-control communications.

---

# Detection Opportunities

Security teams should monitor for:

- Office documents downloading external payloads.
- Suspicious ZIP archive downloads.
- HTTP requests to newly observed domains.
- Cobalt Strike beacon traffic.
- Connections using known malicious Host headers.
- DNS queries to external IP discovery services.
- Unexpected SMTP sessions from workstations.
- Repeated beacon traffic with consistent packet sizes.

Behavior-based detection combined with threat intelligence feeds can identify similar compromises before attackers establish long-term persistence.

---

# Security Recommendations

To reduce the likelihood and impact of similar attacks:

- Block Office macros and untrusted active content.
- Deploy email filtering capable of detecting malicious attachments.
- Inspect outbound HTTP and HTTPS traffic for suspicious domains.
- Integrate IOC feeds into IDS/IPS and SIEM platforms.
- Monitor for known Cobalt Strike indicators.
- Restrict outbound SMTP traffic from user workstations.
- Enable DNS logging and anomaly detection.
- Train users to recognize phishing emails and suspicious document prompts.

---

# Lessons Learned

The **Carnage** challenge demonstrates the importance of network traffic analysis during incident response. Even without endpoint access, investigators can reconstruct the complete attack lifecycle using packet captures. By correlating HTTP requests, TLS handshakes, DNS queries, SMTP traffic, and external threat intelligence, it becomes possible to identify malware downloads, attacker infrastructure, and post-compromise activity.

This room also highlights the value of combining Wireshark with platforms such as VirusTotal. While Wireshark reveals raw network activity, threat intelligence provides the context needed to attribute malicious infrastructure, confirm Cobalt Strike servers, and strengthen Indicators of Compromise.

---

# Conclusion

**Carnage** is an excellent introductory Network Forensics challenge that demonstrates how packet captures can be used to investigate a complete malware infection from initial phishing delivery through post-exploitation communications. Using **Wireshark**, we reconstructed the infection timeline, identified the malicious ZIP archive and embedded payload, analyzed encrypted TLS traffic, discovered multiple attacker-controlled domains, confirmed two **Cobalt Strike** Command and Control servers, and documented post-infection beaconing and SMTP activity.

The investigation reinforces the importance of packet analysis, protocol knowledge, and threat intelligence correlation in modern incident response. It also emphasizes that even encrypted traffic can provide valuable forensic evidence through metadata such as DNS queries, TLS handshakes, HTTP headers, and communication patterns. Overall, Carnage provides an excellent practical introduction to malware traffic analysis and network-based incident investigations.
