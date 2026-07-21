---
title: "Attacking Windows Credential Manager"
date: 2026-07-21 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, credential-manager, mimikatz, post-exploitation]
---

**Credential Manager** is a feature built into Windows since Windows Server 2008 R2 and Windows 7. It allows users and applications to securely store credentials relevant to other systems and websites.

## Where Credentials Are Stored

Credentials are stored in special encrypted folders on the computer, under the user and system profiles:

- `%UserProfile%\AppData\Local\Microsoft\Vault\`
- `%UserProfile%\AppData\Local\Microsoft\Credentials\`
- `%UserProfile%\AppData\Roaming\Microsoft\Vault\`
- `%ProgramData%\Microsoft\Vault\`
- `%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\`

Microsoft often refers to these protected stores as **Credential Lockers** (formerly Windows Vaults). Credential Manager is the user-facing feature/API, while the actual encrypted stores are the vault/locker folders.

Windows stores two types of credentials:

- **Web Credentials**
- **Windows Credentials**

## Enumerating with cmdkey

We can use `cmdkey` to enumerate the credentials stored in the current user's profile:

```powershell
cmdkey /list
```

![Enumerating stored credentials with cmdkey](/assets/img/posts/attacking-windows-credential-manager/cmdkey-enum.png)
_Enumerating credentials with cmdkey_

The second group of information in the image, `Domain:interactive=SRV01\mcharles`, is a domain credential associated with the user `SRV01\mcharles`. *Interactive* means that the credential is used for interactive logon sessions. Whenever we come across this type of credential, we can use `runas` to impersonate the stored user, like so:

```powershell
runas /savecred /user:SRV01\mcharles cmd
```

## Extracting with Mimikatz

Mimikatz can be used to decrypt stored credentials. There are multiple ways to attack these credentials — we can either dump credentials from memory using the `sekurlsa` module, or we can manually decrypt credentials using the `dpapi` module. For this example, we will target the LSASS process with `sekurlsa`. Remember that these commands must run as admin:

```
privilege::debug
sekurlsa::credman
```

![Extracting credentials from LSASS with mimikatz's sekurlsa module](/assets/img/posts/attacking-windows-credential-manager/mimikatz-credman.png)
_Extracting Credential Manager entries with mimikatz_
