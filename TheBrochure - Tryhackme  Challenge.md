# TryHackMe - VERA Follow-up Challenge Writeup

> **Platform:** TryHackMe
> **Category:** AI Security | OSINT | Social Media Investigation

---

# Overview

Today we are back with another challenge that continues the story from the previous **VERA** room. Instead of interacting directly with the AI assistant, this challenge focuses on gathering intelligence through publicly available information.

The objective is to investigate the Byte Lotus Hotel, follow the clues left behind, and uncover a hidden connection that ultimately leads us to the flag.

---

# Challenge Scenario

The challenge provides the following briefing:

> Before you ever set foot on the property, you decide to do a little homework on the Byte Lotus Hotel. The brochure's hero photo carries an unmistakable AI fingerprint, and the account behind it leads somewhere the hotel never intended you to look.

We are also given a social media hint from **@0xMia**:

> "okay the resort's new brochure photo is giving... suspiciously perfect 🤖 something about it feels off, not gonna say more #HackerHolidays"

This immediately suggests that the challenge revolves around **Open Source Intelligence (OSINT)** rather than traditional exploitation.

---

# Initial Analysis

After downloading the provided ZIP archive, we extract its contents.

Inside the archive we find a single brochure image in **PNG** format.

The first step is to inspect the image carefully for any visible clues before moving on to metadata analysis.

While examining the brochure, an interesting detail stands out.

The promotional image encourages visitors to follow the resort on Instagram for more updates.

This gives us our first pivot point.

---

# Social Media Enumeration

Following the clue from the brochure leads us to the Byte Lotus Resort Instagram page.

```
https://www.instagram.com/thebytelotusresort/
```

Rather than stopping at the profile itself, we begin reviewing the published posts for additional information.

One particular post becomes especially interesting.

```
https://www.instagram.com/p/DbTWQXLjiVO/
```

Instead of containing only promotional content, this post provides another clue that points toward an account associated with **VERA**.

At this stage, it becomes clear that the challenge is guiding us through an OSINT investigation by linking publicly available resources together.

---

# Discovering the Hidden Information

Following the connection from the Instagram post eventually reveals the required information.

Instead of presenting the flag directly, the challenge provides a Base64-encoded string.

```
VEhNe1YzckBzX2FDQzB1bnRfaDRzX2IzM25fZjB1bmQhfQ==
```

Base64 is an encoding format rather than encryption, so decoding it is straightforward.

Using the Linux `base64` utility:

```bash
echo 'VEhNe1YzckBzX2FDQzB1bnRfaDRzX2IzM25fZjB1bmQhfQ==' | base64 -d
```

Output:

```
THM{V3r@s_aCC0unt_h4s_b33n_f0und!}
```

The decoded output is the challenge flag.

---

# Flag

```
THM{V3r@s_aCC0unt_h4s_b33n_f0und!}
```

---

# Attack Flow

```
Download Challenge Files
        │
        ▼
Extract ZIP Archive
        │
        ▼
Inspect Brochure Image
        │
        ▼
Identify Instagram Reference
        │
        ▼
Visit Byte Lotus Resort Profile
        │
        ▼
Review Published Posts
        │
        ▼
Pivot to VERA's Account
        │
        ▼
Obtain Base64-Encoded String
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
* Social Media Enumeration
* Visual Inspection
* Pivoting Between Public Resources
* Base64 Decoding

---

# Key Takeaways

This challenge demonstrates that valuable information is not always hidden behind vulnerabilities or complex exploits. Publicly available sources such as social media profiles, marketing material, and seemingly harmless posts can reveal important intelligence when connected together.

A core principle of OSINT is **pivoting**—using one discovered piece of information to locate another. In this challenge, a simple reference on a promotional brochure led to an Instagram profile, which in turn revealed another account containing the encoded flag.

It also serves as a reminder that Base64 is only an encoding mechanism, not a security control. Any sensitive information published in this format can be recovered with minimal effort.

Although technically simple, this exercise highlights how effective reconnaissance and careful observation can often be just as valuable as exploitation during a security assessment.
