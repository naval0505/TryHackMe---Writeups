# TryHackMe – Investigating Windows Logs with Splunk

## Scenario

Today we are investigating another TryHackMe Defensive Security challenge where we are provided with Windows event logs that have already been ingested into Splunk. The SOC team observed suspicious activity across several Windows hosts and suspects that an attacker gained access and established persistence through the creation of a backdoor account.

Our task as SOC Analysts is to analyze the logs, identify malicious activity, trace attacker actions, and determine how persistence was achieved.

---

# Initial Investigation

## Splunk Environment

```text
Main IP :: 10.48.175.211
```

The first step is to open the Splunk Search & Reporting dashboard and set the time range to **All Time** to ensure no events are missed during the investigation.

---

# Q1. How many events were collected and ingested into the main index?

To determine the total number of events stored inside the index:

```spl
index=main
```

After selecting **All Time**, Splunk displays the total number of events.

### Answer

```text
12256
```

This confirms that 12,256 Windows events were collected for analysis.

---

# Q2. What backdoor user account was created by the attacker?

One of the most common persistence techniques is creating a local user account.

Windows records user creation events using:

```text
Event ID 4720
```

Searching for account creation events:

```spl
index=main EventCode=4720
```

Reviewing the results reveals a newly created account.

### Answer

```text
A1berto
```

The attacker intentionally used a username visually similar to a legitimate user account to avoid detection.

---

# Q3. What registry key was updated for the new backdoor user?

User account creation causes Windows Security Account Manager (SAM) entries to be written to the registry.

Searching for the username:

```spl
index=main EventCode=13 A1berto
```

Event ID 13 corresponds to Registry Value Set operations.

The modified registry path is:

### Answer

```text
HKLM\SAM\SAM\Domains\Account\Users\Names\A1berto
```

This confirms that the user account was successfully added to the local system.

---

# Q4. Which legitimate user was the attacker attempting to impersonate?

The suspicious username:

```text
A1berto
```

uses the number **1** in place of the lowercase letter **l**.

This is a common attacker technique known as:

```text
Typosquatting
Visual Impersonation
Lookalike Account Creation
```

The legitimate user being impersonated was:

### Answer

```text
alberto
```

The attacker likely hoped administrators would overlook the slight spelling difference.

---

# Q5. What command was used to create the backdoor account remotely?

Searching process creation logs helps identify the exact command used.

Useful query:

```spl
index=main EventID=1 OR EventID=4688 A1berto
```

Reviewing process execution events reveals a WMIC-based remote execution command.

### Answer

```text
C:\windows\System32\Wbem\WMIC.exe" /node:WORKSTATION6 process call create "net user /add A1berto paw0rd1
```

---

## Analysis

The attacker leveraged:

```text
WMIC.exe
```

to remotely execute:

```cmd
net user /add A1berto paw0rd1
```

This demonstrates:

* Remote administration abuse
* Living-Off-The-Land (LOLBins)
* Persistence through account creation

---

# Q6. How many successful login attempts were observed from the backdoor account?

After creating persistence, attackers typically authenticate using the new account.

Searching for logon events associated with the user:

```spl
index=main A1berto
```

Reviewing authentication activity shows no successful logons.

### Answer

```text
0
```

---

## Analysis

Although the account was successfully created, there is no evidence the attacker used it during the observed timeframe.

Possible explanations:

* Persistence was established for future access.
* Detection occurred before usage.
* The attacker moved to another host.

---

# Q7. Which host executed suspicious PowerShell commands?

PowerShell activity is frequently associated with malware deployment, reconnaissance, and post-exploitation actions.

Initial host analysis:

```spl
index=main
```

Reviewing host activity highlights one system with suspicious PowerShell execution.

### Answer

```text
James.browne
```

---

# Q8. How many PowerShell logging events were recorded?

PowerShell logging was enabled, providing valuable visibility.

Searching:

```spl
index=main EventID=4104 OR EventID=4103
```

Where:

```text
4103 = Module Logging
4104 = Script Block Logging
```

Counting the events reveals:

### Answer

```text
79
```

---

## Why Event IDs 4103 and 4104 Matter

PowerShell logging is one of the most valuable Windows forensic data sources because it records:

* Executed commands
* Decoded scripts
* Downloaded payloads
* Obfuscated attacker activity

Without these logs, identifying PowerShell-based attacks becomes significantly more difficult.

---

# Q9. What malicious URL was identified?

Reviewing the PowerShell execution chain reveals a long Base64-encoded command.

Attackers commonly use:

```powershell
powershell.exe -enc <base64>
```

to hide malicious activity.

After decoding the payload, the destination URL is revealed.

### Answer

```text
hxxp[://]10[.]10[.]10[.]5/news[.]php
```

---

# PowerShell Attack Analysis

The attack chain appears to be:

```text
Encoded PowerShell Command
            ↓
Payload Decoded
            ↓
Connection to 10.10.10.5
            ↓
Request to news.php
            ↓
Potential Payload Download
            ↓
Execution on Host
```

Indicators suggest the URL may have served:

* Malware payloads
* Command-and-Control instructions
* Secondary-stage PowerShell scripts

---

# Indicators of Compromise (IOCs)

## Malicious User Account

```text
A1berto
```

## Impersonated User

```text
alberto
```

## Remote Execution Utility

```text
WMIC.exe
```

## User Creation Command

```cmd
net user /add A1berto paw0rd1
```

## Registry Persistence Entry

```text
HKLM\SAM\SAM\Domains\Account\Users\Names\A1berto
```

## Suspicious Host

```text
James.browne
```

## Malicious URL

```text
hxxp[://]10[.]10[.]10[.]5/news[.]php
```

---

# MITRE ATT&CK Mapping

## Account Creation

```text
T1136.001
Create Account: Local Account
```

The attacker created a new local user for persistence.

---

## Remote Execution

```text
T1047
Windows Management Instrumentation
```

WMIC was used to execute commands remotely.

---

## PowerShell Abuse

```text
T1059.001
PowerShell
```

Encoded PowerShell commands were executed on the compromised host.

---

## Masquerading

```text
T1036
Masquerading
```

The account:

```text
A1berto
```

was created to resemble:

```text
alberto
```

---

# Investigation Summary

During log analysis, we identified a successful persistence attempt through the creation of a malicious local account named **A1berto**, designed to impersonate the legitimate user **alberto**. The attacker remotely created the account using **WMIC.exe** and modified corresponding SAM registry entries. Although no successful logins from the account were observed, the activity clearly indicates persistence preparation.

Further investigation uncovered suspicious PowerShell activity on host **James.browne**, where 79 PowerShell logging events were recorded. Analysis of an encoded PowerShell command revealed communication with the malicious URL:

```text
hxxp[://]10[.]10[.]10[.]5/news[.]php
```

The evidence strongly suggests an attacker attempted to establish persistence, remotely manage compromised hosts, and potentially download additional payloads using PowerShell-based execution techniques.
