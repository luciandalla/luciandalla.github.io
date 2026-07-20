---
title: "Dumping LSASS Credentials on Windows"
date: 2026-07-20 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, lsass, mimikatz, pypykatz, hashcat, post-exploitation]
---

**LSASS** (Local Security Authority Subsystem Service) is a core Windows process responsible for enforcing security policies, handling user authentication, and storing sensitive credential material in memory.

Dumping LSASS is similar to dumping the SAM database: first, we create a file with the extracted memory dump, and after transferring it to the attack machine, we can crack the credentials offline.

Below are two techniques for dumping LSASS.

## Method 1: Task Manager

Open Task Manager, select the **Processes** tab, find and right-click on **Local Security Authority Process**, then select **Create dump file**. A file called `lsass.DMP` is created and saved in `%temp%`.

![Creating an LSASS dump file via Task Manager](/assets/img/posts/dumping-lsass-credentials/task-manager-dump.png){: .normal }
_Creating a dump file through Task Manager_

## Method 2: Rundll32.exe & Comsvcs.dll

This method is more flexible because it doesn't require a GUI. However, some antivirus solutions may flag it as malicious. The first step is to discover the `lsass.exe` PID:

```cmd
tasklist /svc
```

or, on PowerShell:

```powershell
Get-Process lsass
```

With an elevated PowerShell session, we can issue the following command to create the dump file:

```powershell
PS C:\Windows\system32> rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

With this command, we run `rundll32.exe` to call an exported function of `comsvcs.dll`, which in turn calls the `MiniDumpWriteDump` (`MiniDump`) function to dump the LSASS process memory to a specified location (`C:\lsass.dmp`).

## Extracting Credentials with Pypykatz

On the attack host, we can use **Pypykatz**, a powerful tool that can extract credentials from the `.dmp` file. Pypykatz is a Python implementation of Mimikatz (which only runs on Windows).

```bash
pypykatz lsa minidump <file.dmp>
```

![Extracting credentials from the dump file with pypykatz](/assets/img/posts/dumping-lsass-credentials/pypykatz-execution.png){: .normal }
_Running pypykatz against the LSASS dump file_

The tool can extract data from different authentication protocols, such as:

- **MSV**: used by newer Windows versions
- **WDIGEST**: used by older Windows versions, often storing credentials
  in plaintext
- **Kerberos**
- **DPAPI** (masterkey)

## Cracking the Hash

After extracting the hash, we can use hashcat to crack it.

![hashcat cracking the extracted hash](/assets/img/posts/dumping-lsass-credentials/hashcat-cracked.png){: .normal }
_Password recovered after cracking the extracted hash with hashcat_