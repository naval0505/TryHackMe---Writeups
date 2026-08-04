# TryHackMe - Ponzi Writeup

> **Platform:** TryHackMe
> **Challenge:** Ponzi
> **Category:** Web Application Security | Business Logic Vulnerability | Race Condition

---

# Overview

Today we are back with another **TryHackMe** challenge focused on **Web Application Pentesting**. Unlike traditional web exploitation rooms that revolve around SQL Injection or Remote Code Execution, this challenge introduces a **Business Logic Vulnerability**.

The application is a cryptocurrency reward platform where users can claim a daily reward once every 24 hours. At first glance, everything appears to be functioning correctly—the application enforces a cooldown timer and prevents users from claiming rewards repeatedly. However, deeper inspection reveals that the application's server-side logic fails to properly handle multiple requests arriving simultaneously.

By abusing a **Race Condition**, we can bypass the daily reward restriction, dramatically increase our balance, unlock the Whale Vault, and retrieve the final flag.

This room demonstrates why secure application logic is just as important as traditional input validation.

---

# Challenge Scenario

The room begins with the following scenario.

> Ponzi found the resort's wellness portal running a little side project called **Ponzi** — a crypto rewards app, poolside edition.

> He set his towel down, claimed his daily reward, and went to reapply sunscreen.

> He came back to find the sunbed had been "claimed" three times over while he wasn't looking.

> He's convinced the app owes him a spot in the Whale Vault.

> The app disagrees, politely, once every 24 hours.

> Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through.

We are also provided with the application.

```text id="94bzyb"
http://10.49.180.201:3000
```

The objective is straightforward.

* Create an account.
* Investigate the reward mechanism.
* Identify the application's logical weakness.
* Access the Whale Vault.
* Recover the flag.

---

# Understanding the Application

After registering a new guest account and logging in, the application presents a dashboard displaying our current cryptocurrency balance.

A button is available for claiming a **daily reward**.

After claiming the reward once, the application immediately starts a **24-hour countdown timer**, preventing additional claims.

At first glance, everything appears to be properly secured.

However, the room description provides an important hint.

> Somewhere between his request and the server's clock...

This strongly suggests that the vulnerability is not related to authentication or authorization but rather **how the application processes concurrent requests.**

---

# Intercepting the Request

Burp Suite is launched to inspect the request responsible for claiming rewards.

Intercepting the request reveals the following.

```http id="f2m1kr"
POST /claim HTTP/1.1

Host: 10.49.180.201:3000

Cookie: connect.sid=...
```

Interestingly, the request contains almost no user-controlled parameters.

The request body is completely empty.

```text id="z67fj8"
Content-Length: 0
```

This indicates that all reward validation occurs on the server.

Rather than manipulating request parameters, the focus shifts toward **how the server handles multiple requests simultaneously.**

---

# Identifying the Logic Bug

The application appears to perform two separate operations.

1. Check whether the user is eligible for today's reward.
2. Credit the reward and update the claim timestamp.

If these two operations are not executed atomically, multiple requests arriving at nearly the same time may all pass the eligibility check before the timestamp is updated.

This is a classic **Race Condition**.

Instead of exploiting user input, we exploit timing.

---

# Testing for Race Conditions

The intercepted request is sent to **Burp Repeater**.

Using Burp's **Parallel Request Group**, the same request is duplicated multiple times.

Approximately twenty identical requests are placed into the same parallel execution group.

All requests are then sent simultaneously.

Instead of receiving only a single successful response, several requests are accepted.

Returning to the dashboard reveals that the account balance has increased significantly.

This confirms that the server is processing multiple reward claims before updating the user's cooldown.

The race condition is successfully reproduced.

---

# Exploiting the Vulnerability

With the vulnerability confirmed, the attack is repeated on a larger scale.

Rather than sending twenty requests, one hundred identical requests are executed simultaneously.

Every request targets the same endpoint.

```http id="dzhjmm"
POST /claim
```

Because the application fails to synchronize concurrent reward processing, many of the requests succeed.

The balance rapidly increases.

Eventually the dashboard displays approximately:

```text id="8qgfwl"
850 Ponzi Coins
```

This far exceeds the intended daily reward limit.

The application's business logic has been completely bypassed.

---

# Accessing the Whale Vault

Once the required balance threshold is reached, the Whale Vault becomes accessible.

Opening the vault reveals the challenge flag.

```text id="wdm6hf"
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

The challenge is now complete.

---

# Attack Flow

```text id="7ktl7p"
Create Guest Account
        │
        ▼
Login to Dashboard
        │
        ▼
Inspect Daily Reward
        │
        ▼
Intercept POST /claim
        │
        ▼
Analyze Server Logic
        │
        ▼
Identify Race Condition
        │
        ▼
Send Parallel Requests
        │
        ▼
Bypass Reward Cooldown
        │
        ▼
Increase Account Balance
        │
        ▼
Unlock Whale Vault
        │
        ▼
Capture Flag
```

---

# Understanding Race Conditions

A **Race Condition** occurs when multiple requests interact with the same resource at nearly the same time and the application fails to synchronize those operations correctly.

In secure implementations, the server should:

1. Verify eligibility.
2. Immediately lock the account.
3. Update the database.
4. Credit the reward.

Only after completing those steps should another request be processed.

In this application, multiple requests perform the eligibility check simultaneously before any of them update the cooldown timer.

As a result, every request believes the reward is still available.

This allows users to receive the reward multiple times despite the intended "once every 24 hours" restriction.

---

# Vulnerability Identified

**Business Logic Vulnerability**

The application trusts that only one reward request will arrive at a time.

There is no synchronization or locking mechanism protecting the reward transaction.

This enables attackers to exploit concurrent execution and receive multiple rewards.

**Impact**

* Daily reward restriction bypassed.
* Unlimited balance generation.
* Unauthorized access to premium functionality.
* Complete compromise of the application's reward economy.

---

# Mitigation Recommendations

Several measures can prevent this vulnerability.

* Perform reward validation and reward updates inside a single database transaction.
* Lock the user's reward record during processing.
* Reject duplicate reward requests while another request is being processed.
* Use atomic update operations instead of separate validation and write operations.
* Implement server-side concurrency controls rather than relying on client-side timers.

---

# Techniques Used

* Web Application Enumeration
* Burp Suite Repeater
* Parallel Request Groups
* Race Condition Testing
* Business Logic Analysis
* Web Application Exploitation

---

# Flag

```text id="18ov3q"
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

---

# Key Takeaways

This challenge highlights that not every web vulnerability originates from unsanitized input or insecure code execution. Many real-world compromises occur because applications fail to enforce their own business rules consistently under concurrent conditions. Here, the application correctly enforced a 24-hour reward limit for individual requests, but failed to account for multiple requests arriving simultaneously.

Race Conditions are particularly dangerous because they often bypass otherwise well-designed validation logic. Instead of injecting malicious input, the attacker simply exploits timing to force the application into an inconsistent state. Proper transaction handling, record locking, and atomic database operations are essential defenses against this class of vulnerability.

Overall, this room provides an excellent introduction to business logic flaws and demonstrates why modern web penetration testing extends beyond traditional vulnerabilities into understanding how applications behave under real-world conditions.
