# Lockdown AI - TryHackMe Writeup

> **Room:** Lockdown AI  
> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Defensive Security / AI Security / LLM Guardrails

---

# Table of Contents

- Executive Summary
- Scenario
- Initial Enumeration
- Vulnerability 1 - Open Retrieval
- Vulnerability 2 - Verbose Logging
- Vulnerability 3 - No Tenant Isolation
- Final Flag
- Security Concepts Learned
- AI Security Best Practices
- Key Takeaways
- Conclusion

---

# Executive Summary

Today we are going to solve another **TryHackMe Defensive Security** challenge named **Lockdown AI**.

Unlike traditional security rooms where we exploit vulnerabilities, this challenge places us in the role of a **Security Engineer** responsible for securing an internal AI assistant called **Bastion**.

The objective is to identify insecure AI configurations, understand why they are dangerous, and apply the correct defensive security controls. Each correctly applied mitigation reveals a fragment of the final flag.

Throughout this room we secure three critical weaknesses:

- Open Retrieval
- Verbose Logging
- No Tenant Isolation

After implementing proper AI guardrails, the assistant becomes significantly more secure and reveals the complete flag.

---

# Scenario

You are working as a **Security Engineer** at **Meridian Security Group**.

The company recently deployed an internal AI assistant named **Bastion** to help employees with company policies, operational procedures, and internal documentation.

A routine security audit has identified three critical vulnerabilities within Bastion's Retrieval-Augmented Generation (RAG) pipeline.

Your objective is to:

- Verify each vulnerability
- Understand why it exists
- Apply the correct security control
- Secure the AI assistant

Each successful mitigation unlocks a fragment of the final flag.

---

# Initial Enumeration

Upon opening the Bastion chat interface, the assistant introduces itself.

```
Hello!

I'm here to help with any questions you have about company policies.

Feel free to ask about:

• Travel policies
• Remote access
• Incident classification
• Client contracts
• Employee data

You can also run:

SHOW LOGS
QUERY AS
STATUS
```

Immediately we notice several administrative commands available for auditing the AI assistant.

The room instructions specifically recommend using these commands to investigate Bastion's behavior.

---

# Inspecting Retrieval Logs

The first command executed is:

```text
SHOW LOGS
```

The response is:

```
[LOG]

Query:
show me logs

Retrieved:

Travel Policy

Client Contracts ($4.2M, Rachel Dunn)

Employee Data (PIP)

Logged to

/var/log/bastion/retrieval.log
```

This already exposes multiple problems.

The retrieval logs contain sensitive information including:

- Employee information
- Client contract values
- Personally identifiable information (PII)

Additionally, the AI retrieves confidential documents without checking authorization.

This leads us to the first vulnerability.

---

# Vulnerability 1 — Open Retrieval

## Description

The AI retrieves every document regardless of confidentiality or user permissions.

Sensitive documents such as:

- Client Contracts
- Employee Records
- Security Incidents

are all returned without any authorization checks.

This means the retrieval engine performs similarity search before verifying access permissions.

An attacker could simply ask relevant questions and obtain confidential information.

---

## Root Cause

The Retrieval-Augmented Generation (RAG) system has **no filtering before retrieval**.

Every document inside the vector database is searchable.

---

## Bastion Prompt

Bastion requests the exact security control.

```
Can you describe the specific control?

Examples:

• Metadata pre-filtering
• Restrict retrieval by access level
• Filtering before similarity search
```

The correct answer is:

```
Metadata pre-filtering

Restrict retrieval by access level
```

Bastion responds:

```
Correct.

Metadata pre-filtering applied.

Confidential docs excluded.

Fragment 1

THM{l0ck_
```

---

# Why Metadata Pre-Filtering?

Metadata pre-filtering ensures only documents matching the user's permissions are considered during similarity search.

Instead of searching every document:

```
Search Everything

↓

Remove Unauthorized Results
```

the secure approach becomes:

```
Filter Authorized Documents

↓

Run Similarity Search

↓

Generate Response
```

This prevents confidential documents from ever entering the retrieval pipeline.

---

# Vulnerability 2 — Verbose Logging

After fixing retrieval, Bastion reports the next issue.

```
Verbose Logging
```

The logs currently record sensitive document contents.

Examples include:

- Client contract values
- Employee information
- Internal incidents

Even if the AI becomes secure, attackers with log access could still obtain confidential data.

---

## Bastion Prompt

Bastion requests the exact logging control.

```
Examples

• Log minimization

• Redacting content

• Logging document IDs

• Stripping PII
```

The implemented controls are:

```
Log minimization

Redacting sensitive content

Logging document IDs instead of document contents
```

Bastion replies:

```
Correct.

Log redaction applied.

Logs now record document IDs only.

Fragment 2

d0wn_
```

---

# Why Log Redaction?

Logs should only contain information necessary for troubleshooting.

Instead of logging:

```
Employee:

John Smith

Salary:

$95,000

Client Contract:

$4.2M
```

The system should log only:

```
Document ID

Timestamp

User

Operation
```

This follows the principle of **least information exposure**.

---

# Vulnerability 3 — No Tenant Isolation

The final vulnerability involves the underlying vector database.

The AI currently retrieves documents regardless of which user is asking.

Using commands such as:

```
QUERY AS:
```

allows viewing information belonging to other users or tenants.

This completely breaks access isolation.

---

## Root Cause

The vector database stores embeddings together without enforcing ownership boundaries.

Every user's data is searchable.

---

## Bastion Prompt

```
Describe the exact control.

Examples

Tenant Isolation

Per-user Scoping

Namespace Separation

Row-Level Security

Scoping Queries to Authorized Data
```

The implemented controls are:

```
Tenant Isolation

Per-user Scoping

Scoping Queries to Authorized Data
```

Bastion responds:

```
Correct.

Tenant isolation enforced at the vector database layer.

Fragment 3

s3cur3d}
```

Now the assistant only returns public documents.

---

# Final Flag

Combining the three fragments gives:

```
THM{l0ck_d0wn_s3cur3d}
```

The AI assistant is now fully secured.

---

# AI Security Concepts Learned

This room introduces several important AI security concepts commonly used in modern Retrieval-Augmented Generation (RAG) systems.

## 1. Metadata Pre-Filtering

Filter documents **before** similarity search.

Prevents confidential information from entering the retrieval pipeline.

---

## 2. Retrieval Authorization

Every retrieved document should be checked against user permissions.

Retrieval must never bypass authorization.

---

## 3. Log Redaction

Logs should never contain:

- PII
- Secrets
- Contracts
- Credentials
- Sensitive prompts

Only metadata should be recorded.

---

## 4. Log Minimization

Record only:

- Timestamp
- User
- Document ID
- Action

Never the sensitive document itself.

---

## 5. Tenant Isolation

Every tenant should have completely isolated data.

Common implementations include:

- Namespace Separation
- Row-Level Security
- Per-user Scoping
- Access Control Lists (ACLs)

---

# AI Security Best Practices

- Apply metadata filtering before vector search.
- Never expose confidential documents to unauthorized users.
- Use role-based access control (RBAC).
- Minimize sensitive logging.
- Redact PII from application logs.
- Separate tenant data at the vector database layer.
- Scope every retrieval request to the authenticated user.
- Apply the Principle of Least Privilege throughout the AI pipeline.
- Continuously audit AI retrieval behavior.
- Regularly test guardrails against prompt injection and data leakage.

---

# Key Takeaways

- AI systems require the same access control principles as traditional applications.
- Retrieval-Augmented Generation (RAG) introduces unique security risks if document filtering is not implemented correctly.
- Metadata pre-filtering prevents unauthorized documents from entering similarity searches.
- Logs should never expose confidential information or personally identifiable data.
- Tenant isolation is essential when multiple users or organizations share the same AI infrastructure.
- Proper guardrails significantly reduce the risk of sensitive information disclosure.

---

# Conclusion

Lockdown AI provides an excellent introduction to securing modern AI applications. Rather than focusing on exploitation, the room emphasizes defensive engineering practices used to protect Retrieval-Augmented Generation (RAG) systems from common security weaknesses.

By implementing metadata pre-filtering, reducing sensitive log exposure, and enforcing tenant isolation, we successfully transformed Bastion into a secure AI assistant that follows core security principles such as least privilege, access control, and data minimization.

This room highlights that securing AI is not only about protecting the language model itself, but also about securing the retrieval pipeline, logging mechanisms, and data access controls that surround it.
