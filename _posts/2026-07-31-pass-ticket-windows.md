---
title: "Pass the Ticket (PtT) from Windows"
date: 2026-07-31 10:00:00 -0300
categories: [Concepts, Windows]
tags: [windows, kerberos, pass-the-ticket, overpass-the-hash, mimikatz, rubeus]
---

A **Pass the Ticket (PtT)** attack enables lateral movement by using stolen Kerberos tickets instead of NTLM password hashes for authentication. In Kerberos, users authenticate once to obtain a **Ticket Granting Ticket (TGT)** and later present it to the Key Distribution Center (KDC) to request **Ticket Granting Service (TGS)** tickets for specific services, without re-entering their password. Because authentication relies on valid tickets rather than the password itself, an attacker who steals a TGT or TGS — typically after gaining administrative access to a compromised machine — can impersonate the associated user and move laterally within an Active Directory environment. Tools such as **Mimikatz** and **Rubeus** can extract existing tickets or forge new ones for this purpose.

## Harvesting Kerberos Tickets

On Windows, Kerberos tickets are stored in the **LSASS** process, so local administrator privileges are required to extract tickets belonging to other users. With Mimikatz, the `sekurlsa::tickets /export` module dumps and exports all available tickets as `.kirbi` files. Ticket filenames help identify their purpose: tickets ending in **`$`** belong to computer accounts, user tickets follow the format **`username@service-domain.local.kirbi`**, and a ticket for the **`krbtgt`** service is the user's TGT.

[Rubeus](https://github.com/GhostPack/Rubeus) can also dump tickets directly to the console instead of saving them as files. When executed with local administrator privileges, it dumps tickets belonging to all logged-in users:

```powershell
Rubeus.exe dump /nowrap
```

Exported tickets are encoded in Base64; the `/nowrap` option removes line breaks from the output, making them easier to copy and reuse.

## OverPass the Hash (Pass the Key)

**OverPass the Hash** extends the traditional Pass the Hash technique by using a user's Kerberos encryption key (**RC4** or **AES**) instead of authenticating directly with NTLM, forging a valid TGT through the Kerberos authentication process instead. Mimikatz can extract these keys using the `sekurlsa::ekeys` module; both **Mimikatz** and **Rubeus** can then use an obtained key to request a TGT for subsequent Kerberos authentication and lateral movement.

### Mimikatz

![Mimikatz - Pass the Key aka. OverPass the Hash](/assets/img/posts/pass-ticket-windows/mimikatz.png)
_Mimikatz - Pass the Key aka. OverPass the Hash_

This creates a new `cmd.exe` window that can be used to request access to any service in the context of the target user. Mimikatz requires administrative rights to perform this attack.

### Rubeus

![Rubeus - Pass the Key aka. OverPass the Hash](/assets/img/posts/pass-ticket-windows/rubeus.png)
_Rubeus - Pass the Key aka. OverPass the Hash_

Rubeus can perform the same attack without requiring administrative privileges.

## Pass the Ticket (PtT)

Once a valid ticket is obtained, Rubeus can inject it into the current logon session using the `/ptt` option, enabling immediate use for authentication — whether requested directly from memory, imported from a previously exported `.kirbi` file, or supplied as its Base64 representation:

```powershell
# Request a TGT and inject it directly into the current session
Rubeus.exe asktgt /domain:inlanefreight.htb /user:<USER> /rc4:<RC4_HASH> /ptt

# Import a ticket from an exported .kirbi file
Rubeus.exe ptt /ticket:<TICKET_FILE>.kirbi

# Import a ticket from its Base64 representation
Rubeus.exe ptt /ticket:<BASE64_TICKET>
```

If needed, a `.kirbi` file can be converted to Base64 using PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("<TICKET_FILE>.kirbi"))
```

Mimikatz can achieve the same result using the `kerberos::ptt` module and a `.kirbi` file:

```powershell
mimikatz.exe privilege::debug "kerberos::ptt 'C:\Users\<USER>\Desktop\Mimikatz\<TICKET_FILE>.kirbi'"
```

### PowerShell Remoting with Pass the Ticket

Once a ticket is injected into the current session, any process launched from that session — including PowerShell — automatically uses it for Kerberos authentication. This means `Enter-PSSession` can establish a remote connection to another domain-joined system without providing credentials.

With Mimikatz, this only requires importing the ticket beforehand:

```powershell
mimikatz # kerberos::ptt "C:\Users\<USER>\Desktop\<TICKET_FILE>.kirbi"
```

With Rubeus, it's common to first create a **sacrificial logon session** using `createnetonly` (similar to `runas /netonly`), which isolates the new ticket in a separate Logon Type 9 session — preventing the current user's existing TGTs from being overwritten — before requesting and injecting the ticket:

```powershell
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
Rubeus.exe asktgt /user:<USER> /domain:inlanefreight.htb /aes256:<AES256_KEY> /ptt
```