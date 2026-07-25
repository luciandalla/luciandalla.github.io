---
title: "Linux Authentication Process"
date: 2026-07-24 10:00:00 -0300
categories: [Concepts, Linux]
tags: [linux, pam, credential-dumping, john-the-ripper, hashcat, post-exploitation]
---

Linux-based distributions support various authentication mechanisms. One of the most commonly used is Pluggable Authentication Modules (PAM). This module is typically located in `/usr/lib/x86_64-linux-gnu/security/` and interacts with the `/etc/passwd` and `/etc/shadow` files. The PAM library can also prevent users from reusing old passwords. These previous passwords are stored in the `/etc/security/opasswd` file. Administrator (root) privileges are required to read this file.

## /etc/passwd

The `/etc/passwd` file contains information about every user on the system. The file is readable by all users. Each row in the file follows the structure `[username]:[password]:[userID]:[groupID]:[GECOS]:[home-directory]:[default-shell]`.

![Permissions and content of /etc/passwd file](/assets/img/posts/linux-authentication-process/passwd-file.png)
_Permissions and content of /etc/passwd file_

In rare cases, the file stores the password hashes — this happens only on old systems. Normally, the field is filled with `x`. In modern Linux systems, the password hashes are stored in the `/etc/shadow` file.

## /etc/shadow

The `/etc/shadow` file is responsible for password storage and management. Every user registered in `/etc/passwd` must have a matching entry in this file, or it will be considered invalid. The file structure is `[username]:[password]:[last-change]:[min-age]:[max-age]:[warning-period]:[inactivity-period]:[expiration-date]:[reserved-field]`.

![Example of data in /etc/shadow file](/assets/img/posts/linux-authentication-process/shadow-file.png)
_Example of data in /etc/shadow file_

If the password field contains a character such as `!` or `*`, the user cannot log in using a Unix password. However, other authentication methods can still be used. If the password field is empty, no password is required for login.

The password field also follows a particular format, from which we can extract additional information: `$<id>$<salt>$<hashed>`. The `id` specifies which cryptographic hash algorithm was used:

| ID | Algorithm |
|----|-----------|
| `1` | MD5 |
| `2a` | Blowfish |
| `5` | SHA-256 |
| `6` | SHA-512 |
| `sha1` | SHA1crypt |
| `y` | Yescrypt |
| `gy` | Gost-yescrypt |
| `7` | Scrypt |

## Cracking Linux Credentials

Once we have root access on a Linux machine, we can gather user password hashes and attempt to crack them using various methods to recover the plaintext passwords.

To do this, we can use a tool called `unshadow`, which is included with John the Ripper (JtR). It works by combining the `/etc/passwd` and `/etc/shadow` files into a single file suitable for cracking:

```bash
sudo unshadow /etc/passwd /etc/shadow > /tmp/unshadowed.hashes
```

With the unshadowed file ready, we can crack it using Hashcat or another tool:

```bash
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```