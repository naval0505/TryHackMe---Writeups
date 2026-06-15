# TryHackMe - New Hire, Old Artifacts Walkthrough

## Challenge Information

| Category       | Value                                                                                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Platform       | TryHackMe                                                                                                                                                                |
| Challenge Name | New Hire, Old Artifacts                                                                                                                                                  |
| Difficulty     | Easy                                                                                                                                                                     |
| Type           | Defensive Security / Splunk Investigation                                                                                                                                |
| Objective      | Investigate historical endpoint activity and determine whether the Finance department workstation was compromised during a period when endpoint protection was disabled. |

---

# Scenario

We are working as a SOC Analyst for an MSSP called TryNotHackMe.

A newly onboarded customer, Widget LLC, recently connected their endpoint telemetry to the managed Splunk environment. During onboarding, concerns were raised regarding a Finance department workstation belonging to a newly hired Financial Analyst.

The concern revolves around a period in December 2021 when endpoint protection had been disabled and no formal investigation was ever conducted.

Our task is to review historical Splunk logs and identify any indicators of compromise, malicious activity, persistence mechanisms, defensive evasion attempts, and malware execution.

---

# Initial Investigation

After accessing the Splunk environment, we begin with broad searches against the available endpoint logs.

The goal is to identify suspicious process execution, credential theft tools, malware activity, registry modifications, and endpoint tampering.

---

# Question 1

## A Web Browser Password Viewer executed on the infected machine. What is the name of the binary?

Searching Splunk for browser password recovery activity quickly reveals a suspicious executable.

```spl
index=main Browser Password Viewer
```

Result:

```text
C:\Users\FINANC~1\AppData\Local\Temp\11111.exe
```

### Answer

```text
C:\Users\FINANC~1\AppData\Local\Temp\11111.exe
```

---

# Question 2

## What company created the browser password viewer?

Reviewing the metadata associated with the executable reveals the software publisher.

The utility is identified as a NirSoft credential recovery tool.

### Answer

```text
NirSoft
```

---

# Question 3

## Another suspicious binary was executed from the same folder. What was the binary name and original filename?

Using the Temp directory as a pivot point:

```spl
index=main CurrentDirectory="C:\\Users\\Finance01\\AppData\\Local\\Temp\\"
| table Image OriginalFileName
```

Results reveal a second executable:

```text
Image:
C:\Users\Finance01\AppData\Local\Temp\IonicLarge.exe

OriginalFileName:
PalitExplorer.exe
```

Additional process metadata:

```text
Description: Explorer
Product: PalitExplorer
Company: Palit
OriginalFileName: PalitExplorer.exe
```

### Answer

```text
IonicLarge.exe,PalitExplorer.exe
```

---

# Question 4

## What malicious IP address did the binary communicate with?

Using the process discovered previously:

```spl
index=main Image="C:\\Users\\Finance01\\AppData\\Local\\Temp\\IonicLarge.exe"
```

Reviewing network connection events and destination IP fields reveals outbound communication to a suspicious address.

Defanged:

### Answer

```text
2[.]56[.]59[.]42
```

---

# Question 5

## What registry key was modified?

Registry modification events are represented by Sysmon Event ID 13.

Search:

```spl
index=main Image="C:\\Users\\Finance01\\AppData\\Local\\Temp\\IonicLarge.exe" EventCode=13
```

The process modified Windows Defender policy settings.

Registry Key:

```text
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender
```

### Answer

```text
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender
```

---

# Question 6

## Which binaries were terminated and deleted?

Investigating taskkill and deletion activity:

```spl
index=main taskkill
```

Reviewing command execution logs reveals the attacker terminating and deleting malware artifacts.

Relevant files:

```text
WvmIOrcfsuILdX6SNwIRmGOJ.exe

phcIAmLJMAIMSa9j9MpgJo1m.exe
```

### Answer

```text
WvmIOrcfsuILdX6SNwIRmGOJ.exe,phcIAmLJMAIMSa9j9MpgJo1m.exe
```

---

# Question 7

## What was the final PowerShell command executed to alter Windows Defender behavior?

The attacker performed several PowerShell commands targeting Defender settings.

Search:

```spl
index=main powershell
```

Reviewing command history shows multiple Defender configuration modifications intended to weaken endpoint protection and bypass security controls.

The final command in the sequence provides the answer required by the challenge.

---

# Question 8

## Which four IDs were configured by the attacker?

Reviewing the same PowerShell session reveals multiple values being configured.

IDs identified in order of execution:

```text
2147737007
2147737010
2147735503
2147737394
```

### Answer

```text
2147737007,2147737010,2147735503,2147737394
```

---

# Question 9

## Another malicious executable was launched from a different AppData location. What was the full path?

Searching broadly across AppData execution events:

```spl
index=main AppData
```

Results reveal another suspicious application.

Executable:

```text
C:\Users\Finance01\AppData\Roaming\EasyCalc\EasyCalc.exe
```

### Answer

```text
C:\Users\Finance01\AppData\Roaming\EasyCalc\EasyCalc.exe
```

---

# Question 10

## Which DLL files were loaded by EasyCalc.exe?

Using the discovered executable as a pivot:

```spl
index=main Image="C:\\Users\\Finance01\\AppData\\Roaming\\EasyCalc\\EasyCalc.exe"
```

Reviewing module load activity reveals several DLLs loaded from the application's directory.

DLLs identified:

```text
ffmpeg.dll
nw.dll
nw_elf.dll
```

Alphabetically sorted:

### Answer

```text
ffmpeg.dll,nw.dll,nw_elf.dll
```

---

# Attack Chain Reconstruction

Based on the investigation, the compromise likely followed the sequence below:

```text
User Execution
        ↓
Browser Password Viewer Executed
        ↓
Credential Collection
        ↓
IonicLarge.exe Execution
        ↓
Outbound C2 Communication
        ↓
Windows Defender Registry Tampering
        ↓
Defensive Evasion
        ↓
Malware Cleanup Activity
        ↓
Additional Malware Deployment
        ↓
EasyCalc.exe Execution
        ↓
DLL-Based Payload Loading
```

---

# Indicators of Compromise (IOCs)

## Executables

```text
11111.exe
IonicLarge.exe
PalitExplorer.exe
EasyCalc.exe
WvmIOrcfsuILdX6SNwIRmGOJ.exe
phcIAmLJMAIMSa9j9MpgJo1m.exe
```

## DLLs

```text
ffmpeg.dll
nw.dll
nw_elf.dll
```

## Registry Modifications

```text
HKLM\SOFTWARE\Policies\Microsoft\Windows Defender
```

## Network Indicators

```text
2[.]56[.]59[.]42
```

---

# MITRE ATT&CK Mapping

| Technique | Description                              |
| --------- | ---------------------------------------- |
| T1204     | User Execution                           |
| T1055     | Process Injection / Process Manipulation |
| T1112     | Modify Registry                          |
| T1562.001 | Disable or Modify Security Tools         |
| T1071     | Application Layer Protocol Communication |
| T1105     | Ingress Tool Transfer                    |
| T1003     | Credential Access                        |
| T1070     | Indicator Removal on Host                |

---

# Findings

The workstation showed strong evidence of compromise during the period when endpoint protection was disabled.

Evidence collected includes:

* Credential theft tooling execution.
* Execution of multiple suspicious binaries.
* Communication with an external malicious IP.
* Registry modifications targeting Windows Defender.
* Malware cleanup activity to remove evidence.
* Additional payload execution from user AppData directories.
* DLL-based malware loading mechanisms.

The investigation confirms that the endpoint was likely compromised and should be considered untrusted during the affected timeframe.

---

# Conclusion

This investigation demonstrates how historical endpoint logs can reconstruct an attack months after initial compromise. By pivoting through process execution, registry changes, network connections, and module loading events, we were able to identify multiple stages of malicious activity.

The attacker successfully executed credential theft tools, established outbound communication, modified Windows Defender settings to weaken security controls, removed artifacts, and launched additional malware payloads.

The findings provide sufficient evidence for incident response actions, endpoint rebuilding, credential resets, and threat hunting across the wider Widget LLC environment.
