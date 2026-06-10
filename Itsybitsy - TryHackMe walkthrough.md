# ItsyBitsy - TryHackMe Walkthrough

## Challenge Information

| Category       | Value                                                                                  |
| -------------- | -------------------------------------------------------------------------------------- |
| Platform       | TryHackMe                                                                              |
| Challenge Name | ItsyBitsy                                                                              |
| Type           | Blue Team / SOC Investigation                                                          |
| Objective      | Investigate suspicious HTTP traffic using Kibana and identify indicators of compromise |

---

# Scenario

During routine SOC monitoring, Analyst John observed an IDS alert indicating potential Command and Control (C2) communication originating from a user named Browne in the HR department.

A suspicious file containing a pattern matching:

```text id="jngyr8"
THM{________}
```

was reportedly accessed.

Due to limited resources, only HTTP connection logs were collected and ingested into Kibana under the index:

```text id="8zbyu7"
connection_logs
```

Our task is to investigate the logs and answer the provided questions.

---

# Initial Investigation

Access the Kibana dashboard and select:

```text id="l3sxn2"
connection_logs
```

Since the scenario references activity occurring during March 2022, configure the date range accordingly.

```text id="j3ux29"
March 1, 2022
to
March 31, 2022
```

This ensures the investigation focuses on the relevant timeframe.

---

# Question 1

## How many events were returned for the month of March 2022?

After setting the correct time filter and reviewing the event count, Kibana returns:

```text id="yvwnow"
1482
```

### Answer

```text id="j2vxnh"
1482
```

### Analysis

This provides the total number of HTTP events available for investigation during the specified period.

---

# Question 2

## What is the IP associated with the suspected user in the logs?

To identify the user's machine, examine the:

```text id="vfflgk"
source_ip
```

field within Kibana.

Top values observed:

```text id="t26sqt"
192.166.65.52
192.166.65.54
```

Reviewing the suspicious activity reveals that the compromised host is:

### Answer

```text id="gskcln"
192.166.65.54
```

### Analysis

This source IP appears repeatedly throughout the malicious activity timeline and is responsible for the suspicious outbound connections.

---

# Question 3

## The user's machine used a legitimate Windows binary to download a file from the C2 server. What is the name of the binary?

Filtering activity associated with the infected host reveals the following user agent:

```text id="rl3rza"
bitsadmin
```

### Answer

```text id="a4b8sj"
bitsadmin
```

### Analysis

Bitsadmin is a legitimate Microsoft utility used for file transfers.

Attackers frequently abuse it because:

* It is trusted by Windows
* It can download payloads
* It often bypasses basic security controls

This technique maps closely to:

```text id="kjlwm9"
MITRE ATT&CK T1197
BITS Jobs
```

---

# Question 4

## The infected machine connected with a famous file-sharing site that also acted as a C2 server. What is the name of the site?

Filtering the logs associated with:

```text id="2kgkzq"
192.166.65.54
```

reveals outbound HTTP connections to:

```text id="t3zeb7"
pastebin.com
```

### Answer

```text id="f2odod"
pastebin.com
```

### Analysis

Pastebin is frequently abused by attackers for:

* Payload hosting
* Command retrieval
* Data exfiltration
* Malware configuration storage

Because it is a legitimate service, traffic to it often blends into normal network activity.

---

# Question 5

## What is the full URL of the C2 to which the infected host connected?

Reviewing the relevant HTTP request reveals:

```text id="hfp4tb"
Host: pastebin.com
URI: /yTg0Ah6a
```

Combining these values gives the complete URL.

### Answer

```text id="oqjlwm"
pastebin.com/yTg0Ah6a
```

### Analysis

The event details show:

```text id="0u0rb8"
Source IP: 192.166.65.54
User Agent: bitsadmin
Status: 200 OK
```

This strongly suggests the host successfully retrieved content from the remote server.

---

# Question 6

## A file was accessed on the file-sharing site. What is the name of the file?

Visiting the identified Pastebin location reveals the referenced file.

### Answer

```text id="lrhpnd"
secret.txt
```

### Analysis

The retrieved file appears to contain the malicious content referenced in the investigation scenario.

---

# Question 7

## What is the flag contained within the file?

Opening the contents of:

```text id="93h4pm"
secret.txt
```

reveals the challenge flag.

### Answer

```text id="9jlwm7"
THM{SECRET__CODE}
```

---

# Timeline of Activity

```text id="ih9nzs"
March 2022
       │
       ▼
IDS Alert Triggered
       │
       ▼
Suspicious Host Identified
       │
       ▼
192.166.65.54
       │
       ▼
Outbound HTTP Traffic
       │
       ▼
bitsadmin Activity
       │
       ▼
Connection to pastebin.com
       │
       ▼
Accessed /yTg0Ah6a
       │
       ▼
Retrieved secret.txt
       │
       ▼
Flag Discovered
```

---

# Indicators of Compromise (IOCs)

### Source IP

```text id="4t3psm"
192.166.65.54
```

### User Agent

```text id="wm4s9n"
bitsadmin
```

### Domain

```text id="e6i4df"
pastebin.com
```

### URL

```text id="n9vbzf"
pastebin.com/yTg0Ah6a
```

### Retrieved File

```text id="qqm4gm"
secret.txt
```

---

# Final Answers

| Question             | Answer                |
| -------------------- | --------------------- |
| Events in March 2022 | 1482                  |
| Suspected User IP    | 192.166.65.54         |
| Download Binary      | bitsadmin             |
| File Sharing Site    | pastebin.com          |
| Full C2 URL          | pastebin.com/yTg0Ah6a |
| Retrieved File       | secret.txt            |
| Flag                 | THM{SECRET__CODE}     |

---

# Key Takeaways

* Kibana provides powerful visibility into HTTP traffic and user activity.
* Time-based filtering significantly reduces investigation noise.
* Legitimate administrative utilities such as Bitsadmin are frequently abused by attackers.
* Public services like Pastebin are often leveraged as low-cost command-and-control infrastructure.
* User-agent analysis can quickly reveal attacker tooling.
* Correlating source IPs, URLs, and file access patterns is essential during network investigations.

Challenge completed successfully.
