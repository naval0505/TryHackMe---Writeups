# TryHackMe - Kernel-Level Backdoor Investigation

> **Category:** Linux Forensics | Defensive Security
>
> **Difficulty:** Easy
>
> **Objective:** Investigate a compromised Linux system suspected of running a kernel-level backdoor and recover the hidden flag.

---

# Scenario

A high-value Linux system has been compromised. Security analysts detected suspicious activity originating from the Linux kernel, indicating that the attacker has established persistence at the kernel level. Traditional user-space detection tools failed to identify the compromise, suggesting that the malicious component is operating with elevated privileges.

Our objective is to perform live forensic analysis on the compromised system, identify the malicious kernel module, understand its functionality, and recover the hidden flag.

---

# Investigation Methodology

The investigation follows a structured Linux forensic workflow:

```
System Enumeration
        │
        ▼
Running Processes
        │
        ▼
Scheduled Tasks
        │
        ▼
Kernel Logs
        │
        ▼
Loaded Kernel Modules
        │
        ▼
Kernel Module Analysis
        │
        ▼
Static String Analysis
        │
        ▼
Secret Extraction
        │
        ▼
Flag Recovery
```

---

# Initial Enumeration

As with every Linux forensic investigation, the first step is identifying any suspicious running processes and scheduled tasks that may indicate persistence.

## Running Processes

Command executed:

```bash
ps aux | grep -i root
```

This command lists processes running with elevated privileges.

No immediately suspicious user-space processes were identified, suggesting that the malicious activity may not be occurring within a normal userspace application.

---

## Scheduled Tasks

To identify any scheduled persistence mechanisms, the system crontab was inspected.

```bash
cat /etc/crontab
```

No suspicious cron entries were identified.

Since both running processes and scheduled tasks appeared normal, the investigation shifted towards the Linux kernel.

---

# Kernel Log Analysis

Kernel logs often provide valuable insight into low-level system activity.

The following command was executed:

```bash
cat /var/log/kern.log
```

Relevant entries were observed:

```text
spatch: loading out-of-tree module taints kernel.

spatch: module verification failed:
signature and/or required key missing

[CIPHER BACKDOOR] Module loaded.

Write data to /proc/cipher_bd

[CIPHER BACKDOOR] Executing command: id

[CIPHER BACKDOOR] Command Output:

uid=0(root)
gid=0(root)
groups=0(root)
```

---

## Analysis

These log entries immediately stand out for several reasons.

### Unsigned Kernel Module

The kernel reports:

```text
loading out-of-tree module

module verification failed
```

Out-of-tree modules are kernel modules that are not included with the official Linux kernel.

While legitimate third-party drivers may generate similar messages, unsigned kernel modules should always be treated as suspicious during incident response.

---

### Custom Backdoor Messages

Even more concerning are the log messages:

```text
[CIPHER BACKDOOR]
```

The repeated appearance of this custom logging string strongly suggests that the module was intentionally developed to perform malicious actions.

Additional observations include:

```text
Module loaded

Executing command: id

Command Output:
uid=0(root)
```

The module appears capable of executing commands directly inside kernel context and returning their output.

The presence of:

```text
Write data to

/proc/cipher_bd
```

also indicates that the attacker created a custom **/proc** interface for interacting with the backdoor.

At this stage, the evidence strongly suggests that the compromise involves a malicious kernel module rather than a traditional user-space implant.

---

# Investigating the Kernel Module

Since the logs repeatedly referenced **spatch**, the next step was identifying the loaded kernel module.

Command executed:

```bash
modinfo spatch
```

Output:

```text
filename:
/lib/modules/6.8.0-1016-aws/kernel/drivers/misc/spatch.ko

description:
Cipher is always root

author:
Cipher

license:
GPL

name:
spatch
```

---

## Analysis

Several interesting observations can be made.

The module is located under:

```text
kernel/drivers/misc/
```

making it appear similar to a legitimate miscellaneous kernel driver.

However, the description immediately raises suspicion:

```text
Cipher is always root
```

Legitimate kernel modules rarely contain descriptions of this nature.

Combined with the kernel log messages previously observed, this strongly indicates that **spatch.ko** is the malicious component responsible for the compromise.

---

# Static Analysis of the Kernel Module

To better understand the module without reverse engineering it, string analysis was performed.

Command executed:

```bash
strings /lib/modules/6.8.0-1016-aws/kernel/drivers/misc/spatch.ko | head -n 20
```

Relevant output:

```text
get_flag

cipher_bd

/tmp/cipher_output.txt

/bin/sh

%s > %s 2>&1

HOME=/root

[CIPHER BACKDOOR] Module loaded.

[CIPHER BACKDOOR] Failed to create /proc entry

[CIPHER BACKDOOR] Command Output

[CIPHER BACKDOOR] Failed to read output file
```

---

## Analysis

The extracted strings reveal several important indicators regarding the module's behaviour.

### Custom Proc Interface

```text
cipher_bd
```

This matches the `/proc/cipher_bd` interface observed earlier in the kernel logs, confirming that the module exposes a custom communication channel through the `/proc` filesystem.

---

### Command Execution

The presence of:

```text
/bin/sh
```

together with:

```text
%s > %s 2>&1
```

strongly suggests that the module executes shell commands and redirects both standard output and standard error into an output file.

---

### Temporary Output Storage

The module references:

```text
/tmp/cipher_output.txt
```

This indicates that command output is temporarily written to disk before being returned to the user through the backdoor interface.

---

### Hidden Functionality

One of the most interesting strings recovered is:

```text
get_flag
```

Although its exact implementation was not reverse engineered during this challenge, the function name clearly suggests that the module contains dedicated functionality related to retrieving the hidden flag.

This serves as an important clue for further investigation.

---

# Searching for Hidden Secrets

Since the module already contained several suspicious strings, a more targeted search was performed.

Command executed:

```bash
strings /lib/modules/6.8.0-1016-aws/kernel/drivers/misc/spatch.ko | grep -i secret
```

Output:

```text
[CIPHER BACKDOOR]

Here's the secret:

54484d7b73757033725f736e33346b795f643030727d0a
```

---

## Analysis

Unlike previous strings, this output contains a long hexadecimal sequence.

```text
54484d7b73757033725f736e33346b795f643030727d0a
```

The surrounding message:

```text
Here's the secret
```

strongly suggests that this value represents the challenge flag stored inside the malicious kernel module.

Rather than immediately assuming its contents, the encoded value was decoded using **CyberChef**.

---

# Decoding the Secret

The hexadecimal value was copied into CyberChef using the **From Hex** operation.

Input:

```text
54484d7b73757033725f736e33346b795f643030727d0a
```

Decoded output:

```text
THM{sup3r_sn34ky_d00r}
```

---

# Flag

```text
THM{sup3r_sn34ky_d00r}
```

---

# Investigation Summary

The investigation revealed that the system had been compromised through a malicious kernel module named **spatch.ko**.

Evidence collected during the investigation includes:

- Unsigned out-of-tree kernel module.
- Custom kernel log messages.
- Malicious `/proc/cipher_bd` interface.
- Ability to execute shell commands.
- Temporary output file creation.
- Embedded hidden function (`get_flag`).
- Hardcoded hexadecimal secret within the module.

Static analysis alone was sufficient to recover the embedded flag without requiring reverse engineering of the kernel module.

---

# Key Findings

| Category | Observation |
|-----------|-------------|
| Compromise Type | Kernel-Level Backdoor |
| Malicious Module | `spatch.ko` |
| Author | Cipher |
| Proc Interface | `/proc/cipher_bd` |
| Command Execution | `/bin/sh` |
| Output File | `/tmp/cipher_output.txt` |
| Hidden Function | `get_flag` |
| Secret Storage | Embedded Hexadecimal String |
| Flag Recovery | CyberChef (Hex Decode) |

---

# Conclusion

This challenge demonstrates how attackers can achieve deep persistence by operating at the kernel level rather than within traditional user-space applications. Initial forensic steps such as examining running processes and scheduled tasks did not reveal obvious indicators of compromise, emphasizing that conventional detection techniques may be insufficient against kernel-level threats.

Kernel log analysis proved to be the turning point in the investigation, revealing the loading of an unsigned out-of-tree module named **spatch** along with custom log messages generated by the **Cipher Backdoor**. Further inspection of the module using `modinfo` and `strings` uncovered a custom `/proc` interface, shell execution capabilities, temporary output handling, and a hidden function associated with flag retrieval.

By identifying an embedded hexadecimal string labelled as a secret and decoding it with CyberChef, the hidden flag was successfully recovered without requiring full reverse engineering of the kernel module.

This investigation highlights the importance of analysing kernel logs, inspecting loaded modules, and performing static analysis when investigating suspected kernel-level compromises. It also demonstrates how seemingly small artifacts within binaries can reveal critical evidence during a forensic investigation.


# RAW DATA

Today we have another Forensics based challenge from TryHackMe which is easy listed and the flag we have to look for into the systems.

Scenario : A high-value system has been compromised. Security analysts have detected suspicious activity within the kernel, but the attacker’s presence remains hidden. Traditional detection tools have failed, and the intruder has established deep persistence. Investigate a live system suspected of running a kernel-level backdoor.

We start by investigating the kernel logs.

cat kern.log
2026-07-12T02:30:28.565054+00:00 tryhackme kernel: spatch: loading out-of-tree module taints kernel.
2026-07-12T02:30:28.565075+00:00 tryhackme kernel: spatch: module verification failed: signature and/or required key missing - tainting kernel
2026-07-12T02:30:28.565077+00:00 tryhackme kernel: [CIPHER BACKDOOR] Module loaded. Write data to /proc/cipher_bd
2026-07-12T02:30:28.565078+00:00 tryhackme kernel: [CIPHER BACKDOOR] Executing command: id
2026-07-12T02:30:28.572088+00:00 tryhackme kernel: [CIPHER BACKDOOR] Command Output: uid=0(root) gid=0(root) groups=0(root)
2026-07-12T02:30:28.572106+00:00 tryhackme kernel: 
2026-07-12T02:31:08.537811+00:00 tryhackme kernel: traps: mate-power-mana[1967] trap int3 ip:72a00b6050df sp:7ffe1aabe690 error:0 in libglib-2.0.so.0.8000.0[72a00b5c1000+a0000]
root@tryhackme:/var/log# 


Using grep kernel allows us to view any syslog outputs that will have some sort of kernel affect to it. Sure enough, we see a module “spatch”, followed by [CIPHER BACKDOOR] in the listing, which certainly raises alarms.


I want to see if this “spatch” is an active module in this machines kernel, so I do the lsmod command, which lists all LKMs (Loadable Kernel Modules) installed onto the machines kernel.


and with the lsmod we came to find about spatch and then we looked for spatch based file into the root directory


sudo modinfo spatch
filename:       /lib/modules/6.8.0-1016-aws/kernel/drivers/misc/spatch.ko
description:    Cipher is always root
author:         Cipher
license:        GPL
srcversion:     81BE8A2753A1D8A9F28E91E
depends:        
retpoline:      Y
name:           spatch
vermagic:       6.8.0-1016-aws SMP mod_unload

and with strings spatch.ko| more

y: echo "%s" > /proc/cipher_bd
6[CIPHER BACKDOOR] Here's the secret: 54484d7b73757033725f736e33346b
795f643030727d0a

and decoding this hex we get THM{sup3r_sn34ky_d00r}
