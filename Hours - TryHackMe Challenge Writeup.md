# THM — After Hours Attachment Forensics

## Flag

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

## Overview

The challenge provides a password-protected ZIP archive containing artifacts from a Windows system.

The password provided for the archive is:

```text
Aft3rH0ursAtt4chm3ntP4ss
```

The objective is to investigate the artifacts, identify suspicious persistence mechanisms, and recover the flag in the format:

```text
THM{}
```

The important discovery was a malicious **WMI persistence mechanism** stored inside the Windows WMI repository.

---

# 1. Identify the Evidence File

First, identify the downloaded archive:

```bash
ls -lah
```

Example:

```text
attachments-1784136288483.zip
```

Check the file type:

```bash
file attachments-1784136288483.zip
```

Expected result:

```text
Zip archive data
```

---

# 2. Inspect the ZIP Contents

Before extracting the archive, list its contents:

```bash
unzip -l attachments-1784136288483.zip
```

If the archive is password protected, extract it using the supplied password.

Using `7z`:

```bash
7z x attachments-1784136288483.zip -p'Aft3rH0ursAtt4chm3ntP4ss'
```

Or using `unzip`:

```bash
unzip -P 'Aft3rH0ursAtt4chm3ntP4ss' attachments-1784136288483.zip
```

> For forensic work, `7z` is generally preferable because it supports a wide range of archive formats.

---

# 3. Examine the Extracted Files

List the extracted directory:

```bash
find . -type f -ls
```

The important artifacts are associated with the Windows WMI repository.

Among the files we encounter are:

```text
OBJECTS.DATA
INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
```

These are characteristic of a Windows WMI repository.

---

# 4. Recognizing the WMI Repository

Windows Management Instrumentation (WMI) stores information about classes, providers, event filters, consumers, and other management objects.

A particularly important file is:

```text
OBJECTS.DATA
```

This can contain serialized WMI objects.

The repository is interesting from a forensic perspective because attackers can abuse WMI for **persistence**.

For example, an attacker can create:

```text
__EventFilter
        |
        v
__FilterToConsumerBinding
        |
        v
CommandLineEventConsumer
```

This causes a command or program to execute when a particular WMI event occurs.

---

# 5. Quick String Analysis

Before using specialized forensic tooling, perform a simple string search.

```bash
strings OBJECTS.DATA | less
```

Search for common suspicious keywords:

```bash
strings OBJECTS.DATA | grep -i "powershell"
```

```bash
strings OBJECTS.DATA | grep -i "cmd"
```

```bash
strings OBJECTS.DATA | grep -i "consumer"
```

```bash
strings OBJECTS.DATA | grep -i "eventfilter"
```

Also search for commands associated with account creation:

```bash
strings OBJECTS.DATA | grep -Ei "net user|whoami|schtasks|powershell|cmd.exe"
```

The `net user` keyword is particularly interesting.

---

# 6. Search for the Suspicious Command

During analysis, a suspicious command appears:

```text
/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add
```

This immediately stands out because it attempts to create a Windows user account.

The command follows this structure:

```text
net user <username> <password> /add
```

In this case:

```text
Username:
patch
```

and the second argument is a Base64-looking string:

```text
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

---

# 7. Decode the Base64 String

Use the Linux `base64` utility:

```bash
echo 'VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9' | base64 -d
```

The result is:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

Therefore:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

is the flag.

---

# 8. Python Base64 Method

The same operation can be performed with Python:

```python
import base64

data = "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9"

decoded = base64.b64decode(data)

print(decoded.decode())
```

Output:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 9. PowerShell Decoding Method

Since the evidence originates from a Windows environment, PowerShell can also decode the value.

```powershell
[System.Text.Encoding]::UTF8.GetString(
    [System.Convert]::FromBase64String(
        "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9"
    )
)
```

Output:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 10. Investigating the WMI Persistence

The presence of:

```text
CommandLineEventConsumer
```

is important.

A WMI event consumer can execute commands when its associated event filter is triggered.

The typical WMI persistence relationship looks like:

```text
+-------------------+
|   __EventFilter   |
+---------+---------+
          |
          | Binding
          v
+-----------------------------+
| __FilterToConsumerBinding   |
+-------------+---------------+
              |
              v
+-----------------------------+
| CommandLineEventConsumer    |
+-------------+---------------+
              |
              v
        Malicious Command
```

This is why simply looking for normal startup locations such as:

```text
Run
RunOnce
Startup
Scheduled Tasks
Services
```

may not reveal the persistence.

---

# 11. Search for PowerShell

Because WMI persistence is frequently combined with PowerShell, search the repository for PowerShell:

```bash
strings OBJECTS.DATA | grep -i "powershell"
```

Also search for encoded PowerShell:

```bash
strings OBJECTS.DATA | grep -Ei "powershell.*-enc|powershell.*encodedcommand"
```

If a long Base64-looking PowerShell command is discovered, copy it separately for decoding.

---

# 12. Search for Common WMI Persistence Classes

Additional useful searches include:

```bash
strings OBJECTS.DATA | grep -i "__EventFilter"
```

```bash
strings OBJECTS.DATA | grep -i "__FilterToConsumerBinding"
```

```bash
strings OBJECTS.DATA | grep -i "CommandLineEventConsumer"
```

You can combine them:

```bash
strings OBJECTS.DATA | grep -Ei "__EventFilter|__FilterToConsumerBinding|CommandLineEventConsumer"
```

These searches help establish that the suspicious command is not simply random data inside the repository.

---

# 13. Searching for the Flag Directly

Another useful forensic technique is to search for the expected flag prefix:

```bash
strings OBJECTS.DATA | grep "THM{"
```

If the flag is encoded rather than stored directly, this will not necessarily return anything.

Therefore, searching for suspicious encoded strings is important.

For example:

```bash
strings OBJECTS.DATA | grep -E '[A-Za-z0-9+/]{20,}={0,2}'
```

This can produce Base64-like candidates.

Each candidate should then be validated rather than blindly decoded.

---

# 14. Using `grep` Against Extracted Strings

Create a strings dump:

```bash
strings OBJECTS.DATA > objects_strings.txt
```

Then search it:

```bash
grep -in "powershell" objects_strings.txt
```

```bash
grep -in "net user" objects_strings.txt
```

```bash
grep -in "consumer" objects_strings.txt
```

```bash
grep -in "event" objects_strings.txt
```

This makes the investigation easier because you can preserve the extracted strings as a separate forensic artifact.

---

# 15. Hex-Level Investigation

If normal strings analysis is insufficient, inspect the binary directly.

Using `xxd`:

```bash
xxd OBJECTS.DATA | less
```

Or:

```bash
hexdump -C OBJECTS.DATA | less
```

Search for ASCII strings:

```bash
grep -a -i "net user" OBJECTS.DATA
```

The `-a` option tells `grep` to treat the binary file as text.

You can also search for:

```bash
grep -a -i "powershell" OBJECTS.DATA
```

```bash
grep -a -i "CommandLineEventConsumer" OBJECTS.DATA
```

---

# 16. Dedicated WMI Repository Analysis

For deeper analysis, a dedicated WMI repository parser can be used rather than relying solely on `strings`.

The objective is to reconstruct the WMI objects and their properties.

The investigation should focus on objects/classes such as:

```text
__EventFilter
__EventConsumer
CommandLineEventConsumer
__FilterToConsumerBinding
```

This approach is preferable when the repository contains large amounts of unrelated binary data.

The workflow becomes:

```text
OBJECTS.DATA
     |
     v
WMI repository parser
     |
     v
WMI classes/objects
     |
     +---- __EventFilter
     |
     +---- __FilterToConsumerBinding
     |
     +---- CommandLineEventConsumer
                         |
                         v
                    Command
                         |
                         v
                  Encoded value
                         |
                         v
                    Base64
                         |
                         v
                       Flag
```

---

# 17. Why the `net user` Command Matters

The discovered command was:

```text
net user patch <password> /add
```

This is a Windows account creation command.

Breaking it down:

```text
net
```

Windows networking/account administration utility.

```text
user
```

Selects local user account management.

```text
patch
```

The username being created.

```text
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

The password supplied to the new account.

```text
/add
```

Creates the account.

The Base64 value therefore deserves immediate investigation.

---

# 18. Base64 Is Encoding, Not Encryption

The value:

```text
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

looks suspicious because it contains only characters commonly found in Base64.

Base64 is an encoding mechanism, not encryption.

Therefore it can be reversed without a key.

```bash
echo 'VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9' | base64 -d
```

Result:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 19. Alternative CyberChef Method

The same value can be decoded with CyberChef.

Input:

```text
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

Select:

```text
From Base64
```

The output is:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

For offline forensic investigations, however, the command-line or Python method is preferable because it does not require uploading potentially sensitive evidence to an external service.

---

# 20. One-Liner Investigation

Once the archive has been extracted, the basic investigation can be reduced to:

```bash
strings OBJECTS.DATA | grep -Ei "net user|powershell|CommandLineEventConsumer|__EventFilter"
```

Then decode the discovered Base64 value:

```bash
echo 'VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9' | base64 -d
```

Result:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 21. Full Investigation Workflow

The complete investigation can be represented as:

```text
Password-Protected ZIP
        |
        v
Extract using password
        |
        v
Identify WMI repository
        |
        v
OBJECTS.DATA
        |
        v
strings / grep
        |
        v
Identify suspicious WMI objects
        |
        v
CommandLineEventConsumer
        |
        v
Suspicious command
        |
        v
net user patch <Base64> /add
        |
        v
Extract Base64 value
        |
        v
Base64 decode
        |
        v
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 22. Indicators of Compromise Found

During the investigation, the following indicators are useful:

### WMI persistence

```text
__EventFilter
__FilterToConsumerBinding
CommandLineEventConsumer
```

### Suspicious command

```text
net user patch <encoded-value> /add
```

### Encoded data

```text
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

### Created username

```text
patch
```

### Final flag

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 23. Key Lessons

This challenge demonstrates several important Windows forensic concepts:

1. **Do not limit investigations to normal files.**

   Important persistence mechanisms can exist inside Windows management repositories.

2. **WMI can be abused for persistence.**

   Event filters and event consumers can execute commands based on system events.

3. **`OBJECTS.DATA` can contain valuable forensic evidence.**

4. **`strings` is useful for initial triage.**

5. **Base64 is not encryption.**

   Encoded values can often be recovered immediately.

6. **Suspicious account creation commands deserve investigation.**

   Commands such as:

   ```text
   net user <username> <password> /add
   ```

   can indicate attacker activity.

7. **Follow the execution chain.**

   Instead of stopping after finding a suspicious string:

   ```text
   WMI object
       ↓
   Consumer
       ↓
   Command
       ↓
   Encoded data
       ↓
   Decode
       ↓
   Flag
   ```

---

# Final Flag

```text
THM{P4tch_op3ned_th3_BacKd00r}
```
