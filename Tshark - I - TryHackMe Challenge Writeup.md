# TShark: Challenge 1 - TryHackMe Walkthrough

Today we have another TryHackMe Defensive Security challenge focused on network traffic analysis using **TShark**. In this room, we are provided with a packet capture file and must investigate suspicious network activity after the Threat Research Team identifies a potentially malicious domain.

The goal is to analyze the PCAP, investigate contacted domains, correlate findings with VirusTotal, and recover indicators of compromise (IOCs) that can be used for future detection and threat hunting.

---

# Scenario

An alert has been triggered:

```text
"The threat research team discovered a suspicious domain that could be a potential threat to the organisation."
```

The investigation has been assigned to us.

Evidence provided:

```text
teamwork.pcap
```

Location:

```bash
~/Desktop/exercise-files
```

Objectives:

* Investigate contacted domains
* Identify suspicious activity
* Validate indicators using VirusTotal
* Extract useful detection artifacts

---

# Initial Investigation

The first step was reviewing HTTP traffic within the PCAP.

Using TShark:

```bash
tshark -r teamwork.pcap http
```

While reviewing the HTTP requests, one domain immediately stood out due to its unusual structure.

---

# Question 1

## What is the full URL of the malicious/suspicious domain?

Reviewing the HTTP requests revealed the following suspicious URL:

```text
http://www.paypal.com4uswebappsresetaccountrecovery.timeseaways.com/
```

At first glance, it appears to reference PayPal.

However, closer inspection shows that the legitimate PayPal name is embedded within a completely different domain.

This is a common phishing technique designed to trick users into believing they are visiting a trusted service.

### Defanged Format

```text
www[.]paypal[.]com4uswebappsresetaccountrecovery[.]timeseaways[.]com
```

### Answer

```text
http://www.paypal.com4uswebappsresetaccountrecovery.timeseaways.com/
```

---

# URL Analysis

The domain uses several social engineering techniques:

```text
paypal
account
recovery
reset
```

These keywords are commonly associated with credential theft campaigns.

The attacker attempts to create a sense of legitimacy by embedding a trusted brand name inside the hostname.

---

# Question 2

## When was the URL first submitted to VirusTotal?

After searching the domain in VirusTotal:

```text
First Submission
2017-04-17 22:52:53 UTC
```

### Answer

```text
2017-04-17 22:52:53 UTC
```

---

# Threat Intelligence Findings

VirusTotal also revealed:

```text
Last Submission:
2026-06-17 07:02:49 UTC

Last Analysis:
2026-06-17 07:02:49 UTC
```

This indicates the domain has remained visible to security researchers for several years.

---

# Question 3

## Which known service was the domain trying to impersonate?

Reviewing the hostname:

```text
www.paypal.com4uswebappsresetaccountrecovery.timeseaways.com
```

The target of impersonation is obvious.

### Answer

```text
PayPal
```

The attacker is attempting to abuse user trust in the PayPal brand.

---

# Question 4

## What is the IP address of the malicious domain?

Reviewing the HTTP conversations within the capture:

```text
184.154.127.226
```

Defanged:

```text
184[.]154[.]127[.]226
```

### Answer

```text
184[.]154[.]127[.]226
```

---

# Traffic Analysis

The suspicious communications show:

```text
Victim Host
      ↓
Malicious Domain
      ↓
184.154.127.226
```

The destination server responds with:

```text
HTTP/1.1 200 OK
```

indicating successful communication between the victim and the phishing infrastructure.

---

# Question 5

## What email address was used?

Searching for submitted form data:

```bash
tshark -r teamwork.pcap -Y urlencoded-form
```

Results:

```text
POST /inc/visit.php
POST /inc/login.php
```

The presence of login-related requests strongly suggests credential harvesting activity.

To extract submitted parameters:

```bash
tshark -r teamwork.pcap -Y urlencoded-form -V | grep @
```

Output:

```text
Form item: "user" = "johnny5alive@gmail.com"
```

### Defanged Format

```text
johnny5alive[at]gmail[.]com
```

### Answer

```text
johnny5alive[at]gmail[.]com
```

---

# Credential Harvesting Evidence

The captured HTTP requests reveal that the victim interacted with the phishing page and submitted information through:

```text
/inc/login.php
```

This is a strong indicator that the phishing infrastructure was actively collecting credentials.

Observed flow:

```text
Victim Visits Phishing Site
          ↓
Fake PayPal Page Displayed
          ↓
User Enters Information
          ↓
POST Request Sent
          ↓
Credentials Collected
```

---

# Indicators of Compromise (IOCs)

## Malicious Domain

```text
www[.]paypal[.]com4uswebappsresetaccountrecovery[.]timeseaways[.]com
```

## Malicious IP

```text
184[.]154[.]127[.]226
```

## Impersonated Brand

```text
PayPal
```

## Victim Email

```text
johnny5alive[at]gmail[.]com
```

## Suspicious Endpoints

```text
/ins/visit.php
/inc/login.php
```

---

# Attack Chain Reconstruction

```text
Victim Browses Internet
          ↓
Visits Phishing Domain
          ↓
Fake PayPal Login Page
          ↓
User Trusts Website
          ↓
Credentials Submitted
          ↓
HTTP POST Request
          ↓
Attacker Receives Data
```

---

# Security Lessons Learned

This challenge highlights several common phishing indicators:

### Suspicious Domain Names

Attackers often embed legitimate company names inside longer malicious domains.

Example:

```text
paypal.com4uswebappsresetaccountrecovery.timeseaways.com
```

### Credential Harvesting

Monitoring POST requests can quickly reveal attempts to collect user credentials.

### Threat Intelligence Correlation

VirusTotal allows analysts to:

* Identify malicious domains
* Investigate infrastructure
* Determine historical activity
* Collect additional indicators

### Detection Opportunities

Useful detection artifacts include:

* Malicious domains
* Associated IP addresses
* HTTP POST requests
* Credential submission endpoints

---

# Answers Summary

| Question                    | Answer                                                               |
| --------------------------- | -------------------------------------------------------------------- |
| Suspicious URL              | http://www.paypal.com4uswebappsresetaccountrecovery.timeseaways.com/ |
| First VirusTotal Submission | 2017-04-17 22:52:53 UTC                                              |
| Impersonated Service        | PayPal                                                               |
| Malicious IP                | 184[.]154[.]127[.]226                                                |
| Email Address               | johnny5alive[at]gmail[.]com                                          |

---

# Conclusion

This investigation focused on identifying a phishing campaign through packet analysis. By examining HTTP traffic within the PCAP, investigating the suspicious domain in VirusTotal, and extracting submitted credentials from captured requests, it was possible to confirm malicious activity and recover multiple indicators of compromise.

The challenge demonstrates practical SOC investigation techniques involving phishing analysis, packet inspection, threat intelligence enrichment, and IOC extraction. These skills are essential when responding to phishing incidents and validating alerts generated by threat research teams.
