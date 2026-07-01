# TryHackMe – KitKat Writeup

## Scenario

Today we are solving another **TryHackMe Defensive Security** challenge named **KitKat**.

Unlike traditional malware or intrusion investigations, this room focuses on **OSINT**, **digital forensics**, **log analysis**, **email header analysis**, and **timeline reconstruction**. The objective is to identify the insider responsible for leaking confidential shipment information related to a fictional cargo theft investigation.

---

# Scenario Overview

The challenge is based on a fictional investigation inspired by a real cargo theft.

A refrigerated truck transporting over **400,000 KitKat products** disappeared while travelling between Italy and Poland. Investigators believe the theft was carefully planned rather than opportunistic.

ECTA (European Cargo Threat Assessment) obtained an anonymous email sent to a journalist shortly after the theft. Several forensic artifacts have also been recovered, including:

- Anonymous email (.eml)
- Petrol station photograph
- Company access logs
- Employee database
- Additional supporting evidence

Our task is to analyse each artifact, reconstruct the events, identify the insider responsible for leaking sensitive shipment information, and determine the identity of the anonymous whistleblower.

---

# Q1 — Which VPN service was used to send the anonymous email?

The investigation begins by extracting the provided archive and opening the **.eml** email file.

Instead of reading only the visible email, we inspect the **original message source** using Thunderbird to analyse the email headers.

Inside the header we find:

```
Received:
from [193.32.249.132]
```

The public IP address used to send the email is:

```
193.32.249.132
```

This address is investigated using publicly available IP intelligence.

Tools used:

- IPInfo
- RIPE WHOIS

WHOIS information returns:

```
193.32.249.0/24

AS39351
```

The ASN information identifies the VPN provider used by the anonymous sender.

This confirms that the sender attempted to hide their real location before contacting the journalist.

---

# Q2 — What is the full street address of the petrol station?

The supplied PNG image is opened for examination.

Rather than containing hidden metadata, the photograph itself clearly reveals the location.

Visible indicators include:

- ORLEN branding
- Street name
- Building number

Cross-referencing these with Google Maps confirms the location as:

```
Kroměřížská 1281

768 24 Hulín

Czechia
```

This represents the last confirmed location where the stolen refrigerated vehicle was observed.

---

# Q3 — At what time was the confidential shipment exported?

Next we inspect the supplied access log.

```
access_log.csv
```

Instead of looking only for failed logins, we analyse every user action chronologically.

Most users performed normal activities:

- View
- Edit

However one entry immediately stands out.

User:

```
BR-0291
```

performed:

```
EXPORT
```

on a sensitive route PDF.

Exporting confidential logistics documents is highly unusual compared to normal viewing behaviour.

The export occurred at:

```
22:14:09
```

Additional observations strengthen suspicion.

The same employee experienced authentication failures one day earlier.

Later, access permissions appear to have been revoked, suggesting administrators noticed suspicious behaviour.

This becomes the first major indicator of insider involvement.

---

# Q4 — Which employee sent the anonymous email?

To identify the whistleblower, we compare the employee database with the activity logs.

The investigation shows that:

```
BR-0312
```

was active shortly after the suspicious export event.

Timeline observations:

- Active late evening
- Worked regular late shifts
- Present after BR-0291 exported the confidential route

Unlike BR-0291, there is no evidence that BR-0312 stole data.

Instead, their behaviour suggests they witnessed suspicious activity and decided to report it externally.

The email itself also hints at distrust toward internal management, explaining why the employee contacted a journalist instead of reporting internally.

Therefore the anonymous sender is:

```
BR-0312
```

---

# Q5 — Which employee leaked the shipment information?

Returning to the access logs, the insider responsible becomes clear.

Only one employee exported the confidential transportation document.

Employee:

```
BR-0291
```

Action:

```
EXPORT
```

Time:

```
22:14:09
```

This export occurred immediately before the cargo theft, making BR-0291 the primary suspect responsible for leaking the shipment route.

---

# Q6 — What is the leaker's full name?

The employee ID alone is insufficient.

Additional OSINT investigation is required.

Public intelligence sources are used to correlate the leaked email address and available identifiers.

Tools used include:

- Epieos
- Google Maps
- Public OSINT resources

By combining employee records with external intelligence, the full identity of the leaker can be determined.

This demonstrates how internal company records and public information together can identify insider threats.

---

# Investigation Timeline

```
Employee Login Activity
            │
            ▼
Review access_log.csv
            │
            ▼
BR-0291 exports confidential route
            │
            ▼
Sensitive shipment information exposed
            │
            ▼
Cargo truck disappears
            │
            ▼
BR-0312 notices suspicious behaviour
            │
            ▼
Anonymous email sent through VPN
            │
            ▼
Journalist receives leak
            │
            ▼
Digital forensic investigation begins
```

---

# Evidence Collected

### Email Analysis

- Email headers
- Source IP
- VPN attribution
- SMTP routing

---

### Image Analysis

- ORLEN petrol station
- Visual location identification
- Google Maps verification

---

### Log Analysis

- Employee authentication history
- User activity timeline
- Export operations
- Suspicious late-night access

---

### Employee Records

- Employee IDs
- Job roles
- Working hours
- Correlation with log events

---

### OSINT

- WHOIS
- RIPE Database
- IPInfo
- Google Maps
- Epieos

---

# Tools Used

- Thunderbird
- IPInfo
- RIPE WHOIS
- Google Maps
- Epieos
- Spreadsheet Viewer
- CSV Analysis
- Browser Developer Tools

---

# Skills Practiced

- Email Header Analysis
- IP Attribution
- VPN Identification
- Timeline Reconstruction
- Insider Threat Investigation
- Employee Correlation
- Log Analysis
- OSINT Investigation
- Digital Evidence Correlation
- Incident Investigation

---

# Lessons Learned

This investigation demonstrates that insider threat cases rarely rely on a single piece of evidence. Instead, investigators must correlate multiple independent artifacts—including email headers, access logs, employee records, images, and publicly available intelligence—to reconstruct the complete sequence of events. A single unusual action, such as exporting a confidential document outside normal working hours, becomes highly significant when combined with later events such as the cargo theft and an anonymous email sent through a VPN. The challenge highlights the importance of timeline analysis, behavioural anomalies, and OSINT techniques when investigating data leaks and insider threats.

---

# Conclusion

The KitKat challenge focuses on digital forensics and OSINT rather than malware analysis. Through systematic examination of email headers, IP intelligence, access logs, employee activity, and supporting evidence, the investigation identifies both the employee responsible for leaking confidential shipment information and the individual who anonymously reported the incident. The room reinforces the investigative process used by defensive security analysts to correlate evidence, establish timelines, and distinguish malicious insider activity from legitimate whistleblowing.
