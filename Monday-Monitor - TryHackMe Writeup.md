# Monday Monitor - TryHackMe Walkthrough

## Lab Information

| Category   | Value                                                                  |
| ---------- | ---------------------------------------------------------------------- |
| Platform   | TryHackMe                                                              |
| Lab Name   | Monday Monitor                                                         |
| Type       | Blue Team / SOC Investigation                                          |
| Tools Used | Wazuh, Sysmon                                                          |
| Objective  | Investigate suspicious activity and answer incident response questions |

---

# Scenario

Swiftspend Finance, a fintech company, has recently deployed Wazuh and Sysmon to strengthen endpoint monitoring and improve threat detection capabilities.

The company's Senior Security Engineer, John Sterling, conducted a series of security tests to evaluate the effectiveness of the monitoring infrastructure.

The tests were performed on:

```text
April 29, 2024
Between 12:00:00 and 20:00:00
```

Our objective is to investigate the generated telemetry within the Wazuh dashboard, identify malicious activity, and answer the incident response questions.

---

# Initial Access

## Accessing Wazuh

Login credentials provided:

```text
Username: admin
Password: Mond*yM0nit0r7
```

After authentication, navigate to:

```text
Security Events
```

Since the scenario specifies the timeframe, configure the dashboard time filter accordingly.

```text
Date: April 29, 2024
Time Range: 12:00 - 20:00
```

This significantly reduces noise and focuses the investigation on the relevant attack window.

---

# Question 1

## Initial access was established using a downloaded file. What is the file name saved on the host?

Since phishing documents frequently leverage macros and PowerShell to establish access, a good starting point is searching for:

```text
powershell.exe
```

Reviewing process creation events reveals the downloaded file responsible for the initial compromise.

### Answer

```text
SwiftSpend_Financial_Expenses.xlsm
```

### Analysis

The extension:

```text
.xlsm
```

indicates a macro-enabled Microsoft Excel document.

Macro-enabled Office documents remain a common initial access technique used in phishing campaigns.

---

# Question 2

## What is the full command run to create a scheduled task?

Scheduled tasks are commonly used for persistence.

Search for:

```text
schtasks.exe
```

and review associated process creation events.

The following command is identified:

```text
"cmd.exe" /c "reg add HKCU\\SOFTWARE\\ATOMIC-T1053.005 /v test /t REG_SZ /d cGluZyB3d3cueW91YXJldnVsbmVyYWJsZS50aG0= /f & schtasks.exe /Create /F /TN "ATOMIC-T1053.005" /TR "cmd /c start /min "" powershell.exe -Command IEX([System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String((Get-ItemProperty -Path HKCU:\\SOFTWARE\\ATOMIC-T1053.005).test)))" /sc daily /st 12:34"
```

### Analysis

The command performs multiple actions:

1. Creates a registry key
2. Stores encoded content inside the registry
3. Creates a scheduled task
4. Executes PowerShell through the task
5. Decodes and executes the stored registry value

This demonstrates a persistence mechanism using both Registry Storage and Scheduled Tasks.

---

# Question 3

## What time is the scheduled task meant to run?

Looking closely at the scheduled task configuration:

```text
/st 12:34
```

### Answer

```text
12:34
```

### Analysis

The task is configured to execute daily at 12:34.

---

# Question 4

## What is the encoded string decoding to?

The encoded value found in the registry:

```text
cGluZyB3d3cueW91YXJldnVsbmVyYWJsZS50aG0=
```

Using CyberChef or Base64 decoding:

```bash
echo cGluZyB3d3cueW91YXJldnVsbmVyYWJsZS50aG0= | base64 -d
```

Output:

```text
ping www.youarevulnerable.thm
```

### Answer

```text
ping www.youarevulnerable.thm
```

### Analysis

This appears to be a simple network connectivity test or Atomic Red Team simulation activity.

---

# Question 5

## What password was set for the new user account?

A newly created user account is commonly associated with:

```text
net.exe
```

Searching process creation logs reveals:

```text
"C:\Windows\system32\net.exe" user guest
```

Further review of the command-line parameters identifies the password assigned during account creation.

### Answer

```text
guest
```

### Analysis

Attackers frequently create additional local accounts to maintain persistence after gaining access.

Monitoring account creation events is essential for detecting this activity.

---

# Question 6

## Which file is used to dump credentials?

Credential dumping activity commonly involves tools such as:

* Mimikatz
* Rubeus
* Custom credential dumpers

Reviewing process execution events reveals:

```text
C:\Tools\AtomicRedTeam\atomics\T1003.001\bin\x64\memotech.exe
```

### Answer

```text
memotech.exe
```

### Analysis

The executable is associated with Atomic Red Team credential dumping simulations.

This activity maps to:

```text
MITRE ATT&CK T1003.001
LSASS Memory
```

Credential dumping remains one of the most important indicators of post-exploitation activity.

---

# Question 7

## Data was exfiltrated from the host. What was the flag that was part of the data?

Searching for:

```text
THM{
```

within command-line logs reveals a PowerShell script interacting with Pastebin.

The command contains:

```text
powershell.exe
```

along with:

```text
Invoke-RestMethod
```

used to send data externally.

Relevant content:

```text
secrets, api keys, passwords, THM{M0N1T0R_1$_1N_3FF3CT}, confidential, private
```

### Answer

```text
THM{M0N1T0R_1$_1N_3FF3CT}
```

### Analysis

The PowerShell script sends sensitive information to:

```text
https://pastebin.com/api/api_post.php
```

This represents a simulated data exfiltration event.

The use of:

```text
Invoke-RestMethod
```

is a common PowerShell technique used for communicating with external services.

---

# Investigation Summary

During the investigation, the following attacker activities were identified:

### Initial Access

```text
SwiftSpend_Financial_Expenses.xlsm
```

Macro-enabled Office document used to establish execution.

---

### Persistence

```text
Scheduled Task
Registry Storage
```

Task Name:

```text
ATOMIC-T1053.005
```

Execution Time:

```text
12:34
```

---

### Command Execution

```text
PowerShell
cmd.exe
```

Used to decode and execute payloads stored in the registry.

---

### Account Manipulation

```text
net.exe
```

Used to create a local account.

---

### Credential Access

```text
memotech.exe
```

Credential dumping simulation.

MITRE ATT&CK:

```text
T1003.001
```

---

### Data Exfiltration

```text
PowerShell
Invoke-RestMethod
Pastebin API
```

Sensitive information transferred externally.

Flag Identified:

```text
THM{M0N1T0R_1$_1N_3FF3CT}
```

---

# Final Answers

| Question                     | Answer                                                           |
| ---------------------------- | ---------------------------------------------------------------- |
| Initial access file          | SwiftSpend_Financial_Expenses.xlsm                               |
| Scheduled task command       | schtasks.exe persistence command                                 |
| Scheduled execution time     | 12:34                                                            |
| Decoded Base64 string        | ping [www.youarevulnerable.thm](http://www.youarevulnerable.thm) |
| Password for created account | guest                                                            |
| Credential dumping file      | memotech.exe                                                     |
| Exfiltrated flag             | THM{M0N1T0R_1$_1N_3FF3CT}                                        |

---

# Key Takeaways

* Wazuh and Sysmon provide powerful visibility into endpoint activity.
* Time-based filtering dramatically reduces investigation noise.
* Scheduled Tasks and Registry-based payload storage remain common persistence techniques.
* PowerShell continues to be one of the most abused administrative tools by attackers.
* Credential dumping activity should always be treated as a high-severity event.
* Monitoring outbound HTTP requests can help identify data exfiltration attempts.

Lab completed successfully.
