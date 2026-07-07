# TryHackMe - CAPTCHApocalypse

> **Category:** Linux | Web Application Penetration Testing (WAPT)
>
> **Difficulty:** Medium
>
> **Objective:** Analyze the login mechanism of a protected web application, understand the client-side authentication workflow, automate CAPTCHA solving, and successfully authenticate to retrieve the challenge flag.

---

# Scenario

In this challenge, we are presented with a Linux-based web application protected by a CAPTCHA-based authentication system. Unlike traditional login portals, the application encrypts authentication data before sending it to the server, making manual interception and modification significantly more difficult.

The objective is to understand how the client-side authentication process works, analyze the JavaScript responsible for encryption, automate the CAPTCHA solving process, and successfully authenticate to the application.

This room focuses primarily on:

- Web Application Reconnaissance
- JavaScript Analysis
- Client-side Authentication Logic
- CAPTCHA Automation
- Selenium Browser Automation
- OCR (Optical Character Recognition)

---

# Investigation Methodology

```
Host Discovery
        │
        ▼
Port Enumeration
        │
        ▼
Service Enumeration
        │
        ▼
Application Analysis
        │
        ▼
Intercept Authentication
        │
        ▼
JavaScript Analysis
        │
        ▼
Client-side Encryption
        │
        ▼
Automation Development
        │
        ▼
CAPTCHA Solving
        │
        ▼
Successful Authentication
        │
        ▼
Flag Retrieval
```

---

# Target Information

Target IP

```text
10.49.138.196
```

---

# Initial Enumeration

As with every machine, the first step is identifying exposed services.

A full TCP scan was performed.

```bash
nmap -p- 10.49.138.196
```

Results:

```text
22/tcp

80/tcp
```

Only SSH and HTTP services were exposed.

The next step involved service and version detection.

---

# Service Enumeration

Command executed:

```bash
nmap -sC -sV 10.49.138.196
```

Discovered services:

| Port | Service | Version |
|------|----------|----------|
|22|SSH|OpenSSH 8.2p1|
|80|HTTP|Apache 2.4.41|

HTTP Information:

```text
Title

Login
```

Apache Version:

```text
Apache/2.4.41 (Ubuntu)
```

Additional observations:

- PHP Sessions enabled
- HttpOnly flag missing
- Standard HTTP methods supported

Since the application only exposed a login page, the investigation focused entirely on understanding the authentication process.

---

# Initial Application Analysis

The application was opened through Burp Suite with request interception enabled.

A normal login attempt was performed.

Instead of observing clear-text credentials being transmitted, the intercepted request contained a single encrypted data object.

Example:

```json
{
    "data":"WBH6rnyIe2z1Y+jb79gpR1LuggDV7zxBB29mUAOa7OxJIegaCX6bC6r6UORYT+sgjB59D//AbstMuWHB1OjhWHc08RboWPBqhos/fO1ss16HUvFi8gQUTKeZhydnAwTiwUbr2yDIr3XqlMDmtAu25G4HnMXnnDc5d7FTBvQ5ywfhdvSu3cnv3PLMenlXSlypnXtvj8971lvVgx0M+DLjc4C1AACD+JWUFSEY7ndWlikThZE+b6OZbLAuJlElD5MQ5JESeZn+fpBDm21AM4P5nFQr9x2BZR60QVoJrJck4utXrM2eI1ZrebnpAVDtvYuuEoE1Ocl4slqNxf/1mqFTrw=="
}
```

---

## Analysis

Unlike traditional login forms that transmit username and password directly, this application packages all authentication data into an encrypted payload before sending it to the backend.

This immediately prevents several common testing techniques including:

- Parameter tampering
- Credential manipulation
- Simple replay attacks

Because the transmitted data is encrypted, modifying requests directly within Burp Suite provides little useful information.

Instead, understanding the client-side implementation became the next objective.

---

# Authentication Flow

Several login attempts revealed two possible application responses.

Successful authentication redirected the user to:

```text
dashboard.php
```

Failed authentication redirected to:

```text
/index.php?error=true
```

This confirmed that authentication decisions occur after the encrypted payload is processed by the server.

---

# Source Code Inspection

The application's page source was reviewed to identify hidden fields and client-side functionality.

One hidden parameter immediately stood out.

```html
<input
type="hidden"
name="csrf_token"
id="csrf_token"
value="179d72f68d22dcbd66a72ba3a160ca51a8ee79eec538c8a3c4fbf0755bb316e5">
```

---

## Analysis

The login form includes a dynamically generated CSRF token.

This token is submitted together with the authentication request, providing protection against Cross-Site Request Forgery attacks.

Any automated solution therefore needs to retrieve a fresh CSRF token before each authentication attempt.

---

# JavaScript Analysis

Reviewing the page source also revealed an external JavaScript file.

```text
script.js
```

Since authentication requests were encrypted, this file became the primary target for analysis.

Inspecting the login function revealed the following.

```javascript
async function login() {

    const username =
    document.getElementById("username").value;

    const password =
    document.getElementById("password").value;

    const csrf_token =
    document.getElementById("csrf_token").value;

    const captcha_input =
    document.getElementById("captcha_input").value;

    ...
}
```

---

## Analysis

The login function first collects every value entered by the user.

Specifically:

- Username
- Password
- CSRF Token
- CAPTCHA Response

The JavaScript then processes these values before transmitting them to the backend.

Further inspection showed that the collected values are encrypted prior to being sent to:

```text
server.php
```

Only after successful validation does the application redirect users to:

```text
dashboard.php
```

This explains why Burp Suite intercepted only a single encrypted object instead of the individual login parameters.

Rather than attacking the encrypted request directly, it became more practical to automate the browser itself and allow the application's own JavaScript to perform the encryption.

---

# CAPTCHA Challenge

The application additionally required users to solve a CAPTCHA before authentication.

Because the CAPTCHA changed on every request, manual testing quickly became inefficient.

Instead of attempting to reverse the encryption process, the login workflow was automated.

---

# Automation Approach

Browser automation was implemented using Selenium.

The automation process performs the following tasks:

1. Open the login page.
2. Retrieve the current CSRF token.
3. Capture the generated CAPTCHA image.
4. Solve the CAPTCHA using OCR.
5. Populate the login form.
6. Allow the application's JavaScript to encrypt the data.
7. Submit the request.
8. Detect successful authentication.

By interacting with the application as a normal browser would, the client-side encryption mechanism remains intact while repetitive manual interaction is eliminated.

---

# Required Tools

The following Python packages were installed.

```bash
pip3 install \
selenium \
selenium-stealth \
fake-useragent \
Pillow \
pytesseract
```

OCR support:

```bash
sudo apt install tesseract-ocr
```

Google Chrome:

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb

sudo apt install ./google-chrome-stable_current_amd64.deb
```

---

# Automation Script

The browser automation script was executed to repeatedly solve the CAPTCHA and perform authentication.

The script is available in the following repository:

**Automation-Tools-for-Red-Teaming**

```text
https://github.com/naval0505/Automation-Tools-for-Red-Teaming/tree/main/Exploits
```

The automation handled:

- CAPTCHA extraction
- OCR processing
- Browser interaction
- CSRF handling
- Login submission

without requiring manual intervention.

---

# Successful Authentication

After the automation completed successfully, valid credentials were recovered.

```
Username

admin
```

```
Password

tinkerbell
```

The application redirected to the authenticated dashboard, confirming successful login.

---

# Flag

```text
THM{8938aed9fbf43bddb7c0a98f292dca00}
```

---

# Attack Flow

```text
Nmap Scan
      │
      ▼
Service Enumeration
      │
      ▼
Login Portal Analysis
      │
      ▼
Burp Suite Interception
      │
      ▼
Encrypted Authentication Payload
      │
      ▼
Source Code Review
      │
      ▼
CSRF Token Discovery
      │
      ▼
JavaScript Analysis
      │
      ▼
Authentication Workflow
      │
      ▼
CAPTCHA Analysis
      │
      ▼
Browser Automation
      │
      ▼
OCR Recognition
      │
      ▼
Successful Authentication
      │
      ▼
Flag Retrieved
```

---

# Key Findings

| Category | Observation |
|-----------|-------------|
| Operating System | Ubuntu Linux |
| Web Server | Apache 2.4.41 |
| Login Protection | CSRF Token |
| Authentication | Client-side JavaScript Encryption |
| CAPTCHA | Dynamically Generated |
| JavaScript File | `script.js` |
| Authentication Endpoint | `server.php` |
| Success Page | `dashboard.php` |
| Automation Tools | Selenium, Tesseract OCR |
| Credentials | `admin : tinkerbell` |
| Flag | `THM{8938aed9fbf43bddb7c0a98f292dca00}` |

---

# Techniques Demonstrated

- Web Application Enumeration
- JavaScript Source Analysis
- Client-side Authentication Analysis
- CSRF Token Handling
- CAPTCHA Automation
- OCR-Based Recognition
- Browser Automation with Selenium
- Authentication Workflow Analysis

---

# Conclusion

This challenge demonstrates that modern web applications often rely on multiple client-side security mechanisms—including JavaScript-based encryption, CSRF protection, and CAPTCHA validation—to make automated attacks more difficult. While these controls significantly complicate manual testing, they do not eliminate the importance of understanding how the application processes authentication requests.

Rather than attempting to decrypt or tamper with the protected request directly, the authentication workflow was analyzed through the application's JavaScript. This revealed that all login parameters, including the username, password, CSRF token, and CAPTCHA response, were collected and encrypted on the client before being transmitted to the backend. By automating an actual browser with Selenium and solving the CAPTCHA using Tesseract OCR, the normal authentication process could be reproduced without bypassing the application's encryption logic.

This challenge highlights the value of combining source code analysis with browser automation when assessing modern web applications protected by client-side security mechanisms. It also demonstrates how automation can streamline repetitive tasks such as CAPTCHA solving while preserving the application's intended authentication workflow.
