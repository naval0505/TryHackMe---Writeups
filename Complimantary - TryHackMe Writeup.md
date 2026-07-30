# TryHackMe - Complimentary Writeup

> **Platform:** TryHackMe
> **Challenge:** Complimentary
> **Category:** Cloud Security | AWS | Cognito | IAM Misconfiguration | DynamoDB

---

# Overview

Today we are solving another cloud-focused challenge from TryHackMe named **Complimentary**.

Unlike traditional web exploitation rooms, this challenge focuses on **AWS cloud security** and demonstrates how misconfigured cloud permissions can expose sensitive data without requiring a user account.

The application advertises itself as requiring **no registration** or **login**, yet it somehow personalizes information for every visitor. The objective is to understand how the application identifies users and determine whether the backend exposes more information than intended.

---

# Challenge Scenario

The challenge begins with a concierge briefing describing the **Byte Lotus Wellness** application.

The application promises instant access without requiring users to create an account, but still manages to recognize them and display personalized information.

This immediately raises an important security question:

> If there is no authentication, what AWS service is deciding what an anonymous user is allowed to access?

---

# Initial Enumeration

The first instinct is to inspect the application's S3 bucket.

Attempting to list the bucket anonymously:

```bash
aws s3 ls s3://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com \
--no-sign-request
```

returns:

```text
NoSuchBucket
```

This indicates that either the bucket name is incorrect or the application isn't relying on public S3 access.

Rather than continuing to brute-force storage resources, the focus shifts toward the AWS credentials provided during the challenge.

---

# Configuring AWS CLI

The provided temporary credentials are configured into a dedicated AWS CLI profile.

```bash
aws configure --profile byte-lotus
```

The following information is supplied:

* AWS Access Key
* Secret Access Key
* Session Token
* Region: us-east-1

Since temporary credentials are being used, the next step is verifying exactly who we are authenticated as.

---

# Identity Enumeration

Using AWS STS:

```bash
aws sts get-caller-identity \
--profile byte-lotus
```

The response reveals:

```text
Assumed Role

complimentary-cognito-unauth-role
```

This is the most important finding in the challenge.

Rather than authenticating as a real application user, the credentials belong to an **unauthenticated Amazon Cognito Identity Pool role**.

This means the application allows anonymous visitors to receive temporary AWS credentials.

Normally these roles should have extremely limited permissions.

---

# Exploring Available Permissions

With valid AWS credentials available, the next step is determining what resources this role can access.

One of the first services tested is DynamoDB.

Running:

```bash
aws dynamodb scan \
--table-name complimentary-GuestWellnessProfiles \
--profile byte-lotus \
--region us-east-1
```

returns data successfully.

This immediately confirms that anonymous users possess permission to read the DynamoDB table.

---

# Sensitive Information Disclosure

Scanning the table reveals multiple guest records.

One of the retrieved entries contains highly sensitive information, including:

```text
Password:
digitaldetox2026

Location:
25.2055,55.2733

Notes:
Booked the quiet room for his "digital detox."
Checked email twice since writing that.
```

This demonstrates that the database stores far more information than should ever be accessible to anonymous users.

Rather than exposing only public profile data, the application leaks:

* User passwords
* Personal notes
* Guest locations
* Internal profile information

---

# Why This Happened

The root cause of the vulnerability is an **overly permissive IAM policy** attached to the unauthenticated Cognito role.

The intended authentication flow was:

```text
Visitor

↓

Temporary Cognito Identity

↓

Restricted AWS Permissions
```

Instead, the permissions effectively became:

```text
Anonymous Visitor

↓

Temporary AWS Credentials

↓

Full DynamoDB Read Access
```

Because DynamoDB permissions were granted directly to unauthenticated identities, anyone capable of obtaining temporary credentials could enumerate the database without creating an account.

---

# Attack Flow

```text
Application Analysis
        │
        ▼
Identify AWS Backend
        │
        ▼
Configure Temporary Credentials
        │
        ▼
Verify Identity with STS
        │
        ▼
Discover Unauthenticated Cognito Role
        │
        ▼
Enumerate AWS Services
        │
        ▼
Access DynamoDB
        │
        ▼
Read Guest Profiles
        │
        ▼
Recover Sensitive Information
```

---

# Vulnerabilities Identified

* Overly permissive Amazon Cognito unauthenticated role
* Excessive IAM permissions
* Unrestricted DynamoDB table access
* Exposure of plaintext sensitive information
* Broken access control in cloud infrastructure
* Insecure handling of guest data

---

# Key Takeaways

This room highlights one of the most common cloud security mistakes: granting excessive permissions to **unauthenticated identities**. Amazon Cognito is designed to provide temporary AWS credentials to anonymous users when required, but those credentials must follow the principle of least privilege.

Because the associated IAM role was allowed to perform unrestricted DynamoDB scans, anonymous users could retrieve sensitive customer information without ever authenticating. The exposed data included passwords, locations, and private notes, illustrating the impact of misconfigured cloud permissions.

The challenge reinforces the importance of carefully reviewing IAM policies, restricting access to cloud resources, and ensuring that temporary credentials are limited to only the minimum permissions necessary for application functionality.
