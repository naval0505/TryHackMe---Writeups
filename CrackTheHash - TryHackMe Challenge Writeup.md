# TryHackMe - Crack the Hash

> **Category:** Defensive Security | Cryptography
>
> **Difficulty:** Easy → Medium
>
> **Objective:** Identify various cryptographic hash algorithms, determine the correct cracking approach, recover plaintext passwords using online databases and password cracking techniques, and understand when different tools are appropriate.

---

# Scenario

This TryHackMe room focuses on one of the most common tasks performed during penetration testing, digital forensics, and incident response: **password hash cracking**.

Throughout the challenge, multiple hashes generated using different algorithms are presented. The objective is to:

- Identify the hashing algorithm.
- Select the appropriate cracking method.
- Recover the original plaintext password.

The room gradually increases in difficulty by introducing stronger hashing algorithms, salted hashes, and password hashes commonly found on Linux systems.

---

# Investigation Methodology

```
Unknown Hash
      │
      ▼
Hash Identification
      │
      ▼
Determine Hash Algorithm
      │
      ▼
Choose Cracking Method
      │
      ├── Online Database
      ├── CrackStation
      ├── Hashes.com
      └── Hashcat
      │
      ▼
Recover Plaintext Password
      │
      ▼
Verify Result
```

---

# Level 1 - Basic Hash Identification

The first stage focuses on recognising common hashing algorithms and recovering passwords using publicly available hash databases.

---

# Question 1

Hash:

```text
48bb6e862e54f2a795ffc4e541caed4d
```

---

## Hash Identification

The hash is 32 hexadecimal characters long, strongly suggesting an **MD5** hash.

Rather than immediately attempting manual cracking, the hash was searched using **CrackStation**.

Recovered password:

```text
easy
```

---

## Analysis

MD5 is considered cryptographically broken and should never be used for password storage.

Because billions of MD5 hashes already exist in public rainbow tables, weak passwords can often be recovered instantly.

---

# Question 2

Hash:

```text
CBFDAC6008F9CAB4083784CBD1874F76618D2A97
```

---

## Hash Identification

The hash length indicates **SHA-1**.

After confirming the format, the hash was searched using CrackStation.

Recovered password:

```text
password123
```

---

## Analysis

Although SHA-1 is stronger than MD5 from a collision perspective, it is still unsuitable for password storage because it is:

- Fast
- Unsalted
- Widely indexed online

This allows weak passwords to be recovered almost instantly.

---

# Question 3

Hash:

```text
1C8BFE8F801D79745C4631D09FFF36C82AA37FC4CCE4FC946683D7B336B63032
```

---

## Hash Identification

The first step was identifying the hashing algorithm.

Command:

```bash
hashid 1C8BFE8F801D79745C4631D09FFF36C82AA37FC4CCE4FC946683D7B336B63032
```

Possible matches:

```text
SHA-256

Snefru-256

RIPEMD-256

SHA3-256

Haval-256

GOST

Skein
```

Based on the length and challenge context, the correct algorithm is **SHA-256**.

Searching the hash in CrackStation revealed:

```text
letmein
```

---

## Analysis

SHA-256 itself remains cryptographically secure.

However, when passwords are hashed **without salting**, weak passwords remain vulnerable because they can still be indexed in public cracking databases.

---

# Question 4

Hash:

```text
$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom
```

---

## Hash Identification

Hash identification:

```bash
hashid '$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom'
```

Results:

```text
bcrypt

Blowfish (OpenBSD)

Woltlab Burning Board
```

The prefix:

```text
$2y$
```

identifies the hash as **bcrypt**.

Recovered password:

```text
bleh
```

The hash was successfully resolved using **Hashes.com**.

---

## Analysis

Unlike MD5 or SHA-1, bcrypt is specifically designed for password storage.

It incorporates:

- Salt
- Configurable work factor
- Slow hashing

These features make brute-force attacks significantly more difficult compared to traditional cryptographic hashes.

---

# Question 5

Hash:

```text
279412f945939ba78ce0758d3fd83daa
```

---

## Hash Identification

Command:

```bash
hashid 279412f945939ba78ce0758d3fd83daa
```

Possible matches:

```text
MD5

MD4

MD2

Double MD5
```

The hash was cracked using CrackStation.

Recovered password:

```text
Eternity22
```

---

## Analysis

As with the first challenge, this demonstrates how weak passwords protected only with MD5 can often be recovered immediately using publicly available databases.

---

# Level 1 Summary

Recovered passwords:

| Hash Algorithm | Password |
|----------------|----------|
| MD5 | easy |
| SHA-1 | password123 |
| SHA-256 | letmein |
| bcrypt | bleh |
| MD5 | Eternity22 |

---

# Level 2 - Advanced Hash Cracking

The second stage introduces stronger password hashes.

Unlike the previous tasks, the challenge notes that all passwords originate from the **RockYou** wordlist.

This means dictionary attacks become a practical approach.

Tools such as **Hashcat** are particularly useful for this type of attack.

---

# Question 1

Hash:

```text
F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85
```

---

## Hash Identification

Running HashID:

```bash
hashid F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85
```

Possible algorithms:

```text
SHA-256

SHA3-256

RIPEMD-256

Snefru-256
```

The hash was searched using Hashes.com.

Recovered password:

```text
paule
```

---

## Analysis

Even though SHA-256 is cryptographically secure, weak passwords remain vulnerable if no salt is used.

Password complexity remains equally important as algorithm selection.

---

# Question 2

Hash:

```text
1DFECA0C002AE40B8619ECF94819CC1B
```

---

## Hash Identification

The hash length suggested either MD5 or NTLM.

After querying CrackStation, the recovered password was:

```text
n63umy8lkf4i
```

The challenge identifies this as an **NTLM** hash.

---

## Analysis

NTLM remains widely encountered during Windows security assessments.

Because NTLM uses unsalted password hashes, dictionary attacks remain highly effective against weak passwords.

---

# Question 3

Hash:

```text
$6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02.
```

Salt:

```text
aReallyHardSalt
```

---

## Hash Identification

HashID output:

```text
SHA-512 Crypt
```

The prefix:

```text
$6$
```

identifies the hash as **SHA-512 Crypt**, which is commonly used on Linux systems.

Recovered password:

```text
waka99
```

---

## Analysis

Unlike standard SHA-512, **SHA-512 Crypt** incorporates:

- Salt
- Multiple hashing rounds

This significantly increases resistance against precomputed rainbow table attacks.

Nevertheless, weak passwords remain vulnerable to dictionary attacks.

---

# Question 4

Hash:

```text
e5d8870e5bdd26602cab8dbe07a942c8669e56d6
```

Salt:

```text
tryhackme
```

---

## Hash Identification

Running HashID:

```bash
hashid e5d8870e5bdd26602cab8dbe07a942c8669e56d6
```

Possible algorithms:

```text
SHA-1

Double SHA-1

RIPEMD-160

Tiger-160
```

Recovered password:

```text
481616481616
```

Salt:

```text
tryhackme
```

---

## Analysis

This challenge demonstrates a **salted SHA-1** hash.

The salt forces attackers to compute new hashes for every candidate password, preventing the direct use of rainbow tables.

However, dictionary attacks remain successful when users choose weak passwords.

---

# Password Recovery Summary

| Question | Hash Type | Password |
|-----------|-----------|----------|
| Level 1 - Q1 | MD5 | easy |
| Level 1 - Q2 | SHA-1 | password123 |
| Level 1 - Q3 | SHA-256 | letmein |
| Level 1 - Q4 | bcrypt | bleh |
| Level 1 - Q5 | MD5 | Eternity22 |
| Level 2 - Q1 | SHA-256 | paule |
| Level 2 - Q2 | NTLM | n63umy8lkf4i |
| Level 2 - Q3 | SHA-512 Crypt | waka99 |
| Level 2 - Q4 | Salted SHA-1 | 481616481616 |

---

# Tools Used

During the investigation, several tools were used depending on the type of hash.

| Tool | Purpose |
|------|---------|
| HashID | Hash identification |
| CrackStation | Online password recovery |
| Hashes.com | Online cracking database |
| Hashcat | Dictionary-based offline cracking |
| CyberChef | Data decoding and transformations |

---

# Key Learning Points

- Always identify the hash before attempting to crack it.
- Hash length and prefixes provide valuable clues.
- Unsalted hashes are significantly easier to crack.
- Salting prevents the use of precomputed rainbow tables.
- bcrypt and SHA-512 Crypt are designed specifically for password storage.
- Weak passwords remain vulnerable regardless of the hashing algorithm.
- Dictionary attacks are often more efficient than brute-force attacks when common password lists such as RockYou are applicable.

---

# Conclusion

This challenge provides a practical introduction to password hash identification and cracking techniques commonly encountered during defensive security investigations and penetration testing. By analyzing the format of each hash and selecting appropriate tools such as HashID, CrackStation, Hashes.com, and Hashcat, multiple hashing algorithms—including MD5, SHA-1, SHA-256, bcrypt, NTLM, and SHA-512 Crypt—were successfully identified and cracked.

The exercises demonstrate that the security of stored passwords depends not only on the strength of the hashing algorithm but also on proper implementation. Fast, unsalted algorithms such as MD5 and SHA-1 are highly susceptible to rainbow table and dictionary attacks, whereas modern password hashing algorithms like bcrypt and SHA-512 Crypt provide stronger protection through salting and computational cost. Nevertheless, even robust algorithms cannot compensate for weak passwords, highlighting the continued importance of strong password policies and secure password storage practices.
