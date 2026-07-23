---
title: "Credential Hunting in Windows"
date: 2026-07-23 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, credential-hunting, lazagne, findstr, post-exploitation]
---

Once we have access to a target Windows machine, it is important to hunt credentials across the file system. Many operating systems have built-in search features, which make it easy to search documents for passwords. The first step is to ask yourself how the machine's user uses the system, in order to focus the search with clear goals.

Some key terms can be used in our search:

- `password`
- `passphrase`
- `key`
- `username`
- `user account`
- `creds`
- `users`
- `passkey`
- `configuration`
- `dbcredential`
- `dbpassword`
- `pwd`
- `login`
- `credentials`

## Manual Credential Hunting

If we have GUI access, it is natural to start the search using Windows' built-in search feature.

![Searching for credential-related files with Windows Search](/assets/img/posts/credential-hunting-windows/windows-search.png)
_Searching for credentials with Windows' built-in search feature_

We can also use `findstr` to search for patterns across many types of files. Keeping the common key terms in mind, we can use variations of this command to discover credentials on a Windows target:

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

## Discovering with LaZagne

[LaZagne](https://github.com/AlessandroZ/LaZagne) is made up of modules, each of which targets different software when looking for passwords. Some of the common modules are described below:

- **browsers**: extracts passwords from various browsers, including Chromium, Firefox, Microsoft Edge, and Opera
- **chats**: extracts passwords from various chat applications, including Skype
- **mails**: searches through mailboxes for passwords, including Outlook and Thunderbird
- **memory**: dumps passwords from memory, targeting KeePass and LSASS
- **sysadmin**: extracts passwords from the configuration files of various sysadmin tools, like OpenVPN and WinSCP
- **windows**: extracts Windows-specific credentials, targeting LSA secrets, Credential Manager, and more
- **wifi**: dumps WiFi credentials

Once LaZagne is installed on the target machine, we can execute the following command to run all included modules:

```cmd
start LaZagne.exe all
```

![Extracting stored credentials with LaZagne](/assets/img/posts/credential-hunting-windows/lazagne-execution.png)
_Extracting credentials with LaZagne_