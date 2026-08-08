---
title: "Pass the Ticket (PtT) a partir do Windows"
date: 2026-07-31 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, kerberos, pass-the-ticket, overpass-the-hash, mimikatz, rubeus]
---

Um ataque de **Pass the Ticket (PtT)** permite movimentação lateral usando tickets Kerberos roubados em vez de hashes de senha NTLM para autenticação. No Kerberos, usuários se autenticam uma vez para obter um **Ticket Granting Ticket (TGT)** e depois o apresentam ao Key Distribution Center (KDC) para solicitar tickets **Ticket Granting Service (TGS)** para serviços específicos, sem precisar reinserir a senha. Como a autenticação depende de tickets válidos e não da senha em si, um atacante que rouba um TGT ou TGS — normalmente após obter acesso administrativo numa máquina comprometida — consegue se passar pelo usuário associado e se mover lateralmente dentro de um ambiente de Active Directory. Ferramentas como **Mimikatz** e **Rubeus** conseguem extrair tickets existentes ou forjar novos para essa finalidade.

## Coletando Tickets Kerberos

No Windows, os tickets Kerberos ficam armazenados no processo **LSASS**, então privilégios de administrador local são necessários para extrair tickets pertencentes a outros usuários. Com o Mimikatz, o módulo `sekurlsa::tickets /export` extrai e exporta todos os tickets disponíveis como arquivos `.kirbi`. Os nomes dos arquivos de ticket ajudam a identificar seu propósito: tickets terminados em **`$`** pertencem a contas de computador, tickets de usuário seguem o formato **`username@service-domain.local.kirbi`**, e um ticket para o serviço **`krbtgt`** é o TGT do usuário.

O [Rubeus](https://github.com/GhostPack/Rubeus) também consegue extrair tickets diretamente no console, em vez de salvá-los como arquivos. Quando executado com privilégios de administrador local, ele extrai os tickets de todos os usuários logados:

```powershell
Rubeus.exe dump /nowrap
```

Os tickets exportados são codificados em Base64; a opção `/nowrap` remove as quebras de linha da saída, facilitando copiá-los e reutilizá-los.

## OverPass the Hash (Pass the Key)

O **OverPass the Hash** estende a técnica tradicional de Pass the Hash usando a chave de criptografia Kerberos de um usuário (**RC4** ou **AES**) em vez de autenticar diretamente com NTLM, forjando um TGT válido através do processo de autenticação Kerberos. O Mimikatz consegue extrair essas chaves usando o módulo `sekurlsa::ekeys`; tanto o **Mimikatz** quanto o **Rubeus** conseguem então usar uma chave obtida para solicitar um TGT para autenticação Kerberos subsequente e movimentação lateral.

### Mimikatz

![Mimikatz - Pass the Key, ou OverPass the Hash](/assets/img/posts/pass-ticket-windows/mimikatz.png)
_Mimikatz - Pass the Key, ou OverPass the Hash_

Isso cria uma nova janela do `cmd.exe` que pode ser usada para solicitar acesso a qualquer serviço no contexto do usuário alvo. O Mimikatz requer privilégios administrativos para realizar esse ataque.

### Rubeus

![Rubeus - Pass the Key, ou OverPass the Hash](/assets/img/posts/pass-ticket-windows/rubeus.png)
_Rubeus - Pass the Key, ou OverPass the Hash_

O Rubeus consegue realizar o mesmo ataque sem exigir privilégios administrativos.

## Pass the Ticket (PtT)

Assim que um ticket válido é obtido, o Rubeus consegue injetá-lo na sessão de logon atual usando a opção `/ptt`, permitindo uso imediato para autenticação — seja solicitado diretamente da memória, importado de um arquivo `.kirbi` exportado previamente, ou fornecido como sua representação em Base64:

```powershell
# Solicita um TGT e o injeta diretamente na sessão atual
Rubeus.exe asktgt /domain:inlanefreight.htb /user:<USER> /rc4:<RC4_HASH> /ptt

# Importa um ticket de um arquivo .kirbi exportado
Rubeus.exe ptt /ticket:<TICKET_FILE>.kirbi

# Importa um ticket a partir da sua representação em Base64
Rubeus.exe ptt /ticket:<BASE64_TICKET>
```

Se necessário, um arquivo `.kirbi` pode ser convertido para Base64 usando o PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("<TICKET_FILE>.kirbi"))
```

O Mimikatz consegue alcançar o mesmo resultado usando o módulo `kerberos::ptt` e um arquivo `.kirbi`:

```powershell
mimikatz.exe privilege::debug "kerberos::ptt 'C:\Users\<USER>\Desktop\Mimikatz\<TICKET_FILE>.kirbi'"
```

### PowerShell Remoting com Pass the Ticket

Assim que um ticket é injetado na sessão atual, qualquer processo iniciado a partir dessa sessão — incluindo o PowerShell — passa a usá-lo automaticamente para autenticação Kerberos. Isso significa que o `Enter-PSSession` consegue estabelecer uma conexão remota com outro sistema no domínio sem fornecer credenciais.

Com o Mimikatz, isso só exige importar o ticket previamente:

```powershell
mimikatz # kerberos::ptt "C:\Users\<USER>\Desktop\<TICKET_FILE>.kirbi"
```

Com o Rubeus, é comum primeiro criar uma **sessão de logon sacrificial** usando o `createnetonly` (parecido com `runas /netonly`), que isola o novo ticket numa sessão separada do tipo Logon Type 9 — evitando que os TGTs existentes do usuário atual sejam sobrescritos — antes de solicitar e injetar o ticket:

```powershell
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
Rubeus.exe asktgt /user:<USER> /domain:inlanefreight.htb /aes256:<AES256_KEY> /ptt
```