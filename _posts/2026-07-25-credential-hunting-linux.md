---
title: "Credential Hunting in Linux"
date: 2026-07-25 10:00:00 -0300
categories: [Concepts, Linux]
tags: [linux, credential-dumping, credential-hunting, mimipenguin, lazagne, firefox-decrypt, post-exploitation]
---

Credential hunting is one of the first post-exploitation steps, as exposed credentials can quickly lead to privilege escalation. Valuable credentials may be found in configuration files, databases, scripts, logs, command history, memory, or browser keyrings. A thorough understanding of the target system's role and environment helps prioritize where to search and increases the chances of uncovering useful credentials.

## Files

### Configuration Files

The first step in system enumeration is to identify configuration files, as they provide valuable insights into the system's setup and potential security weaknesses. Narrowing the search to common configuration file extensions makes the enumeration process more efficient and focused.

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done
```

### Databases

We can apply the same search approach to other file extensions, such as those used by databases, and then inspect the matching files for sensitive information.

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

### Notes

Searching for notes can reveal valuable information, including credentials, system details, and documentation of internal processes. Since notes may have arbitrary names or no file extension, it's important to search for both `.txt` files and extensionless files during enumeration.

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

### Scripts

Scripts often contain sensitive information, including hardcoded credentials used to automate tasks and system processes. Examining scripts can reveal passwords or other secrets that may be leveraged for privilege escalation or lateral movement.

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

### Cronjobs

Cron jobs automate the execution of commands and scripts, sometimes exposing hardcoded credentials required for scheduled tasks. Reviewing system-wide and user-specific cron configurations can reveal sensitive information and potential privilege escalation opportunities.

```bash
cat /etc/crontab
```

### History Files

History files provide valuable insight into previous user activity and system operations, often revealing commands, credentials, or administrative actions. Reviewing files such as `.bash_history`, `.bashrc`, and `.bash_profile` can uncover useful information for privilege escalation and system enumeration.

```bash
tail -n5 /home/*/.bash*
```

### Log Files

Log files record system activity, authentication events, service behavior, and application errors, making them a valuable source of information during enumeration. Reviewing logs can reveal failed login attempts, scheduled tasks, service configurations, and other indicators useful for privilege escalation. Understanding the purpose and format of common log files helps identify relevant information more efficiently during security assessments.

```bash
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null;fi;done
```

## Memory and Cache

### MimiPenguin

Credentials used by applications and authenticated users may be stored in memory or browser keyrings, making them valuable targets during post-exploitation. Tools such as [mimipenguin](https://github.com/huntergregal/mimipenguin) can extract these credentials from Linux systems, although they require root privileges to operate.

```bash
sudo python3 mimipenguin.py
```

### LaZagne

[LaZagne](https://github.com/AlessandroZ/LaZagne) is a powerful credential extraction tool capable of recovering passwords and hashes from numerous sources, including browsers, SSH, Wi-Fi, cloud services, keyrings, and configuration files. It supports a wide range of Linux credential stores, making it an effective tool for post-exploitation and credential hunting.

```bash
sudo python2.7 laZagne.py all
```

### Firefox Decrypt

Firefox stores saved credentials in encrypted files such as logins.json, but these passwords can still be recovered if an attacker gains access to the user's profile. [Firefox Decrypt](https://github.com/unode/firefox_decrypt) is a specialized tool that extracts and decrypts these stored credentials, making it valuable for credential hunting during post-exploitation.

```bash
python3.9 firefox_decrypt.py
```