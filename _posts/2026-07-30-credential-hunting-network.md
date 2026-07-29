---
title: "Credential Hunting on the Network"
date: 2026-07-29 10:00:00 -0300
categories: [Concepts, Networking]
tags: [network, credential-dumping, credential-hunting, wireshark, pcredz, snaffler, powerhuntshares, manspider, netexec]
---

Many modern applications use TLS to protect data in transit, but legacy, misconfigured, or test systems may still rely on unencrypted protocols, exposing sensitive information. Attackers can exploit these plaintext protocols to capture credentials from network traffic using tools such as Wireshark or Pcredz.

## Network Traffic

### Wireshark

Wireshark is a powerful packet analyzer that enables efficient inspection of live or captured network traffic through its flexible filtering engine. By applying filters or searching for specific strings, such as "password" in unencrypted HTTP traffic, analysts can quickly identify exposed credentials and sensitive information.

![Hunting credentials with Wireshark](/assets/img/posts/credential-hunting-network/wireshark.png)
_Hunting credentials with Wireshark_

### Pcredz

[Pcredz](https://github.com/lgandx/PCredz) is a credential extraction tool that analyzes live network traffic or packet captures to identify exposed credentials, authentication hashes, and other sensitive data across multiple protocols. It supports extracting information such as FTP, SMTP, POP, IMAP, HTTP, SNMP, NTLM, Kerberos, and credit card data from network traffic. By processing a PCAP file or monitoring a live interface, Pcredz quickly automates the discovery of credentials that would otherwise require manual analysis.

![Hunting credentials with Pcredz](/assets/img/posts/credential-hunting-network/pcredz.png)
_Hunting credentials with Pcredz_

## Network Shares

Corporate network shares often contain valuable information, but improperly stored credentials and configuration files can make them an attractive target for attackers. Effective credential hunting involves searching for sensitive keywords, common file types, and strategically prioritizing high-value shares, such as those used by IT teams. Tools like MANSPIDER, Snaffler, and NetExec automate this process, making it easier to discover exposed secrets across Windows and Linux network shares.

### Hunting from Windows

#### Snaffler

[Snaffler](https://github.com/SnaffCon/Snaffler) is a C# tool that automatically discovers accessible network shares in an Active Directory environment and searches them for files containing potential credentials or other sensitive information. It uses built-in detection rules to identify interesting files and patterns, though manual review is often required to filter out false positives. Options such as `-u`, `-i`, and `-n` help refine searches by targeting Active Directory users or specific network shares.

![Hunting credentials with Snaffler](/assets/img/posts/credential-hunting-network/snaffler.png)
_Hunting credentials with Snaffler_

#### PowerHuntShares

[PowerHuntShares](https://github.com/NetSPI/PowerHuntShares) is a PowerShell tool that automates the discovery and assessment of SMB shares across a Windows domain, even when executed from a non-domain-joined machine. It identifies accessible shares, analyzes permissions, detects potential security risks, and generates a comprehensive HTML report for easy review. Although highly effective for large-scale share enumeration, scans may take considerable time in large enterprise environments.

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public
```

### Hunting from Linux

#### MANSPIDER

[MANSPIDER](https://github.com/blacklanternsecurity/MANSPIDER) is a Linux-based tool that remotely scans SMB shares for sensitive files and credentials without requiring access to a domain-joined machine. It supports content-based searches, downloads matching files for further analysis, and is commonly executed through its official Docker container to simplify deployment. Its flexible configuration options make it an effective solution for hunting credentials in Windows environments from a Linux attack host.

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider 10.129.234.121 -c 'passw' -u 'user' -p 'password!'
```

#### NetExec

NetExec can search SMB network shares for sensitive information using its `--spider` option, allowing both file name and content inspection. By searching for keywords such as "passw", it helps quickly identify files that may contain exposed credentials or other secrets. This capability makes NetExec a versatile tool for automating credential hunting during Windows network assessments.

```bash
nxc smb 10.129.234.121 -u user -p 'password!' --spider IT --content --pattern "passw"
```
