---
title: "Attacking Active Directory: Password Attacks and NTDS.dit"
date: 2026-07-22 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, active-directory, ntds, credential-dumping, password-attacks, post-exploitation, netexec, kerbrute]
---

Understanding Active Directory (AD) and how to attack it is an essential skill for any penetration tester. Many organizations use Windows across their infrastructure and rely on Active Directory to manage users, computers, and permissions.

## Dictionary Attack Against Active Directory

Before launching an attack, it is important to understand the target organization's structure. Performing a dictionary attack without knowing any valid usernames can generate unnecessary noise.

The first step is to research the company and identify its username pattern. This can often be done by searching for employee accounts associated with the company's domain. After collecting a list of employee names, we can use [Username Anarchy](https://github.com/urbanadventurer/username-anarchy) to generate common username variations based on those names:

```bash
./username-anarchy -i names.txt
```

With a list of potential usernames, we can use [Kerbrute](https://github.com/ropnop/kerbrute) to enumerate valid Active Directory users:

```bash
./kerbrute_linux_amd64 userenum --dc <IP> --domain <DOMAIN> <WORDLIST>
```

![Enumerating valid Active Directory users with Kerbrute](/assets/img/posts/attacking-ad-ntds/kerbrute-userenum.png)
_Enumerating valid domain users with Kerbrute_

Once we have a list of valid users, we can perform a password dictionary
attack against Active Directory using NetExec:

```bash
netexec smb <IP> -u <USER> -p <WORDLIST>
```

![NetExec performing a password dictionary attack](/assets/img/posts/attacking-ad-ntds/netexec-password-attack.png)
_Cracking a valid password with NetExec_

Be aware that some organizations enforce account lockout policies after a certain number of failed login attempts. However, this is not enabled by default in Active Directory.

## Capturing NTDS.dit

**NT Directory Services (NTDS)** is the directory service used by Active Directory to organize and manage network resources. The `NTDS.dit` file is the most important database in the domain — it stores domain usernames, password hashes, and other critical directory information. Capturing this file allows us to extract password hashes and crack them offline.

With a valid username and password, we can connect to the target using Evil-WinRM, which provides a remote PowerShell session:

```bash
evil-winrm -i <IP> -u <USER> -p <PASSWORD>
```

Once connected, the first step is to verify the user's privileges using `net localgroup` and `net user <username>`. Administrative or Domain Admin privileges are required to copy the `NTDS.dit` file.

We could create a Volume Shadow Copy (VSS) to copy the file, but the fastest way to dump the `NTDS.dit` database is by executing NetExec from the attack machine:

```bash
netexec smb <IP> -u <USER> -p <PASSWORD> -M ntdsutil
```

![Dumping NTDS.dit remotely with NetExec](/assets/img/posts/attacking-ad-ntds/netexec-ntds-dump.png)
_Dumping the NTDS.dit database remotely with NetExec_

Congratulations! You now have the domain password hashes, which can be cracked offline using tools such as Hashcat or John the Ripper.