# TryHackMe - REvil Corp Walkthrough

## Challenge Information

| Category   | Value                                                                                                        |
| ---------- | ------------------------------------------------------------------------------------------------------------ |
| Platform   | TryHackMe                                                                                                    |
| Challenge  | REvil Corp                                                                                                   |
| Difficulty | Medium                                                                                                       |
| Type       | Digital Forensics & Incident Response                                                                        |
| Tool Used  | Mandiant Redline                                                                                             |
| Objective  | Investigate a ransomware infection and identify attacker artifacts, victim activity, and malware indicators. |

---

# Scenario

One of the employees at Lockman Group reported that all of his files had suddenly been renamed with an unfamiliar extension. After a quick inspection, the IT department suspected ransomware activity and escalated the case to the Incident Response team.

As the incident responder, our task is to investigate the compromised workstation using a forensic image loaded into Mandiant Redline and identify all indicators of compromise associated with the attack.

---

# About Mandiant Redline

Mandiant Redline is a free incident response and forensic analysis platform used by DFIR analysts to investigate compromised systems.

It allows investigators to:

* Analyze memory and system artifacts
* Review browser history
* Examine downloaded files
* Investigate timelines
* Review user activity
* Detect malware persistence
* Correlate indicators of compromise

For this challenge, the supplied analysis file already contains collected forensic artifacts from the infected workstation.

---

# Q1. What is the compromised employee's full name?

After opening the analysis file in Redline:

```text
System Information
    └── User Information
```

We find the logged-in user details.

**Answer**

```text
John Coleman
```

---

# Q2. What is the operating system of the compromised host?

Still under:

```text
System Information
```

Operating System:

```text
Windows 7 Home Premium 7601 Service Pack 1
```

**Answer**

```text
Windows 7 Home Premium 7601 Service Pack 1
```

---

# Q3. What is the name of the malicious executable that the user opened?

After exploring:

```text
File Download History
```

A suspicious executable appears.

**Answer**

```text
WinRAR2021.exe
```

This was the initial payload executed by the victim.

---

# Q4. What is the full URL used to download the malicious binary?

Within the same download history entry Redline shows the source URL.

**Answer**

```text
http://192.168.75.129:4748/Documents/WinRAR2021.exe
```

This confirms the malware was manually downloaded rather than delivered through a browser exploit.

---

# Q5. What is the MD5 hash of the binary?

Navigating deeper into the downloaded file metadata reveals the file hashes.

**Answer**

```text
890a58f200dfff23165df9e1b088e58f
```

---

# Q6. What is the size of the binary?

The file metadata shows:

**Answer**

```text
164 KB
```

---

# Q7. What extension were the victim's files renamed to?

After examining the user's Desktop and encrypted files:

**Answer**

```text
.t48s39la
```

This extension was appended to all encrypted files.

Example:

```text
report.docx
        ↓
report.docx.t48s39la
```

---

# Q8. How many files were renamed with that extension?

Using:

```text
Timeline
```

Filtering for:

```text
.t48s39la
```

and reviewing modified files reveals:

**Answer**

```text
48
```

A total of 48 files were encrypted.

---

# Q9. What is the full path of the wallpaper changed by the attacker?

Attackers commonly replace desktop wallpapers with ransom messages.

Searching timeline artifacts for image files reveals:

**Answer**

```text
C:\Users\John Coleman\AppData\Local\Temp\hk8.bmp
```

---

# Q10. What ransom note was left on the Desktop?

Reviewing Desktop artifacts reveals the ransom note.

**Answer**

```text
t48s39la-readme.txt
```

---

# Q11. What file was left inside "Links for United States"?

Location:

```text
C:\Users\John Coleman\Favorites\Links for United States\
```

File discovered:

**Answer**

```text
GobiernoUSA.gov.url.t48s39la
```

The file itself was also encrypted by the ransomware.

---

# Q12. What hidden 0-byte file was created on the Desktop?

Examining Desktop artifacts and sorting by file size reveals a hidden file.

**Answer**

```text
d60dff40.lock
```

Size:

```text
0 bytes
```

Lock files are commonly used by ransomware to track encryption status.

---

# Q13. What is the MD5 hash of the decryptor downloaded by the user?

The victim later downloaded a decryptor hoping to recover files.

Reviewing downloaded files reveals:

```text
decryptor.exe
```

Associated MD5:

**Answer**

```text
f617af8c0d276682fdf528bb3e72560b
```

---

# Q14. What URL did the victim visit for free decryption?

The ransom note offered one free file decryption through a public portal.

Examining browser history reveals the victim attempted to visit it.

**Answer**

```text
http://decryptor.top/644E7C8EFA02FBB7
```

This matches the victim identifier generated by the ransomware.

---

# Q15. What three names are associated with this ransomware family?

Using indicators collected during the investigation and external threat intelligence research, the malware family was identified.

Common names used by security vendors:

**Answer**

```text
REvil,Sodin,Sodinokibi
```

Alphabetical order:

```text
REvil
Sodin
Sodinokibi
```

---

# Malware Attribution

The ransomware identified in this challenge is:

```text
REvil (Sodinokibi)
```

REvil became one of the most notorious Ransomware-as-a-Service (RaaS) operations in the world and was responsible for numerous high-profile attacks against:

* Kaseya
* JBS Foods
* Acer
* Travelex
* Hundreds of managed service providers

---

# Indicators of Compromise

## Malicious Executable

```text
WinRAR2021.exe
```

## Download Source

```text
http://192.168.75.129:4748/Documents/WinRAR2021.exe
```

## Malware MD5

```text
890a58f200dfff23165df9e1b088e58f
```

## Ransom Extension

```text
.t48s39la
```

## Ransom Note

```text
t48s39la-readme.txt
```

## Decryption Portal

```text
http://decryptor.top/644E7C8EFA02FBB7
```

## Decryptor MD5

```text
f617af8c0d276682fdf528bb3e72560b
```

---

# Attack Timeline

```text
Victim downloads WinRAR2021.exe
            ↓
Malware executed
            ↓
Files encrypted
            ↓
.t48s39la extension appended
            ↓
Wallpaper changed
            ↓
Ransom note dropped
            ↓
48 files encrypted
            ↓
Victim downloads decryptor
            ↓
Victim accesses attacker portal
```

---

# Lessons Learned

### User Awareness

The attack originated from a manually downloaded executable pretending to be legitimate software.

### File Download Monitoring

Monitoring downloads from unknown internal IP addresses could have identified the threat earlier.

### Ransomware Indicators

Common indicators include:

* Mass file renaming
* New file extensions
* Wallpaper modifications
* Creation of ransom notes
* Browser activity toward payment portals

### Incident Response

Tools like Mandiant Redline significantly accelerate ransomware investigations by consolidating:

* Browser history
* File system changes
* Timeline analysis
* User activity
* Malware metadata

---

# Conclusion

The investigation revealed that John Coleman executed a malicious executable named **WinRAR2021.exe**, which deployed the **REvil/Sodinokibi ransomware**. The malware encrypted 48 files, renamed them using the **.t48s39la** extension, modified the desktop wallpaper, dropped a ransom note, and directed the victim to a decryption portal. Through Redline analysis, all critical indicators of compromise were successfully identified, allowing attribution of the attack to the REvil ransomware family.
