# TryHackMe - Investigating Conti Ransomware using Splunk

## Defensive Security Challenge Walkthrough

### Scenario

Today we are investigating a ransomware incident involving **Conti Ransomware**, one of the most notorious Ransomware-as-a-Service (RaaS) operations.

Users reported they could no longer access Outlook, and the Exchange Administrator was unable to log into the Exchange Admin Center. During the initial investigation, suspicious ransom notes were discovered on the Exchange server.

Our objective is to use **Splunk** to investigate the compromise, identify attacker activity, determine persistence mechanisms, and understand how the ransomware was deployed.

---

# Environment

### Splunk Credentials

```text
Username : bellybear
Password : password!!!
```

### Splunk URL

```text
http://10.48.172.66:8000
```

---

# Initial Reconnaissance

After logging into Splunk, navigate to:

```text
Search & Reporting
```

Before hunting for indicators, it is useful to understand which log sources are available.

### Query

```spl
index="main" earliest=0 | stats count by sourcetype | sort -count
```

### Why?

This query displays all available log sources and helps determine where useful evidence is stored.

Available data sources included:

| Sourcetype                                       | Description                                |
| ------------------------------------------------ | ------------------------------------------ |
| WinEventLog:Security                             | Authentication and security-related events |
| WinEventLog:Application                          | Application logs                           |
| WinEventLog:System                               | Windows system events                      |
| WinEventLog:Microsoft-Windows-Sysmon/Operational | Sysmon telemetry                           |
| IIS                                              | Web server logs                            |
| WinEventLog:Setup                                | Windows installation events                |
| misc_text                                        | Miscellaneous text-based logs              |

Since Sysmon provides excellent visibility into process creation, file creation, network activity, and process injection, it becomes the primary source for our investigation.

---

# Question 1

## Can you identify the location of the ransomware?

The first step was identifying suspicious file creation activity.

### Query

```spl
index=main sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=11
```

### Why?

Sysmon Event ID 11 corresponds to:

```text
File Create
```

This allows us to identify files written to disk by suspicious processes.

Reviewing the events revealed the ransomware executable being created from:

```text
C:\Users\Administrator\Documents\cmd.exe
```

This immediately stands out because ransomware normally should not originate from a user's Documents directory.

### Answer

```text
C:\Users\Administrator\Documents\cmd.exe
```

---

# Question 2

## What is the Sysmon Event ID for the related file creation event?

The same query above revealed:

```text
EventCode=11
```

Sysmon Event ID 11 corresponds to file creation operations.

### Answer

```text
11
```

---

# Question 3

## Can you find the MD5 hash of the ransomware?

### Query

```spl
index=main Image="c:\\Users\\Administrator\\Documents\\cmd.exe"
```

Reviewing the Sysmon details shows the file hash information.

### Finding

```text
290C7DFB01E50CEA9E19DA81A781AF2C
```

### Answer

```text
290C7DFB01E50CEA9E19DA81A781AF2C
```

---

# Question 4

## What file was saved to multiple folder locations?

To identify files repeatedly written across the system:

### Query

```spl
index=main EventCode=11
| timechart count by TargetFilename limit=10
```

### Why?

This visualizes frequently created files and highlights files repeatedly written throughout the environment.

One file appeared significantly more than others:

```text
readme.txt
```

This is consistent with ransomware behavior because ransom notes are typically dropped into multiple directories.

### Answer

```text
readme.txt
```

---

# Question 5

## What command did the attacker use to create a new user?

A common persistence technique is creating additional user accounts.

### Query

```spl
index=main "net user"
```

Reviewing the CommandLine field revealed:

```text
net user /add securityninja hardToHack123$
```

### Why This Matters

Attackers often create new local accounts to maintain access after the initial compromise.

### Answer

```text
net user /add securityninja hardToHack123$
```

---

# Question 6

## What process was migrated into for persistence?

Sysmon Event ID 8 records process injection activity.

### Query

```spl
index=main EventCode=8
```

Reviewing the injected process details revealed:

### Original Process

```text
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

### Migrated Process

```text
C:\Windows\System32\wbem\unsecapp.exe
```

### Why This Matters

Attackers frequently migrate from suspicious processes into legitimate Windows processes to blend in with normal activity.

### Answer

```text
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe,
C:\Windows\System32\wbem\unsecapp.exe
```

---

# Question 7

## What process was used to retrieve system hashes?

Continuing the process migration investigation revealed access to:

```text
C:\Windows\System32\lsass.exe
```

### Why This Matters

LSASS stores:

* Authentication credentials
* NTLM hashes
* Kerberos tickets
* Security tokens

Attackers commonly target LSASS to extract credentials and move laterally across the network.

### Answer

```text
C:\Windows\System32\lsass.exe
```

---

# Question 8

## What web shell was deployed?

Since Exchange servers are common targets, searching for ASPX files is a logical next step.

### Query

```spl
index=main ".aspx"
```

Reviewing the IIS logs and URI fields revealed:

```text
i3gfPctK1c2x.aspx
```

### Why This Matters

ASPX web shells provide remote command execution through IIS and are commonly deployed after exploiting Microsoft Exchange vulnerabilities.

### Answer

```text
i3gfPctK1c2x.aspx
```

---

# Question 9

## What command executed the web shell?

Reviewing events associated with the web shell revealed:

```text
attrib.exe -r \\win-aoqkg2as2q7.bellybear.local\C$\Program Files\Microsoft\Exchange Server\V15\FrontEnd\HttpProxy\owa\auth\i3gfPctK1c2x.aspx
```

### Why This Matters

The attacker modified file attributes on the deployed web shell to facilitate access and persistence.

### Answer

```text
attrib.exe -r \\win-aoqkg2as2q7.bellybear.local\C$\Program Files\Microsoft\Exchange Server\V15\FrontEnd\HttpProxy\owa\auth\i3gfPctK1c2x.aspx
```

---

# Question 10

## Which CVEs were used during the attack?

External threat intelligence research regarding Conti ransomware campaigns revealed the following vulnerabilities were commonly exploited:

### CVEs

```text
CVE-2020-0796
CVE-2018-13374
CVE-2018-13379
```

### Why These Matter

#### CVE-2020-0796

SMBGhost Remote Code Execution vulnerability.

#### CVE-2018-13374

Fortinet SSL VPN Path Traversal vulnerability.

#### CVE-2018-13379

Fortinet SSL VPN credential disclosure vulnerability.

These vulnerabilities have historically been used by ransomware operators to gain initial access before deploying ransomware payloads.

### Answer

```text
CVE-2020-0796
CVE-2018-13374
CVE-2018-13379
```

---

# Attack Chain Reconstruction

```text
Initial Access
      │
      ▼
Exchange Server Compromise
      │
      ▼
Web Shell Deployment
(i3gfPctK1c2x.aspx)
      │
      ▼
PowerShell Execution
      │
      ▼
Process Migration
(unsecapp.exe)
      │
      ▼
Credential Access
(lsass.exe)
      │
      ▼
Persistence
(New User Creation)
      │
      ▼
Ransom Note Deployment
(readme.txt)
      │
      ▼
Conti Ransomware Execution
```

---

# MITRE ATT&CK Mapping

| Technique | Description                       |
| --------- | --------------------------------- |
| T1190     | Exploit Public-Facing Application |
| T1505.003 | Web Shell                         |
| T1059.001 | PowerShell                        |
| T1055     | Process Injection                 |
| T1003.001 | LSASS Memory                      |
| T1136     | Create Account                    |
| T1486     | Data Encrypted for Impact         |

---

# Conclusion

The investigation revealed a complete Conti ransomware intrusion chain. The attacker gained access to the Exchange server, deployed an ASPX web shell, executed PowerShell commands, migrated into legitimate Windows processes, accessed LSASS for credential harvesting, created a new user account for persistence, and ultimately deployed ransomware along with ransom notes across the system.

The presence of web shell activity, process migration, LSASS interaction, and account creation demonstrates a hands-on-keyboard intrusion consistent with Conti ransomware operations observed in real-world incidents.
