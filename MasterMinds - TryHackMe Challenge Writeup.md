# Masterminds - TryHackMe Writeup

## Overview

Today we are going to solve another **Defensive Security** challenge from **TryHackMe** named **Masterminds**.

Unlike traditional penetration testing rooms, this challenge focuses entirely on **Network Forensics**, **Incident Response**, and **Threat Intelligence**. We are provided with multiple packet capture (PCAP) files collected from compromised systems inside the Finance department of an organization. Our objective is to investigate the captured network traffic, identify indicators of compromise (IOCs), determine how each machine became infected, and attribute the malware families involved using open-source intelligence.

---

# Scenario

The scenario provided states:

> Three machines in the Finance department at Pfeffer PLC were compromised. The Incident Response team suspects the compromise originated from a phishing campaign and an infected USB drive. Network traffic from the affected endpoints has been preserved. Our task is to investigate the traffic using tools such as **Wireshark** and **Brim**, identify malicious communications, recover Indicators of Compromise (IOCs), and determine the malware responsible for each infection.

---

# Investigation Methodology

For this investigation the following methodology was used:

1. Identify each victim using DHCP traffic.
2. Analyze HTTP requests and responses.
3. Review DNS requests for suspicious domains.
4. Identify downloaded payloads.
5. Correlate malicious infrastructure using VirusTotal, URLhaus and public Threat Intelligence.
6. Validate IDS alerts using Brim and Suricata.
7. Build a timeline for each infection.

The investigation consists of **three independent infections**, each analyzed separately.

---

# Infection 1 Analysis

## Identifying the Victim Machine

We begin with the first packet capture.

The quickest way to identify a workstation inside a DHCP environment is by inspecting DHCP packets because they usually contain the hostname assigned by Windows.

Wireshark Filter:

```text
dhcp
```

Only four DHCP packets are present, making the investigation straightforward.

Among them we identify the Windows hostname:

```text
WIN-3145VKLNUNS
```

Filtering specifically for that hostname:

```text
dhcp.option.hostname == "WIN-3145VKLNUNS"
```

reveals the assigned IP address.

**Victim IP**

```text
192.168.75.249
```

---

# Investigating Suspicious HTTP Requests

The next objective is identifying outbound HTTP communications made by the compromised workstation.

Applying the HTTP protocol filter:

```text
http
```

reveals several suspicious web requests.

During the investigation two domains immediately stand out because both return **HTTP 404 Not Found** responses.

The first request:

```text
http://cambiasuhistoria.growlab.es/wp-content/hGhY2/
```

Server Response:

```text
HTTP/1.1 404 Not Found
```

The second request:

```text
http://www.letscompareonline.com/de.letscompareonline.com/wYd/
```

also returns:

```text
HTTP/1.1 404 Not Found
```

Although both servers respond with HTTP 404, these requests are still highly suspicious because malware frequently contacts unavailable or removed infrastructure after campaigns have ended.

The requested suspicious domains are therefore:

```text
cambiasuhistoria.growlab.es

www.letscompareonline.com
```

---

# Successful HTTP Communication

Not every malicious request fails.

Continuing through the HTTP traffic reveals another outbound connection.

Request:

```text
GET /wp-content/lMMC/?subid1=20210830-0642-0217-8b33-6ab4bf16c29c
```

Host:

```text
ww25.gocphongthe.com
```

Destination IP:

```text
199.59.242.153
```

Unlike the previous requests, this communication completed successfully and the response body length matched the expected value from the challenge.

Recovered IOC:

**Domain**

```text
ww25.gocphongthe.com
```

**Destination IP**

```text
199.59.242.153
```

---

# DNS Investigation

Malware generally performs DNS lookups before establishing HTTP communications.

To investigate DNS activity we switch to DNS filtering.

```text
dns
```

During analysis one domain repeatedly appears.

```text
cab.myfkn.com
```

Including both lowercase and capitalized requests, the capture contains:

```text
7 DNS requests
```

Repeated DNS lookups often indicate malware attempting to maintain communication with command-and-control infrastructure.

---

# Additional HTTP Activity

Another interesting HTTP request is observed during the investigation.

Request URI:

```text
http://bhaktivrind.com/cgi-bin/JBbb8/
```

To isolate the traffic:

```text
http.response_for.uri == "http://bhaktivrind.com/cgi-bin/JBbb8/"
```

This request successfully identifies another malicious endpoint contacted by the victim.

Recovered IOC:

```text
http://bhaktivrind.com/cgi-bin/JBbb8/
```

---

# Malware Download

Further inspection eventually reveals the initial malware download.

HTTP Request:

```text
GET /catzx.exe HTTP/1.1
```

Host:

```text
hdmilg.xyz
```

Destination IP:

```text
185.239.243.112
```

Complete URI:

```text
http://hdmilg.xyz/catzx.exe
```

The executable being downloaded directly over HTTP is a strong indicator of malware delivery.

Recovered Indicators:

**Malicious Domain**

```text
hdmilg.xyz
```

**Destination IP**

```text
185.239.243.112
```

**Downloaded File**

```text
catzx.exe
```

---

# Threat Intelligence Correlation

Rather than interacting with the malicious infrastructure, the recovered IOC is investigated using **VirusTotal** and public threat intelligence feeds.

Searching for the recovered indicators reveals that the domain appears in a published IOC collection.

Reference:

```text
Weekend Emotet IoCs and Notes
2021/01/22-24
```

The indicators were published by security researcher:

```text
jroosen
```

This allows us to confidently attribute the downloaded payload to the **Emotet** malware family.

Emotet is one of the most well-known modular banking trojans and malware loaders. It primarily spreads through phishing emails and malicious Office documents, downloads additional payloads, steals credentials, and often deploys other malware families such as TrickBot, QakBot, Ryuk ransomware, and Cobalt Strike. Even after infrastructure is taken down, historical PCAP analysis frequently reveals its command-and-control patterns, making threat intelligence correlation an essential part of incident response.

---

# Infection 1 Summary

## Victim

```text
192.168.75.249
```

## Suspicious Domains

```text
cambiasuhistoria.growlab.es

www.letscompareonline.com

ww25.gocphongthe.com

bhaktivrind.com

hdmilg.xyz
```

## Malicious Server

```text
185.239.243.112
```

## Downloaded Payload

```text
catzx.exe
```

## DNS Requests

```text
cab.myfkn.com

7 Requests
```

## Malware Family

```text
Emotet
```

At this point, the first infection has been completely reconstructed. We successfully identified the compromised host, recovered multiple indicators of compromise, observed outbound communications to malicious infrastructure, extracted the downloaded payload, and attributed the infection to the **Emotet** malware family using public threat intelligence. The next stage of the investigation focuses on **Infection 2**, where a different malware family and attack pattern are analyzed using both **Wireshark** and **Brim**.


# Infection 2 Analysis

After completing the investigation of the first compromised workstation, we now move to the second packet capture and repeat the same methodology to identify the victim, analyze its network activity, and determine the malware responsible for the compromise.

---

# Identifying the Victim Machine

As with the first investigation, we begin by inspecting DHCP traffic.

Wireshark Filter:

```text
dhcp
```

Filtering by the Windows hostname:

```text
dhcp.option.hostname == "WIN-3145VKLNUNS"
```

reveals the assigned IP address.

**Victim IP**

```text
192.168.75.146
```

---

# HTTP Investigation

To investigate outbound communications originating from the compromised host, we filter HTTP traffic associated with the victim.

```text
ip.addr == 192.168.75.146 && http
```

Several suspicious HTTP requests are immediately visible.

---

# POST Requests

Unlike the previous infection, this machine repeatedly sends **HTTP POST** requests to the same external server.

Destination IP:

```text
5.181.156.252
```

The capture shows:

```text
3 POST Requests
```

Repeated POST requests usually indicate malware beaconing or exfiltrating information to its Command and Control (C2) server.

---

# Malware Download

Continuing through the HTTP traffic reveals another executable download.

Request:

```text
GET /jollion/apines.exe HTTP/1.1
```

Host:

```text
hypercustom.top
```

Full URI:

```text
http://hypercustom.top/jollion/apines.exe
```

Destination IP:

```text
45.95.203.28
```

Recovered IOCs:

**Domain**

```text
hypercustom.top
```

**Downloaded File**

```text
jollion/apines.exe
```

**Destination IP**

```text
45.95.203.28
```

The presence of an executable downloaded directly over HTTP strongly suggests malware delivery.

---

# Suricata Alert Analysis

This investigation also includes IDS logs generated by **Suricata**, making **Brim** an ideal tool for validation.

Opening the PCAP in Brim and reviewing the generated alerts reveals two IDS events.

Alert Type:

```text
A Network Trojan was detected
```

The alerts identify the communication between:

Source:

```text
192.168.75.146
```

Destination:

```text
45.95.203.28
```

The IDS alerts confirm the malicious communication already observed manually in Wireshark.

---

# Threat Intelligence Correlation

The downloaded domain:

```text
hypercustom.top
```

is investigated using **URLhaus**.

URLhaus identifies the payload as belonging to:

```text
RedLine Stealer
```

RedLine Stealer is an information-stealing Trojan designed to harvest browser credentials, cookies, cryptocurrency wallets, autofill data, FTP credentials, VPN configurations, and other sensitive information from infected systems. It communicates with remote command-and-control infrastructure using HTTP POST requests to upload stolen information, which explains the repeated POST traffic observed in the packet capture.

---

# Infection 2 Summary

## Victim

```text
192.168.75.146
```

## Malicious Domain

```text
hypercustom.top
```

## Downloaded Payload

```text
jollion/apines.exe
```

## Destination IP

```text
45.95.203.28
```

## POST Beacon IP

```text
5.181.156.252
```

## IDS Detection

```text
Suricata
"A Network Trojan was detected"
```

## Malware Family

```text
RedLine Stealer
```

---

# Infection 3 Analysis

The final packet capture represents a completely different compromise.

Again, the investigation begins by identifying the victim through DHCP traffic.

---

# Identifying the Victim

Wireshark Filter:

```text
dhcp.option.hostname == "DESKTOP-PC79AQK"
```

Assigned IP:

```text
192.168.75.232
```

---

# Command-and-Control Discovery

Filtering HTTP requests generated by the victim:

```text
ip.addr == 192.168.75.232 && http
```

reveals three suspicious domains contacted sequentially.

```text
efhoahegue.ru

afhoahegue.ru

xfhoahegue.ru
```

These domains are contacted in chronological order and all serve malware binaries.

---

# Resolving Infrastructure

Examining the destination IP addresses reveals:

```text
63.251.106.25

199.21.76.77

162.217.98.146
```

These represent the infrastructure hosting the malicious payloads.

---

# DNS Investigation

Filtering DNS traffic associated with the first malicious domain shows:

```text
2 DNS Requests
```

Although relatively few queries were generated for the individual domain, the overall DNS activity throughout the packet capture is much larger.

---

# Malware Downloads

Reviewing the HTTP requests shows that multiple payloads were downloaded.

Instead of a single executable, the victim downloads:

```text
s1.exe

s2.exe

s3.exe

s4.exe

s5.exe
```

A total of:

```text
5 Executables
```

This indicates a staged infection where multiple binaries are delivered during the compromise.

---

# User-Agent Analysis

The malware identifies itself using the following HTTP User-Agent:

```text
Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0
```

Threat actors commonly spoof legitimate browser User-Agents to blend malicious traffic with normal web browsing activity.

---

# DNS Statistics Using Brim

Instead of manually counting DNS requests, Brim provides an efficient query.

```text
_path=="dns"
| count() by query
| sort -r count
| sum(count)
```

Result:

```text
986 DNS Requests
```

This demonstrates how SIEM-style tools can rapidly summarize large packet captures.

---

# Threat Intelligence Correlation

Rather than interacting with the malicious domains directly, the first recovered domain is searched using public threat intelligence.

Searching:

```text
"efhoahegue"
```

returns multiple intelligence reports linking the infrastructure to:

```text
Phorpiex
```

Additional confirmation is available through the Malware Trails repository.

Phorpiex (also known as Trik) is a modular worm that spreads through removable media, spam campaigns, and compromised systems. It is capable of downloading additional malware, participating in botnet operations, distributing ransomware, and propagating itself across infected environments. Its infrastructure often rotates through multiple domains and IP addresses, matching the behavior observed within the packet capture.

---

# Infection 3 Summary

## Victim

```text
192.168.75.232
```

## C2 Domains

```text
efhoahegue.ru

afhoahegue.ru

xfhoahegue.ru
```

## Infrastructure

```text
63.251.106.25

199.21.76.77

162.217.98.146
```

## Downloaded Files

```text
s1.exe

s2.exe

s3.exe

s4.exe

s5.exe
```

## DNS Requests

```text
986
```

## Malware Family

```text
Phorpiex
```

---

# Overall Attack Timeline

```text
Victim Identification
        │
        ▼
DHCP Analysis
        │
        ▼
HTTP Investigation
        │
        ▼
DNS Investigation
        │
        ▼
Executable Downloads
        │
        ▼
Command and Control Communications
        │
        ▼
Threat Intelligence Correlation
        │
        ▼
Malware Attribution
```

---

# Indicators of Compromise (IOCs)

### Domains

```text
cambiasuhistoria.growlab.es
www.letscompareonline.com
ww25.gocphongthe.com
bhaktivrind.com
hdmilg.xyz
hypercustom.top
efhoahegue.ru
afhoahegue.ru
xfhoahegue.ru
cab.myfkn.com
```

### IP Addresses

```text
192.168.75.249
192.168.75.146
192.168.75.232
199.59.242.153
185.239.243.112
45.95.203.28
5.181.156.252
63.251.106.25
199.21.76.77
162.217.98.146
```

### Malware Families

```text
Emotet
RedLine Stealer
Phorpiex
```

---

# MITRE ATT&CK Mapping

| Technique                  | ATT&CK ID |
| -------------------------- | --------- |
| Phishing                   | T1566     |
| User Execution             | T1204     |
| Ingress Tool Transfer      | T1105     |
| Command and Control        | T1071     |
| Application Layer Protocol | T1071.001 |
| Malware Download           | T1587     |
| Data Exfiltration          | T1041     |
| Information Stealing       | T1005     |

---

# Tools Used

* Wireshark
* Brim
* Suricata
* VirusTotal
* URLhaus
* Pastebin IOC Feeds
* Malware Bazaar
* Public Threat Intelligence Sources

---

# Lessons Learned

This investigation demonstrates how packet captures alone can reveal the complete lifecycle of a malware infection. By combining protocol analysis, DNS inspection, HTTP request reconstruction, IDS alerts, and open-source threat intelligence, it is possible to identify compromised hosts, recover malware delivery URLs, reconstruct command-and-control communications, and accurately attribute malware families without executing a single malicious sample.

The investigation also highlights the value of combining multiple tools. Wireshark provides detailed packet-level visibility, Brim enables fast querying and aggregation of large datasets, Suricata generates actionable intrusion alerts, and external intelligence platforms such as VirusTotal and URLhaus help attribute infrastructure to known malware campaigns.

---

# Conclusion

The **Masterminds** challenge provides a realistic digital forensics and incident response scenario involving three independent malware infections. Through systematic packet analysis, each compromised host was identified, malicious infrastructure was reconstructed, downloaded payloads were recovered, and the activity was successfully attributed to three well-known malware families: **Emotet**, **RedLine Stealer**, and **Phorpiex**. This exercise demonstrates how network forensics, IDS telemetry, and threat intelligence complement one another during incident response, enabling analysts to rapidly identify indicators of compromise, understand attacker behavior, and produce actionable findings without interacting directly with malicious infrastructure.
