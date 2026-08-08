---
title: "Atacando o Windows Credential Manager"
date: 2026-07-21 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, credential-manager, mimikatz, post-exploitation]
---

O **Credential Manager** é um recurso presente no Windows desde o Windows Server 2008 R2 e o Windows 7. Ele permite que usuários e aplicações armazenem, de forma segura, credenciais relevantes para outros sistemas e sites.

## Onde as Credenciais Ficam Armazenadas

As credenciais ficam armazenadas em pastas criptografadas especiais no computador, sob os perfis de usuário e de sistema:

- `%UserProfile%\AppData\Local\Microsoft\Vault\`
- `%UserProfile%\AppData\Local\Microsoft\Credentials\`
- `%UserProfile%\AppData\Roaming\Microsoft\Vault\`
- `%ProgramData%\Microsoft\Vault\`
- `%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\`

A Microsoft costuma se referir a esses armazenamentos protegidos como **Credential Lockers** (antigamente Windows Vaults). O Credential Manager é o recurso/API voltado ao usuário, enquanto os armazenamentos criptografados de fato são as pastas de vault/locker.

O Windows armazena dois tipos de credenciais:

- **Web Credentials**
- **Windows Credentials**

## Enumerando com o cmdkey

Podemos usar o `cmdkey` para enumerar as credenciais armazenadas no perfil do usuário atual:

```cmd
cmdkey /list
```

![Enumerando credenciais armazenadas com o cmdkey](/assets/img/posts/attacking-windows-credential-manager/cmdkey-enum.png)
_Enumerando credenciais com o cmdkey_

O segundo grupo de informação na imagem, `Domain:interactive=SRV01\mcharles`, é uma credencial de domínio associada ao usuário `SRV01\mcharles`. *Interactive* significa que a credencial é usada para sessões de logon interativas. Sempre que encontrarmos esse tipo de credencial, podemos usar o `runas` para nos passarmos pelo usuário armazenado, assim:

```cmd
runas /savecred /user:SRV01\mcharles cmd
```

## Extraindo com o Mimikatz

O Mimikatz pode ser usado para descriptografar credenciais armazenadas. Existem várias formas de atacar essas credenciais — podemos extrair credenciais da memória usando o módulo `sekurlsa`, ou descriptografar credenciais manualmente usando o módulo `dpapi`. Neste exemplo, vamos mirar o processo LSASS com o `sekurlsa`. Lembre-se de que esses comandos precisam rodar como admin:

```
privilege::debug
sekurlsa::credman
```

![Extraindo credenciais do LSASS com o módulo sekurlsa do mimikatz](/assets/img/posts/attacking-windows-credential-manager/mimikatz-credman.png)
_Extraindo entradas do Credential Manager com o mimikatz_