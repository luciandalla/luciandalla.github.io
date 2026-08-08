---
title: "Extraindo Hashes do SAM e Segredos LSA no Windows"
date: 2026-07-19 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, registry, sam, dpapi, post-exploitation, hashcat, netexec]
---

Com acesso administrativo a um sistema Windows, conseguimos extrair rapidamente os arquivos associados ao banco de dados SAM, transferi-los para nossa máquina de ataque e começar a quebrar os hashes offline.

## Hives do Registro

- **HKLM\SAM**: contém os hashes de senha das contas de usuário locais
- **HKLM\SYSTEM**: armazena a boot key do sistema, usada para criptografar o banco de dados SAM
- **HKLM\SECURITY**: contém informações sensíveis usadas pela Local Security Authority (LSA)

## Extraindo os Hives Localmente

Esses hives podem ser copiados usando o `reg.exe`. Isso requer privilégios de **SYSTEM**.

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

![Extraindo os hives SAM, SYSTEM e SECURITY com reg.exe](/assets/img/posts/dumping-sam-hashes-windows/reg-save.png)
_Extraindo os hives do registro localmente com reg.exe_

## Quebrando os Hashes

Depois de mover os arquivos para a máquina de ataque, podemos usar o **secretsdump do Impacket** para extrair os hashes:

```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py \
  -sam sam.save -security security.save -system system.save LOCAL
```

É importante sempre extrair o `HKLM\SYSTEM` junto com o `HKLM\SAM`, porque o `HKLM\SYSTEM` guarda a boot key, necessária para descriptografar o
`HKLM\SAM`.

![Saída do secretsdump.py mostrando os hashes extraídos](/assets/img/posts/dumping-sam-hashes-windows/secretsdump-output.png)
_Extraindo os hashes offline com o secretsdump_

Preste atenção à linha de saída `Dumping local SAM hashes (uid:rid:lmhash:nthash)`, que especifica o tipo de hash.

Podemos salvar os hashes em um arquivo e usar o hashcat para quebrá-los:

```bash
sudo hashcat -m 1000 hashesfile.txt /usr/share/wordlists/rockyou.txt
```

![hashcat quebrando os hashes NT extraídos](/assets/img/posts/dumping-sam-hashes-windows/hashcat-cracked.png)
_Quebrando os hashes extraídos com o hashcat_

## HKLM\SECURITY: Credenciais de Domínio em Cache e DPAPI

O `HKLM\SECURITY` contém informações de logon de domínio em cache, especificamente na forma de hashes **DCC2**. Esse tipo de hash, identificado como modo `2100` no hashcat, é muito mais lento de quebrar que hashes NT — é realmente difícil de quebrar dentro do prazo típico de um pentest.

O `HKLM\SECURITY` também contém chaves de máquina e de usuário para o **DPAPI** (Data Protection Application Programming Interface). Ele é usado para proteger dados como senhas de preenchimento automático do Chrome e Internet Explorer, senhas de contas de e-mail, credenciais de conexões remotas e credenciais de acesso a recursos compartilhados, redes wireless e VPNs.

Podemos usar o `mimikatz.exe` para descriptografar esses dados manualmente. Por exemplo:

```
dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" unprotect
```

## Extração Remota

Com acesso a credenciais que tenham privilégios de administrador local, é possível extrair os segredos LSA e o SAM remotamente, por exemplo com os seguintes comandos:

```bash
netexec smb <ip> --local-auth -u <user> -p <password> --lsa
netexec smb <ip> --local-auth -u <user> -p <password> --sam
```

![netexec extraindo segredos LSA remotamente](/assets/img/posts/dumping-sam-hashes-windows/netexec-lsa-sam.png)
_Extração remota de segredos LSA com netexec_
