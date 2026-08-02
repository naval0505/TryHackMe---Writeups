# TryHackMe - Overheard at Breakfast Writeup

> **Platform:** TryHackMe
> **Challenge:** Overheard at Breakfast
> **Category:** OSINT | Digital Footprinting | Gravatar | Email Hash Analysis

---

# Overview

Today we are back with another **OSINT-based** challenge from **TryHackMe** named **Overheard at Breakfast**.

Unlike traditional penetration testing challenges that involve exploiting services or gaining shell access, this room focuses entirely on **Open Source Intelligence (OSINT)**. The objective is to analyze a screenshot containing a private conversation, identify subtle clues left behind by the participants, and pivot through publicly available information to uncover an account that was never intended to be discovered.

This challenge demonstrates an important lesson in OSINT investigations: seemingly insignificant pieces of information—such as profile pictures, usernames, or hashed email addresses—can often reveal much more than intended when correlated across multiple online services.

---

# Challenge Scenario

The room begins with the following scenario.

> The breakfast terrace is loud this morning, clinking cutlery, espresso machines, the usual chatter. One guest couldn't help but linger at a nearby table, seeing more of a conversation than they were meant to.

> When the table's occupant stepped away for a refill, they seized the moment and grabbed a screenshot before it could disappear. Somewhere in that conversation is enough to track down an account nobody was supposed to find.

From the description, it is immediately clear that no active exploitation will be required.

Instead, the challenge revolves around careful observation, extracting useful information from the provided screenshot, and following OSINT pivot points.

---

# Examining the Challenge Files

The provided challenge consists of a ZIP archive.

Verifying the downloaded file:

```bash
file overheard-at-breakfast-1784259780309.zip
```

Output:

```text
Zip archive data
```

After extracting the archive, the contents reveal a single image.

The screenshot becomes the primary source of intelligence throughout the investigation.

---

# Initial Image Analysis

The first step in any OSINT challenge involving images is to inspect every visible element before jumping into specialized tools.

While reviewing the screenshot, several interesting observations can immediately be made.

The conversation appears to take place between two known participants.

```text
Mia (@Lambo)

Ponzi
```

These names have appeared in previous Byte Lotus challenges, making them useful pivot points.

Rather than focusing solely on the chat itself, attention should also be given to profile pictures, usernames, timestamps, and any identifiers that may reveal external services.

At this stage there is no obvious flag or hidden message visible inside the screenshot.

This suggests that another form of correlation is required.

---

# Identifying the Pivot Point

One of the profile images appears familiar.

Many online platforms—including WordPress, GitHub, Stack Overflow, Atlassian, and numerous blogging platforms—retrieve profile pictures from **Gravatar**.

Gravatar generates globally recognized avatars based on a user's email address.

Historically, Gravatar profile images have been linked using an **MD5 hash** of the user's email address.

This creates an interesting OSINT opportunity.

If the screenshot contains a Gravatar image, it may be possible to recover publicly available profile information associated with that account.

Instead of attempting to brute-force the email manually, specialized Gravatar lookup tools can be used.

---

# Alternative Investigation Methods

At this stage there are multiple possible approaches.

One option is using **GHunt**, which can assist in identifying Google account information from publicly available artifacts.

However, GHunt requires additional configuration and authentication before use.

For this challenge, using dedicated Gravatar lookup services provides a much faster approach.

This demonstrates an important concept during OSINT investigations:

> There is often more than one valid methodology. Choosing the most efficient path saves significant time.

---

# Using Gravatar Lookup Tools

Several public tools are available for querying Gravatar profiles.

One example is:

```text
http://toolator.com/gravatar-checker
```

Another commonly used resource is:

```text
https://gravatar.com/site/check
```

These tools allow investigators to search for Gravatar-related information using email hashes or profile identifiers.

After providing the required information obtained from the screenshot, the lookup successfully resolves the associated Gravatar profile.

Instead of only returning profile details, the page displays an additional message.

---

# Recovering the Hidden Information

The discovered page returns the following text.

```text
Funny thing about email hashes, they follow you places you didn't expect.

Glad you found the right corner of the internet!

Here is your prize:

VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9
```

The value clearly resembles **Base64** encoding.

Rather than representing encrypted information, Base64 simply encodes data into ASCII characters.

Decoding the value is straightforward.

Using the Linux `base64` utility:

```bash
echo "VEhNe1MzY3JlVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50MWZpM2R9" | base64 -d
```

Output:

```text
THM{S3cr3T_Pr0fil3_H4s_b33n_Id3nt1fi3d}
```

The decoded value is the final challenge flag.

---

# Flag

```text
THM{S3cr3T_Pr0fil3_H4s_b33n_Id3nt1fi3d}
```

---

# Attack Flow

```text
Download Challenge Files
        │
        ▼
Extract ZIP Archive
        │
        ▼
Inspect Screenshot
        │
        ▼
Identify Conversation Participants
        │
        ▼
Recognize Gravatar Profile
        │
        ▼
Research Gravatar Investigation Methods
        │
        ▼
Use Gravatar Lookup Tool
        │
        ▼
Locate Hidden Profile
        │
        ▼
Recover Base64 String
        │
        ▼
Decode Base64
        │
        ▼
Capture Flag
```

---

# Techniques Used

* Open Source Intelligence (OSINT)
* Image Analysis
* Digital Footprinting
* Profile Correlation
* Gravatar Enumeration
* Email Hash Investigation
* Base64 Decoding

---

# Key Takeaways

This challenge demonstrates how information shared across multiple online platforms can unintentionally expose an individual's digital footprint. Services such as Gravatar rely on identifiers derived from email addresses, allowing profile information to persist across websites even when users believe they have removed or hidden it.

A key lesson from this room is the importance of **pivoting** during OSINT investigations. Rather than attempting to guess hidden information directly, investigators identify one reliable artifact—in this case, a Gravatar profile—and use it to uncover additional publicly accessible information. Small clues, when combined across multiple sources, often reveal far more than intended.

Finally, the challenge reinforces that encoding mechanisms such as Base64 provide no security by themselves. Whenever encoded strings are encountered during an investigation, they should always be analyzed and decoded before assuming they contain meaningless data.

---

# Conclusion

Overheard at Breakfast is a beginner-friendly OSINT challenge that emphasizes observation, digital footprint analysis, and effective pivoting between publicly available resources. Instead of exploiting software vulnerabilities, the room rewards careful examination of available evidence and demonstrates how profile correlation can uncover hidden accounts. It serves as a practical introduction to Gravatar-based investigations and highlights how seemingly harmless online identifiers can expose far more information than users expect.
