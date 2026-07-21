---
title: "Dumping SAM Hashes and LSA Secrets on Windows"
date: 2026-07-19 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, registry, sam, dpapi, post-exploitation, hashcat, netexec]
---

With administrative access to a Windows system, we can quickly dump the files associated with the SAM database, transfer them to our attack host, and begin cracking the hashes offline.

## Registry Hives

- **HKLM\SAM**: contains the password hashes for local user accounts
- **HKLM\SYSTEM**: stores the system boot key, which is used to encrypt the SAM database
- **HKLM\SECURITY**: contains sensitive information used by the Local Security Authority (LSA)

## Extracting the Hives Locally

These hives can be copied using `reg.exe`. This requires **SYSTEM** privileges.

```powershell
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

![Extracting the SAM, SYSTEM and SECURITY hives with reg.exe](/assets/img/posts/dumping-sam-hashes-windows/reg-save.png)
_Extracting the registry hives locally with reg.exe_

## Cracking the Hashes

After moving the files to the attack machine, we can use **Impacket's secretsdump** to dump the hashes:

```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py \
  -sam sam.save -security security.save -system system.save LOCAL
```

It's important to always extract `HKLM\SYSTEM` together with `HKLM\SAM`, because `HKLM\SYSTEM` holds the boot key, which is required to decrypt `HKLM\SAM`.

![Output of secretsdump.py showing the dumped hashes](/assets/img/posts/dumping-sam-hashes-windows/secretsdump-output.png)
_Dumping the hashes offline with secretsdump_

Pay attention to the output line `Dumping local SAM hashes (uid:rid:lmhash:nthash)`, which specifies the hash type. We can save the hashes to a file and use hashcat to crack them:

```bash
sudo hashcat -m 1000 hashesfile.txt /usr/share/wordlists/rockyou.txt
```

![hashcat cracking the extracted NT hashes](/assets/img/posts/dumping-sam-hashes-windows/hashcat-cracked.png)
_Cracking the extracted hashes with hashcat_

## HKLM\SECURITY: Cached Domain Credentials and DPAPI

`HKLM\SECURITY` contains cached domain logon information, specifically in the form of **DCC2** hashes. This hash type, identified as mode `2100` in hashcat, is much slower to crack than NT hashes — it's really difficult to crack within a typical pentest timeframe.

`HKLM\SECURITY` also contains machine and user keys for **DPAPI** (Data Protection Application Programming Interface). It is used to protect data such as autocomplete passwords from Chrome and Internet Explorer, email account passwords, credentials for remote machine connections, and credentials for accessing shared resources, wireless networks, and VPNs.

We can use `mimikatz.exe` to decrypt this data manually. For example:

```
dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" unprotect
```

## Remote Dumping

With access to credentials that have local administrator privileges, it is possible to dump LSA secrets and the SAM remotely, for example with the following commands:

```bash
netexec smb <ip> --local-auth -u <user> -p <password> --lsa
netexec smb <ip> --local-auth -u <user> -p <password> --sam
```

![netexec extracting LSA secrets remotely](/assets/img/posts/dumping-sam-hashes-windows/netexec-lsa-sam.png)
_Remote extraction of LSA secrets with netexec_

