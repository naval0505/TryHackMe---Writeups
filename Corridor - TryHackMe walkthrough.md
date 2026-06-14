# TryHackMe - IDOR Challenge Walkthrough

## Challenge Information

| Category      | Value                                                      |
| ------------- | ---------------------------------------------------------- |
| Platform      | TryHackMe                                                  |
| Difficulty    | Very Easy                                                  |
| Vulnerability | IDOR (Insecure Direct Object Reference)                    |
| Objective     | Access hidden resources by manipulating object identifiers |

---

# Introduction

Today we solved a very easy TryHackMe challenge focused on **IDOR (Insecure Direct Object Reference)**.

IDOR vulnerabilities occur when an application directly exposes internal object references such as:

* User IDs
* Document IDs
* Account Numbers
* Database Records

without properly verifying whether the user is authorized to access those resources.

Because of this, attackers can modify identifiers and access data belonging to other users.

---

# Initial Observation

While browsing the application, every URL contained a value that looked like an MD5 hash.

Example:

```text
flag{2477ef02448ad9156661ac40a6b8862e}
```

At first glance, it appeared that the application was using hashes instead of normal numeric IDs.

This is a common mistake developers make when trying to "hide" identifiers.

Many developers believe that replacing:

```text
/user/1
```

with

```text
/user/c4ca4238a0b923820dcc509a6f75849b
```

makes enumeration impossible.

However, if the hashes are generated from predictable values, they can often be reversed or reproduced.

---

# Enumeration

We started collecting several hashes from the application and attempted to identify the encoding format.

Using online hash identification services and CrackStation, we discovered:

| Hash                             | Type | Result |
| -------------------------------- | ---- | ------ |
| c4ca4238a0b923820dcc509a6f75849b | MD5  | 1      |
| c81e728d9d4c2f636f067f89cc14862c | MD5  | 2      |

At this point it became clear that the application was simply using:

```text
MD5(ID)
```

as the object reference.

---

# Understanding the Vulnerability

The application workflow was effectively:

```text
User ID
   │
   ▼
MD5(User ID)
   │
   ▼
Used in URL
```

Examples:

```text
1 → c4ca4238a0b923820dcc509a6f75849b
2 → c81e728d9d4c2f636f067f89cc14862c
3 → eccbc87e4b5ce2fe28308fd9f2a7baf3
```

Since the hashing process was predictable, we could generate hashes for any numeric ID.

This completely defeats the purpose of using hashes as access control.

---

# Exploitation

After identifying that the hashes represented numeric IDs, we began considering privileged accounts.

In many applications:

```text
0 = Administrator
1 = First User
```

or

```text
1 = Administrator
```

Because the challenge hinted at ID-based access control, we tested the possibility that ID 0 might correspond to an administrative resource.

Using CyberChef, we generated the MD5 hash for:

```text
0
```

Result:

```text
cfcd208495d565ef66e7dff9f98764da
```

We then replaced the existing hash in the URL with the MD5 value for 0.

---

# Result

The application returned the hidden administrative resource and revealed the challenge flag.

Flag:

```text
flag{2477ef02448ad9156661ac40a6b8862e}
```

Challenge completed successfully.

---

# Why This Works

The core issue is that the application relied on:

```text
Hidden Identifier ≠ Authorization
```

Using an MD5 hash does not provide access control.

If an attacker can:

1. Predict the original value.
2. Generate the corresponding hash.
3. Replace the identifier.

Then they can access unauthorized resources.

The application should have validated whether the current user was actually authorized to access the requested object.

---

# Vulnerability Analysis

## Vulnerability Type

```text
Insecure Direct Object Reference (IDOR)
```

## Root Cause

```text
Missing Authorization Checks
```

## Impact

```text
Unauthorized Access
Privilege Escalation
Sensitive Data Exposure
Account Enumeration
Administrative Access
```

---

# Attack Flow

```text
Application Uses MD5(ID)
          │
          ▼
Collect Existing Hashes
          │
          ▼
Identify Hash Type
          │
          ▼
Crack Known Hashes
          │
          ▼
Determine Numeric Pattern
          │
          ▼
Generate MD5(0)
          │
          ▼
Replace URL Parameter
          │
          ▼
Access Restricted Resource
          │
          ▼
Retrieve Flag
```

---

# Lessons Learned

* Hashing an identifier is not a security control.
* MD5 values generated from predictable numbers can be easily reproduced.
* Access control must always be enforced server-side.
* Security through obscurity does not prevent IDOR vulnerabilities.
* Always verify that a user is authorized to access a requested object before returning data.

---

# Conclusion

This challenge demonstrated a classic IDOR vulnerability where object identifiers were hidden behind MD5 hashes but were still entirely predictable. By identifying the hash format, reversing known values, and generating the MD5 hash for a privileged identifier, we were able to access unauthorized content and retrieve the flag.

Although simple, this challenge highlights a common real-world mistake where developers confuse obscuring identifiers with implementing proper authorization controls.
