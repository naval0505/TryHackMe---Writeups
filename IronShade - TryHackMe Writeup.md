# IronShade - TryHackMe Walkthrough

## Challenge Information

| Category       | Value                                                                                                                       |
| -------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Platform       | TryHackMe                                                                                                                   |
| Challenge Name | IronShade                                                                                                                   |
| Type           | Defensive Security / Linux Forensics                                                                                        |
| Objective      | Investigate a compromised Linux server and identify attacker activity, persistence mechanisms, and indicators of compromise |

---

# Scenario

Based on threat intelligence reports, an APT group known as **IronShade** has been actively targeting Linux servers across the region.

To better understand the attack methodology used by this threat actor, a vulnerable Linux honeypot was intentionally exposed to the internet with weak SSH configurations.

The server was eventually compromised, and our objective is to perform a full compromise assessment and identify:

* Backdoor accounts
* Persistence mechanisms
* Suspicious services
* Malicious processes
* SSH activity
* Installed malware
* Indicators of compromise

---

# Question 1

## What is the Machine ID of the machine being investigated?

Using:

```bash
hostnamectl
```

Output:

```text
Machine ID: dc7c8ac5c09a4bbfaf3d09d399f10d96
```

### Answer

```text
dc7c8ac5c09a4bbfaf3d09d399f10d96
```

---

# Question 2

## What backdoor user account was created on the server?

While reviewing user directories inside:

```text
/home
```

a suspicious account was discovered:

```text
mircoservice
```

The username closely resembles the legitimate word:

```text
microservice
```

which is a common attacker technique known as typosquatting.

### Answer

```text
mircoservice
```

---

# Question 3

## What cronjob was configured for persistence?

Reviewing root's cron jobs:

```bash
sudo crontab -l
```

reveals:

```text
@reboot /home/mircoservice/printer_app
```

This ensures execution whenever the system reboots.

### Answer

```text
@reboot /home/mircoservice/printer_app
```

### Analysis

This is a classic persistence mechanism.

The attacker configured a malicious binary to execute automatically every time the server starts.

---

# Question 4

## What suspicious hidden process was running from the backdoor account directory?

Checking running processes:

```bash
ps aux | grep -i mirco
```

Output:

```text
/home/mircoservice/.tmp/.strokes
```

### Answer

```text
.strokes
```

### Analysis

The process is hidden inside a hidden directory and disguised with a hidden filename.

Such naming conventions are frequently used by malware to avoid detection.

---

# Question 5

## How many processes were found running from the backdoor account directory?

Process review reveals:

```text
/home/mircoservice/.tmp/.strokes
/home/mircoservice/printer_app
```

Total:

```text
2
```

### Answer

```text
2
```

---

# Question 6

## What is the name of the hidden file found in the root directory?

Listing hidden files:

```bash
ls -lah /
```

reveals:

```text
.systmd
```

### Answer

```text
.systmd
```

### Analysis

The filename closely resembles:

```text
systemd
```

which is likely intended to blend in with legitimate Linux system files.

---

# Question 7

## What suspicious services were installed on the server?

Investigating system services:

```bash
ls /etc/systemd/system
```

reveals suspicious entries:

```text
backup.service
strokes.service
```

Sorting alphabetically as required:

### Answer

```text
backup.service, strokes.service
```

### Analysis

Both services appear attacker-created and are unrelated to normal Ubuntu installations.

---

# Question 8

## When was the backdoor account created?

Reviewing authentication logs:

```bash
cat /var/log/auth.log
```

reveals account creation activity.

Timestamp:

```text
Aug 5 22:05:33
```

### Answer

```text
Aug 5 22:05:33
```

---

# Question 9

## From which IP address were multiple SSH connections observed?

Searching SSH logs:

```bash
strings /var/log/auth.log | grep -i ssh
```

reveals:

```text
Invalid user microservice from 10.11.75.247
```

### Answer

```text
10.11.75.247
```

### Analysis

This IP repeatedly appears throughout authentication activity and is associated with the attacker.

---

# Question 10

## How many failed SSH login attempts were observed on the backdoor account?

Using:

```bash
lastb | grep -i mirco
```

Output shows:

```text
8
```

failed login attempts.

### Answer

```text
8
```

---

# Question 11

## Which malicious package was installed?

Reviewing package installation logs:

```bash
grep " install " /var/log/dpkg.log
```

reveals:

```text
install pscanner:amd64 1.5
```

### Answer

```text
pscanner
```

### Analysis

This package was installed shortly after compromise and stands out from normal operating system packages.

---

# Question 12

## What secret code was found within the package metadata?

Checking package information:

```bash
dpkg -l | grep pscanner
```

Output:

```text
pscanner 1.5 amd64 Secret_code{_tRy_Hack_ME_}
```

### Answer

```text
Secret_code{_tRy_Hack_ME_}
```

---

# Indicators of Compromise (IOCs)

## Malicious User

```text
mircoservice
```

## Suspicious Processes

```text
/home/mircoservice/.tmp/.strokes
/home/mircoservice/printer_app
```

## Suspicious Files

```text
.systmd
```

## Persistence

```text
@reboot /home/mircoservice/printer_app
```

## Malicious Services

```text
backup.service
strokes.service
```

## Attacker IP

```text
10.11.75.247
```

## Malicious Package

```text
pscanner
```

---

# Timeline of Attack

```text
Attacker Gains Access
          │
          ▼
Backdoor Account Created
(mircoservice)
          │
          ▼
Persistence Established
(cronjob)
          │
          ▼
Malicious Services Installed
          │
          ▼
Hidden Malware Executed
(.strokes)
          │
          ▼
Additional Binary Deployed
(printer_app)
          │
          ▼
pscanner Installed
          │
          ▼
Repeated SSH Activity
from 10.11.75.247
```

---

# Final Answers

| Question              | Answer                                 |
| --------------------- | -------------------------------------- |
| Machine ID            | dc7c8ac5c09a4bbfaf3d09d399f10d96       |
| Backdoor User         | mircoservice                           |
| Cronjob               | @reboot /home/mircoservice/printer_app |
| Hidden Process        | .strokes                               |
| Processes Running     | 2                                      |
| Hidden Root File      | .systmd                                |
| Suspicious Services   | backup.service, strokes.service        |
| Account Creation Time | Aug 5 22:05:33                         |
| Attacker IP           | 10.11.75.247                           |
| Failed SSH Attempts   | 8                                      |
| Malicious Package     | pscanner                               |
| Secret Code           | Secret_code{*tRy_Hack_ME*}             |

---

# Key Takeaways

* Attackers frequently create look-alike user accounts to blend into production environments.
* Hidden files and hidden directories remain common persistence locations on Linux systems.
* Cron jobs are frequently abused for long-term persistence.
* Systemd services should always be reviewed during compromise assessments.
* Authentication logs provide valuable evidence regarding attacker activity.
* Package installation logs can reveal malware deployment timelines.
* Typosquatting account names remain an effective evasion technique used by threat actors.

Challenge completed successfully.
