# REvil Corp - TryHackMe Writeup

## Basic Information

| Category | Details |
|----------|----------|
| Platform | TryHackMe |
| Room Name | REvil Corp |
| Difficulty | Easy |
| Category | Defensive Security / Incident Response |
| Tool Used | Mandiant Redline |
| Operating System | Windows 7 |

---

# Introduction

Today we have another **TryHackMe Defensive Security** challenge listed as **Easy**, named **REvil Corp**.

Unlike traditional penetration testing rooms, this challenge focuses entirely on **Incident Response (IR)** and **Digital Forensics**. Instead of attacking a live machine, we are provided with forensic evidence collected from a compromised Windows workstation. Our objective is to analyze the evidence using **Mandiant Redline**, reconstruct the ransomware infection, identify the malicious activity, and answer a series of investigative questions.

The compromised workstation belongs to an employee whose files have suddenly been encrypted and renamed with an unfamiliar extension. The Incident Response team has already collected forensic artifacts, and it is now our responsibility to determine exactly what happened, identify the malware involved, and gather useful Indicators of Compromise (IOCs).

Throughout this investigation, we will leverage **Redline**, one of the most popular memory and host investigation tools used by forensic analysts, to examine system information, downloaded files, browser history, filesystem artifacts, and ransomware indicators.

---

# Scenario

The provided scenario states:

> One of the employees at Lockman Group gave the IT department a call after noticing that all of his files had been renamed with a strange extension that he had never seen before. After briefly examining the workstation, the IT staff immediately suspected ransomware and escalated the incident to the Incident Response team for further investigation.

Our objective is to investigate the forensic image using **Mandiant Redline** and determine how the compromise occurred while answering all investigation questions.

---

# Objectives

During this investigation we aim to:

- Identify the compromised employee.
- Determine the operating system.
- Identify the malicious executable.
- Recover the download source.
- Collect malware hashes.
- Identify encrypted files.
- Determine the ransomware extension.
- Count encrypted files.
- Continue building indicators for later threat intelligence.

---

# Investigation Methodology

The investigation follows the workflow below.

```
Forensic Evidence
        │
        ▼
Redline Analysis
        │
        ▼
System Information
        │
        ▼
User Identification
        │
        ▼
Download History
        │
        ▼
Malware Identification
        │
        ▼
File System Analysis
        │
        ▼
Encrypted Files
        │
        ▼
Timeline Analysis
        │
        ▼
Incident Reconstruction
```

---

# About Mandiant Redline

**Mandiant Redline** is a forensic and incident response tool designed to analyze host artifacts collected from Windows systems.

It allows investigators to examine:

- Running processes
- Loaded modules
- Browser history
- Registry information
- Download history
- File system artifacts
- Persistence mechanisms
- Memory artifacts
- Indicators of Compromise (IOCs)

Rather than manually navigating through Windows artifacts, Redline organizes evidence into searchable categories, making investigations significantly faster.

For this challenge, an analysis package has already been prepared.

Opening the **Analysis File** located on the Desktop loads all collected forensic evidence into Redline.

---

# Starting the Investigation

Once the analysis package is loaded, Redline presents several investigation categories including:

- System Information
- File System
- Processes
- Users
- Browser History
- Host Artifacts
- Timeline

These sections will be used throughout the investigation.

---

# Question 1

## Identify the Compromised Employee

The first objective is identifying which employee owns the compromised workstation.

Navigating to the **System Information** section immediately reveals the primary logged-in user.

The account owner is identified as:

```
John Coleman
```

### Answer

```
John Coleman
```

---

# Why This Matters

Identifying the affected user is one of the first steps during every incident response investigation.

Knowing the compromised account allows responders to:

- Scope the incident.
- Search authentication logs.
- Review email activity.
- Identify lateral movement.
- Notify the affected employee.

---

# Question 2

## Determine the Operating System

Remaining inside the system information section reveals the operating system details.

The compromised machine is running:

```
Windows 7 Home Premium

Service Pack 1

Build 7601
```

### Answer

```
Windows 7 Home Premium 7601 Service Pack 1
```

---

# Why the Operating System Matters

Knowing the operating system helps investigators:

- Identify applicable vulnerabilities.
- Understand available security features.
- Determine supported logging mechanisms.
- Compare malware behavior across Windows versions.

Older operating systems such as Windows 7 are particularly attractive ransomware targets because they often lack modern security protections.

---

# Question 3

## Identify the Malicious Executable

The scenario suggests that the user manually executed malicious software.

Instead of searching every directory individually, Redline makes this much easier.

Navigating to:

```
Browser History

↓

Download History
```

reveals recently downloaded files.

Among legitimate downloads, one executable immediately stands out.

```
WinRAR2021.exe
```

The filename attempts to impersonate the legitimate WinRAR installer, making it highly convincing for unsuspecting users.

### Answer

```
WinRAR2021.exe
```

---

# Why Fake Installers Are Common

Cybercriminals frequently disguise malware as popular software installers.

Examples include:

- WinRAR
- Google Chrome
- Microsoft Office
- Adobe Reader
- VLC Media Player

Users trust familiar names, making fake installers an effective social engineering technique.

---

# Question 4

## Determine the Download URL

The next objective is identifying where the malware originated.

Remaining inside the browser download history reveals the complete download record.

The URL recorded by Internet Explorer is:

```
http://192.168.75.129:4748/Documents/WinRAR2021.exe
```

This includes both the server location and the downloaded executable.

### Answer

```
http://192.168.75.129:4748/Documents/WinRAR2021.exe
```

---

# Why Download History Is Valuable

Browser download history often contains:

- Original filenames.
- Source URLs.
- Download timestamps.
- Browser version.
- Download location.
- User account.

These artifacts are extremely useful for identifying malware distribution infrastructure.

---

# Question 5

## Determine the Malware MD5 Hash

To collect Indicators of Compromise, the malicious executable must be located within the filesystem.

Using Redline's **File System** view, navigate to:

```
C:\Users\John Coleman\Downloads
```

Opening the file properties for:

```
WinRAR2021.exe
```

reveals several useful attributes including:

- File size
- Digital signature
- Owner
- Hashes

The MD5 hash displayed is:

```
890a58f200dfff23165df9e1b088e58f
```

### Answer

```
890a58f200dfff23165df9e1b088e58f
```

---

# Why Hashes Matter

Cryptographic hashes uniquely identify files.

Investigators commonly submit hashes to:

- VirusTotal
- MalwareBazaar
- Threat Intelligence Platforms
- Internal IOC databases

This quickly determines whether malware has been observed previously.

---

# Question 6

## Determine the File Size

Reviewing the same file properties reveals:

```
164 Kilobytes
```

### Answer

```
164 KB
```

Although simple, file size can help analysts distinguish between malware variants and confirm file integrity during investigations.

---

# Question 7

## Determine the Ransomware Extension

The next phase of the investigation focuses on identifying the ransomware's impact.

Searching the filesystem quickly reveals hundreds of files sharing a common extension.

Example:

```
sdl-redline.zip.t48s39la
```

The final extension added to encrypted files is:

```
.t48s39la
```

The same extension appears across multiple directories including:

- Desktop
- Documents
- Pictures
- Music

This consistent renaming pattern confirms successful ransomware encryption.

### Answer

```
.t48s39la
```

---

# Why File Extensions Matter

Most ransomware families append unique extensions to encrypted files.

These extensions allow investigators to:

- Identify ransomware families.
- Search public decryptors.
- Correlate threat intelligence.
- Assist victims during recovery.

---

# Question 8

## Determine the Number of Encrypted Files

The final objective for this phase is determining the overall encryption impact.

Within Redline's **Host Search** view, changing the timeline filter to:

```
Modified

↓

Changed
```

displays all recently modified files.

The total count shown is:

```
48
```

### Answer

```
48
```

This indicates that forty-eight files were encrypted and renamed during the ransomware attack.

---

# Investigation Progress

At this stage we have successfully:

- Identified the compromised employee.
- Determined the operating system.
- Identified the malicious executable.
- Located the malware download source.
- Collected the malware MD5 hash.
- Determined the malware size.
- Identified the ransomware file extension.
- Calculated the total number of encrypted files.

The next phase of the investigation focuses on identifying the modified wallpaper, ransom note, hidden lock file, decryptor binary, attacker infrastructure, malware family attribution, Indicators of Compromise (IOCs), MITRE ATT&CK mapping, incident timeline, and security recommendations.

# Question 9

## Identify the Modified Wallpaper

One of the common techniques used by ransomware operators is replacing the victim's desktop wallpaper with a ransom message or warning.

The next objective is identifying the image that replaced the original wallpaper.

Using Redline's **File System** search and filtering for bitmap files (`.bmp`) quickly reveals a suspicious image inside the temporary directory.

The file path is:

```
C:\Users\John Coleman\AppData\Local\Temp\hk8.bmp
```

This bitmap was used by the ransomware as the victim's new desktop wallpaper after encryption completed.

### Answer

```
C:\Users\John Coleman\AppData\Local\Temp\hk8.bmp
```

---

# Why Attackers Change Wallpapers

Changing the desktop wallpaper ensures the victim immediately notices the ransomware attack.

Instead of waiting for the user to discover encrypted files, the attacker displays:

- Payment instructions
- Threat messages
- Contact information
- Cryptocurrency wallet details

This increases the likelihood that the victim will read the ransom instructions.

---

# Question 10

## Identify the Ransom Note

Almost every ransomware family leaves a ransom note after completing encryption.

Searching the Desktop directory reveals the newly created text file.

The filename is:

```
t48s39la-readme.txt
```

### Answer

```
t48s39la-readme.txt
```

The ransom note typically contains:

- Victim ID
- Payment instructions
- Contact information
- Tor website
- Decryption instructions

These notes often serve as valuable threat intelligence because they can help identify the ransomware family responsible for the attack.

---

# Question 11

## Identify the File Created Inside Favorites

The ransomware created an unusual directory under the user's Favorites folder.

Navigating to:

```
C:\Users\John Coleman\Favorites\
```

reveals:

```
Links for United States
```

Inside this folder a single encrypted shortcut file exists.

```
GobiernoUSA.gov.url.t48s39la
```

### Answer

```
GobiernoUSA.gov.url.t48s39la
```

This confirms that ransomware encrypted browser shortcut files in addition to traditional documents.

---

# Question 12

## Identify the Hidden Lock File

Many ransomware families generate lock files during execution.

These files may indicate:

- Encryption completion
- Running instances
- Victim identification
- Infection status

Searching the Desktop reveals a hidden file.

```
d60dff40.lock
```

Properties:

- Hidden
- Archive
- 0 Bytes

### Answer

```
d60dff40.lock
```

Although empty, this file likely served as an internal marker used by the ransomware during execution.

---

# Question 13

## Identify the Decryptor Hash

After the infection, the victim attempted to recover their encrypted files.

A decryptor executable was downloaded onto the Desktop.

```
d.e.c.r.yp.tor.exe
```

Inspecting the file properties reveals its MD5 hash.

```
f617af8c0d276682fdf528bb3e72560b
```

### Answer

```
f617af8c0d276682fdf528bb3e72560b
```

This demonstrates a common real-world response where victims attempt to locate publicly available decryptors before considering payment.

---

# Question 14

## Identify the Decryption Portal

The ransom note instructs victims to visit a website where one encrypted file can supposedly be decrypted free of charge.

Browser history confirms that the victim attempted to access this service.

The recorded URL is:

```
http://decryptor.top/644E7C8EFA02FBB7
```

### Answer

```
http://decryptor.top/644E7C8EFA02FBB7
```

This URL contains a unique victim identifier generated by the ransomware operator.

Such URLs are frequently used to:

- Verify victim identity.
- Display ransom amounts.
- Upload encrypted files.
- Negotiate payments.

---

# Question 15

## Identify the Malware Family

The final objective is attributing the malware.

Using the malware MD5 hash obtained earlier and searching public threat intelligence platforms such as VirusTotal identifies the ransomware family.

Several aliases are associated with this malware.

The three names are:

```
REvil
Sodin
Sodinokibi
```

### Answer

```
REvil,Sodin,Sodinokibi
```

These names all refer to the same ransomware family.

REvil (also known as **Sodinokibi**) became one of the most notorious ransomware operations in history due to its large-scale ransomware-as-a-service (RaaS) campaigns targeting organizations worldwide.

---

# Incident Timeline

The forensic evidence allows reconstruction of the attack sequence.

```
Victim Downloads Fake WinRAR
            │
            ▼
WinRAR2021.exe Executed
            │
            ▼
Ransomware Starts
            │
            ▼
Files Encrypted
            │
            ▼
.t48s39la Extension Added
            │
            ▼
Wallpaper Replaced
            │
            ▼
Ransom Note Created
            │
            ▼
Hidden Lock File Created
            │
            ▼
Victim Downloads Decryptor
            │
            ▼
Victim Visits Decryption Portal
```

---

# Indicators of Compromise (IOCs)

| Indicator | Value |
|-----------|-------|
| Victim | John Coleman |
| Operating System | Windows 7 Home Premium SP1 |
| Malicious Binary | WinRAR2021.exe |
| Download URL | `http://192.168.75.129:4748/Documents/WinRAR2021.exe` |
| Malware MD5 | 890a58f200dfff23165df9e1b088e58f |
| File Size | 164 KB |
| Ransomware Extension | `.t48s39la` |
| Encrypted Files | 48 |
| Wallpaper | `C:\Users\John Coleman\AppData\Local\Temp\hk8.bmp` |
| Ransom Note | `t48s39la-readme.txt` |
| Hidden Lock File | `d60dff40.lock` |
| Decryptor MD5 | f617af8c0d276682fdf528bb3e72560b |
| Decryption Portal | `http://decryptor.top/644E7C8EFA02FBB7` |
| Malware Family | REvil / Sodin / Sodinokibi |

---

# MITRE ATT&CK Mapping

| Technique | ATT&CK ID |
|------------|-----------|
| User Execution | T1204 |
| Phishing / Malicious File Delivery | T1566 |
| Data Encrypted for Impact | T1486 |
| Indicator Removal (File Modification) | T1070 |
| Modify System Image (Wallpaper) | T1491.001 |
| Inhibit System Recovery | T1490 |
| Impact | TA0040 |

---

# Detection Opportunities

Security teams could identify this attack by monitoring:

- Executables downloaded from untrusted sources.
- Suspicious binaries masquerading as legitimate software.
- Large numbers of file rename operations.
- Sudden creation of unfamiliar file extensions.
- Desktop wallpaper modifications.
- Creation of ransom notes.
- Hidden lock files.
- Execution of unsigned binaries.
- Browser visits to known ransomware payment portals.

Modern EDR platforms can also detect abnormal filesystem activity associated with ransomware encryption before significant damage occurs.

---

# Security Recommendations

## Keep Systems Updated

The compromised workstation was running **Windows 7**, an operating system that no longer receives regular security updates.

Organizations should migrate unsupported systems to currently supported operating systems.

---

## Application Control

Restrict execution of unknown binaries using:

- Windows Defender Application Control (WDAC)
- AppLocker
- Software Restriction Policies

This prevents unauthorized executables from launching.

---

## User Awareness Training

Employees should be educated to recognize fake installers and suspicious downloads.

Only software obtained from trusted vendors should be executed.

---

## Endpoint Detection and Response

Deploy EDR solutions capable of detecting:

- Mass file encryption.
- Ransomware behavior.
- Unsigned executable execution.
- Suspicious filesystem modifications.

Behavioral detection is often more effective than signature-based antivirus.

---

## Offline Backups

Maintain secure, offline backups of critical data.

Regularly test backup restoration procedures to ensure business continuity following a ransomware incident.

---

## Threat Intelligence

Submit collected hashes and indicators to threat intelligence platforms such as VirusTotal to identify malware families and correlate ongoing campaigns.

---

# Lessons Learned

This investigation demonstrates the importance of host-based forensic analysis during ransomware incidents.

Using Mandiant Redline, we successfully reconstructed the attack without requiring live access to the compromised machine. By examining system information, browser history, download records, and filesystem artifacts, we identified the initial infection vector, the malicious executable, the encrypted files, and the ransomware family responsible.

The room also highlights the value of collecting Indicators of Compromise (IOCs). File hashes, malicious URLs, ransomware extensions, and ransom notes can all be shared with security teams to improve future detection and response efforts.

Perhaps the most important takeaway is that ransomware investigations rely heavily on careful artifact analysis. Every file, timestamp, and registry entry contributes to rebuilding the attack timeline and understanding the adversary's actions.

---

# Conclusion

**REvil Corp** is an excellent introductory Digital Forensics and Incident Response (DFIR) challenge that demonstrates how investigators can reconstruct a ransomware incident using host forensic artifacts. Through systematic analysis with Mandiant Redline, we identified the compromised employee, determined the operating system, traced the malicious download, collected malware hashes, identified encrypted files, and attributed the attack to the **REvil (Sodinokibi)** ransomware family.

The investigation further revealed the ransomware's impact, including encrypted file extensions, desktop wallpaper modification, ransom note creation, hidden lock files, and the victim's unsuccessful attempt to recover files using a decryptor. By combining browser history, filesystem metadata, and threat intelligence, a complete timeline of the incident was reconstructed.

From a defensive perspective, this challenge reinforces the importance of user awareness, application control, endpoint detection, regular backups, and forensic readiness. Overall, REvil Corp provides an excellent introduction to host-based ransomware investigations and demonstrates the practical value of forensic tools such as **Mandiant Redline** during real-world incident response engagements.
