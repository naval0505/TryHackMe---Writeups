# Friday Overtime - TryHackMe Walkthrough

## Challenge Information

| Category       | Value                                                                                                                                               |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Platform       | TryHackMe                                                                                                                                           |
| Challenge Name | Friday Overtime                                                                                                                                     |
| Type           | Defensive Security / Threat Intelligence                                                                                                            |
| Objective      | Investigate malware samples, perform threat intelligence analysis, identify malware family, associated infrastructure, and indicators of compromise |

---

# Scenario

It is a Friday evening at PandaProbe Intelligence when SwiftSpend Finance submits an urgent threat intelligence request.

Several suspicious files were discovered during an internal security sweep and have been forwarded for investigation.

As the on-duty Cyber Threat Intelligence Analyst, our objective is to:

* Download and inspect the malware samples.
* Perform static analysis.
* Correlate indicators with threat intelligence sources.
* Identify malware families and ATT&CK techniques.
* Discover infrastructure used by the threat actor.
* Produce intelligence findings for remediation.

---

# Accessing the Investigation Portal

Credentials provided:

```text
Username: ericatracy
Password: Intel321!
```

After logging into the provided portal through Chromium, we begin reviewing the ticket.

---

# Question 1

## Who shared the malware samples?

Reviewing the submitted ticket reveals the sender information.

Message excerpt:

```text
My name is Oliver Bennett from the Cybersecurity Division at SwiftSpend Finance.
```

### Answer

```text
Oliver Bennett
```

---

# Downloading the Malware Samples

The malware archive was attached to the ticket.

Archive:

```text
samples.zip
```

Password:

```text
Panda321!
```

Extracting:

```bash
unzip samples.zip
```

Files extracted:

```text
cbmrpa.dll
maillfpassword.dll
pRsm.dll
qmsdp.dll
wcdbcrk.dll
```

---

# Question 2

## What is the SHA1 hash of pRsm.dll?

Using:

```bash
sha1sum *
```

Results:

```text
22532a8c8594cd8a3294e68ceb56accf37a613b3  cbmrpa.dll
8a98a023164b50dec5126eda270d394e06a144ff  maillfpassword.dll
9d1ecbbe8637fed0d89fca1af35ea821277ad2e8  pRsm.dll
0781a2b6eb656d110a3a8f60e8bce9d407e4c4ff  qmsdp.dll
10fb52e4a3d5d6bda0d22bb7c962bde95b8da3dd  wcdbcrk.dll
```

### Answer

```text
9d1ecbbe8637fed0d89fca1af35ea821277ad2e8
```

---

# Question 3

## Which malware framework utilizes these DLLs as add-on modules?

Researching the DLL names across public threat intelligence sources reveals they are plugins used by the:

```text
MgBot
```

malware framework.

### About MgBot

MgBot is a modular malware framework commonly associated with the Chinese APT group:

```text
Evasive Panda
```

The malware uses downloadable DLL modules to extend functionality such as:

* Credential theft
* Audio capture
* Surveillance
* File collection
* Command execution

### Answer

```text
MgBot
```

---

# Question 4

## Which MITRE ATT&CK Technique is linked to pRsm.dll?

Threat intelligence analysis indicates:

```text
pRsm.dll
```

is responsible for recording audio from the victim machine.

According to MITRE ATT&CK this behavior maps to:

```text
T1123
```

Technique:

```text
Audio Capture
```

### Answer

```text
T1123
```

---

# Question 5

## What is the CyberChef defanged URL of the malicious download location first seen on 2020-11-02?

Threat intelligence reporting linked MgBot activity to the URL:

```text
http://update.browser.qq.com/qmbs/QQ/QQUrlMgr_QQ88_4296.exe
```

Defanging the URL in CyberChef produces:

### Answer

```text
hxxp[://]update[.]browser[.]qq[.]com/qmbs/QQ/QQUrlMgr_QQ88_4296.exe
```

---

# Question 6

## What is the CyberChef defanged IP address of the C2 server first detected on 2020-09-14?

Identified Command and Control IP:

```text
122.10.90.12
```

Defanged:

### Answer

```text
122[.]10[.]90[.]12
```

### Why Defang?

Defanging prevents:

* Accidental clicking
* Automated hyperlink creation
* Accidental communication with malicious infrastructure

Common replacements:

```text
.  -> [.]
http -> hxxp
```

---

# Question 7

## What is the MD5 hash of the SpyAgent spyware hosted on the same IP targeting Android devices?

Using the identified C2 IP:

```text
122.10.90.12
```

and pivoting through VirusTotal threat intelligence results reveals multiple malware samples.

One Android sample associated with the SpyAgent family shows:

```text
951F41930489A8BFE963FCED5D8DFD79
```

Converting to standard lowercase MD5 format:

### Answer

```text
951f41930489a8bfe963fced5d8dfd79
```

---

# Malware Intelligence Summary

## Malware Framework

```text
MgBot
```

## Threat Actor

```text
Evasive Panda
```

## Malware Type

```text
Modular Backdoor
```

## Capabilities

```text
Credential Theft
Audio Capture
Command Execution
File Collection
Surveillance
Persistence
```

---

# Indicators of Compromise (IOCs)

## DLL Modules

```text
cbmrpa.dll
maillfpassword.dll
pRsm.dll
qmsdp.dll
wcdbcrk.dll
```

---

## SHA1 Hash

```text
9d1ecbbe8637fed0d89fca1af35ea821277ad2e8
```

---

## Malicious URL

```text
hxxp[://]update[.]browser[.]qq[.]com/qmbs/QQ/QQUrlMgr_QQ88_4296.exe
```

---

## Command & Control Server

```text
122[.]10[.]90[.]12
```

---

## Android SpyAgent MD5

```text
951f41930489a8bfe963fced5d8dfd79
```

---

# MITRE ATT&CK Mapping

| Technique ID | Technique     |
| ------------ | ------------- |
| T1123        | Audio Capture |

---

# Final Answers

| Question                | Answer                                                              |
| ----------------------- | ------------------------------------------------------------------- |
| Who shared the samples? | Oliver Bennett                                                      |
| SHA1 of pRsm.dll        | 9d1ecbbe8637fed0d89fca1af35ea821277ad2e8                            |
| Malware Framework       | MgBot                                                               |
| MITRE ATT&CK Technique  | T1123                                                               |
| Defanged URL            | hxxp[://]update[.]browser[.]qq[.]com/qmbs/QQ/QQUrlMgr_QQ88_4296.exe |
| Defanged C2 IP          | 122[.]10[.]90[.]12                                                  |
| SpyAgent MD5            | 951f41930489a8bfe963fced5d8dfd79                                    |

---

# Key Takeaways

* Modular malware frameworks often use DLL plugins to extend functionality.
* Threat intelligence correlation is essential for identifying malware families and attribution.
* MgBot is a highly modular malware platform associated with Evasive Panda operations.
* ATT&CK mapping helps defenders understand adversary behaviors and detection opportunities.
* Defanging indicators is a standard CTI practice when documenting malicious infrastructure.
* Pivoting from infrastructure indicators often uncovers additional malware families and campaigns.

Challenge completed successfully.
