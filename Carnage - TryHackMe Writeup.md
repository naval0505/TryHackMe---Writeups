# TryHackMe - Carnage Writeup

## Scenario

Today we are solving another Defensive Security challenge from TryHackMe called **Carnage**. In this challenge we are provided with a packet capture (PCAP) file from a real-world malware infection scenario and must investigate the network traffic to identify the infection chain, malicious infrastructure, malware communication, and attacker activity.

### Incident Scenario

Eric Fischer from the Purchasing Department at Bartell Ltd received an email from a trusted contact containing a Microsoft Word document attachment. After opening the document, Eric accidentally clicked **Enable Content**, allowing the embedded malicious code to execute.

Shortly afterward, the Security Operations Center (SOC) received alerts indicating suspicious outbound connections originating from Eric's workstation. The network traffic was captured and provided to us for investigation.

Our task is to analyze the packet capture and determine exactly what happened during the compromise.

---

# Environment

The investigation is performed using **Wireshark**.

One useful change before beginning analysis is modifying the timestamp display format.

Navigate to:

```text
Edit → Preferences → Time Display Format
```

Select:

```text
UTC Date and Time
```

This makes correlation of events significantly easier throughout the investigation.

---

# Question 1

## What was the date and time for the first HTTP connection to the malicious IP?

After examining the initial HTTP communications and converting timestamps to UTC Date and Time format, the first malicious HTTP connection occurred at:

```text
2021-09-24 16:44:38
```

### Answer

```text
2021-09-24 16:44:38
```

---

# Question 2

## What is the name of the ZIP file that was downloaded?

Following the first malicious HTTP request reveals a file download.

The downloaded archive is:

```text
documents.zip
```

### Answer

```text
documents.zip
```

---

# Question 3

## What was the domain hosting the malicious ZIP file?

Examining the HTTP GET request shows the full URL used for the download.

```text
http://attirenepal.com/incidunt-consequatur/documents.zip
```

The hosting domain is:

```text
attirenepal.com
```

### Answer

```text
attirenepal.com
```

---

# Question 4

## Without downloading the file, what is the name of the file inside the ZIP archive?

Locate the packet responsible for downloading **documents.zip**.

Right-click the packet and select:

```text
Follow → HTTP Stream
```

Within the returned content we can observe the filename stored inside the archive.

```text
chart-1530076591.xls
```

### Answer

```text
chart-1530076591.xls
```

---

# Question 5

## What is the name of the webserver hosting the malicious ZIP file?

Looking at the HTTP response headers for the ZIP download reveals the server software.

```text
Server: LiteSpeed
```

### Answer

```text
LiteSpeed
```

---

# Question 6

## What is the version of the webserver?

Within the same HTTP response we can observe additional version information.

```text
PHP/7.2.34
```

### Answer

```text
PHP/7.2.34
```

---

# Question 7

## Malicious files were downloaded from multiple domains. What were the three domains involved?

To identify additional malicious infrastructure, inspect TLS handshakes.

Filter:

```text
tls.handshake.type == 1
```

This displays Client Hello packets and reveals requested domains.

The three domains involved were:

```text
finejewels.com
thietbiagt.com
new.americold.com
```

### Answer

```text
finejewels.com
thietbiagt.com
new.americold.com
```

---

# Question 8

## Which Certificate Authority issued the SSL certificate to the first domain?

Using the TLS handshake packets associated with the first domain and following the TCP stream reveals certificate information.

The SSL certificate was issued by:

```text
GoDaddy
```

### Answer

```text
GoDaddy
```

---

# Question 9

## What are the two Cobalt Strike C2 server IP addresses?

By analyzing HTTP traffic and validating findings using VirusTotal Community intelligence, two Cobalt Strike command-and-control servers were identified.

```text
185.125.204.174
185.106.96.158
```

VirusTotal community reports confirmed both IPs were associated with Cobalt Strike infrastructure.

### Answer

```text
185.125.204.174
185.106.96.158
```

---

# Question 10

## What is the Host header for the first Cobalt Strike IP?

Investigating the first Cobalt Strike IP within VirusTotal's Community section reveals the Host header used.

```text
ocsp.verisign.com
```

### Answer

```text
ocsp.verisign.com
```

---

# Question 11

## What is the domain name associated with the first Cobalt Strike server?

VirusTotal Community intelligence reveals:

```text
survmeter.live
```

### Answer

```text
survmeter.live
```

---

# Question 12

## What is the domain name associated with the second Cobalt Strike server?

Further investigation of the second C2 IP identifies:

```text
securitybusinpuff.com
```

### Answer

```text
securitybusinpuff.com
```

---

# Question 13

## What is the domain used during post-infection traffic?

To identify post-compromise communication, filter for POST requests.

```text
http.request.method == "POST"
```

Reviewing these packets reveals communication with:

```text
maldivehost.net
```

### Answer

```text
maldivehost.net
```

---

# Question 14

## What are the first eleven characters sent to the malicious domain?

Examining the first POST request to the malicious domain reveals:

```text
http://maldivehost.net/zLIisQRWZI9/OQsaDixzHTgtfjMcGypGenpldWF5eWV9f3k=
```

The first eleven characters are:

```text
zLIisQRWZI9
```

### Answer

```text
zLIisQRWZI9
```

---

# Question 15

## What was the length of the first packet sent to the C2 server?

Inspection of the first beacon packet shows a packet length of:

```text
281
```

### Answer

```text
281
```

---

# Question 16

## What was the Server header for the malicious domain?

Following the HTTP stream of the malicious communication reveals:

```text
Apache/2.4.49 (cPanel) OpenSSL/1.1.1l mod_bwlimited/1.4
```

### Answer

```text
Apache/2.4.49 (cPanel) OpenSSL/1.1.1l mod_bwlimited/1.4
```

---

# Question 17

## When did the DNS query for the public IP lookup occur?

A useful filter:

```text
dns.qry.name contains "api"
```

This reveals requests made to an external IP lookup service.

The query occurred at:

```text
2021-09-24 17:00:04
```

### Answer

```text
2021-09-24 17:00:04
```

---

# Question 18

## What was the queried domain?

The DNS request targeted:

```text
api.ipify.org
```

This service is commonly used by malware to determine the victim's public IP address.

### Answer

```text
api.ipify.org
```

---

# Question 19

## What was the first MAIL FROM address observed?

To identify spam activity, analyze SMTP traffic.

Within the SMTP conversation we find:

```text
MAIL FROM:<farshin@mailfa.com>
```

### Answer

```text
farshin@mailfa.com
```

---

# Question 20

## How many SMTP packets were observed?

Apply an SMTP display filter and observe the packet count shown by Wireshark.

```text
Displayed: 1439
```

### Answer

```text
1439
```

---

# Infection Timeline

```text
User opens malicious Word document
            ↓
User enables macros/content
            ↓
documents.zip downloaded
            ↓
chart-1530076591.xls executed
            ↓
Additional payloads retrieved from multiple domains
            ↓
TLS communication established
            ↓
Cobalt Strike beacon activity begins
            ↓
Victim public IP checked via api.ipify.org
            ↓
Post-infection communication with maldivehost.net
            ↓
SMTP spam activity observed
```

---

# Indicators of Compromise (IOCs)

## Malicious Domains

```text
attirenepal.com
finejewels.com
thietbiagt.com
new.americold.com
survmeter.live
securitybusinpuff.com
maldivehost.net
```

## Malicious Files

```text
documents.zip
chart-1530076591.xls
```

## Cobalt Strike Infrastructure

```text
185.125.204.174
185.106.96.158
```

## External Service Abuse

```text
api.ipify.org
```

## Suspicious Email

```text
farshin@mailfa.com
```

---

# Conclusion

The packet capture clearly demonstrates a malware infection that began with a malicious Office document delivered through email. After the victim enabled content, malware downloaded additional payloads, contacted multiple malicious domains, and established communication with Cobalt Strike command-and-control servers. The malware later performed external IP discovery, communicated with post-infection infrastructure, and generated SMTP spam activity. Through careful analysis of HTTP, TLS, DNS, and SMTP traffic, the complete attack chain and associated indicators were successfully identified.
