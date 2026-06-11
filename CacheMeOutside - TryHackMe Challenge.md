# Cache Me Outside - TryHackMe Walkthrough

## Challenge Information

| Category       | Value                                                                                |
| -------------- | ------------------------------------------------------------------------------------ |
| Platform       | TryHackMe                                                                            |
| Challenge Name | Cache Me Outside                                                                     |
| Type           | Defensive Security / OSINT                                                           |
| Objective      | Perform reverse OSINT to identify a retired hacker and trace their digital footprint |

---

# Scenario

Years after leaving the hacking scene, a retired hacker has unintentionally left traces of his identity scattered across the public internet.

The investigation begins with what appears to be a simple conversation screenshot. However, hidden inside the image is the first clue leading to multiple online profiles and publicly exposed information.

Our task is to:

* Identify the retired hacker
* Trace his online presence
* Correlate publicly available information
* Discover his contact details
* Determine his final location

---

# Initial Investigation

The challenge begins with a screenshot.

Careful inspection of the image reveals a public profile URL:

```text
https://www.komoot.com/user/5667624959835
```

This becomes our starting point.

---

# Komoot Profile Analysis

Opening the Komoot profile reveals information associated with the account owner.

Further investigation eventually leads to another platform.

A linked GitHub profile is identified:

```text
https://github.com/jiml33t
```

The username:

```text
jiml33t
```

appears to be used consistently across platforms.

At this stage we have our primary OSINT target.

---

# GitHub Enumeration

The next step is reviewing the GitHub profile for exposed information.

Public repositories and commits often contain:

* Usernames
* Emails
* Metadata
* Historical account details

One useful technique is reviewing commit patches directly.

GitHub allows commit data to be viewed in patch format.

Example:

```text
https://github.com/jiml33t/jiml33t/commit/7b2c8e0a540c36f2e09da5945066020621d6a059.patch
```

Reviewing the patch reveals author metadata.

---

# Email Discovery

Within the commit information, the email address becomes visible.

Recovered email:

```text
jimleepro1@gmail.com
```

This provides a verified contact point associated with the target.

---

# Phone Number Enumeration

The next challenge requires identifying the target's phone number.

Using the recovered email address, a message is sent.

An automated response is returned.

Contents:

```text
Good day,

I will be absent from the office while I prepare for a marathon.
You can contact my on my phone for anything urgent.

Best Regards,

JL
0x4A4C

JIM LEE

CYBERSECURITY CONSULTANT

jimleepro1@gmail.com

L33T SECURITY
PENTESTING · RED TEAM · CONSULTING
```

The autoresponder provides additional confirmation that:

```text
Jim Lee
```

is indeed the account owner.

The challenge answer can then be extracted from the contact details contained within the response.

---

# Social Media Investigation

To continue the investigation, attention shifts to the target's social media activity.

A Threads profile is discovered:

```text
https://www.threads.com/@jiml33t/post/DYCg8B1iMAl/media
```

The post contains an image.

At first glance, the image appears unrelated.

However, geolocation analysis often relies on:

* Architecture
* Street signs
* Public transport
* Road markings
* Language
* Urban infrastructure

---

# Image Geolocation

Analyzing the image reveals several geographical indicators.

Using image analysis and geolocation techniques, the location is identified as:

```text
Timișoara
Romania
```

This significantly narrows the search area.

---

# Tram Station Identification

The challenge requires identifying the specific tram station visible in the image.

After comparing landmarks and transportation infrastructure within the city, the exact station is identified.

Recovered location:

```text
Piața Gheorghe Domășneanu
```

This represents the final location required to complete the investigation.

---

# Investigation Path

```text
Conversation Screenshot
          │
          ▼
Komoot Profile
          │
          ▼
GitHub Profile
          │
          ▼
Commit Patch Analysis
          │
          ▼
Email Discovery
          │
          ▼
Autoresponder Investigation
          │
          ▼
Phone Number Recovery
          │
          ▼
Threads Profile
          │
          ▼
Image Analysis
          │
          ▼
Timișoara, Romania
          │
          ▼
Piața Gheorghe Domășneanu
```

---

# Key OSINT Techniques Used

## Profile Pivoting

Moving from one platform to another using:

* Shared usernames
* Linked accounts
* Public references

---

## GitHub Metadata Analysis

Examining:

```text
Commit History
Patch Files
Author Metadata
```

to recover exposed email addresses.

---

## Email Intelligence

Using known email addresses to:

* Verify ownership
* Trigger autoresponders
* Gather additional contact information

---

## Social Media Analysis

Reviewing public posts to identify:

* Travel locations
* Habits
* Interests
* Physical whereabouts

---

## Geolocation

Using image-based OSINT techniques to identify:

* Cities
* Public infrastructure
* Transportation hubs
* Precise locations

---

# Final Answers

| Question          | Answer                                              |
| ----------------- | --------------------------------------------------- |
| GitHub Username   | jiml33t                                             |
| Email Address     | [jimleepro1@gmail.com](mailto:jimleepro1@gmail.com) |
| Identified Person | Jim Lee                                             |
| City              | Timișoara                                           |
| Country           | Romania                                             |
| Tram Station      | Piața Gheorghe Domășneanu                           |

---

# Key Takeaways

* Small pieces of publicly available information can often be chained together to reveal an individual's identity.
* GitHub commit metadata remains one of the most common sources of accidental information disclosure.
* Public social media profiles frequently expose location data.
* Automated email responses can reveal valuable OSINT information.
* Username reuse across platforms significantly simplifies attribution efforts.
* Effective OSINT investigations rely heavily on correlation rather than a single source of information.

Challenge completed successfully.
