---
title: "Atacando o Active Directory: Ataques de Senha e o NTDS.dit"
date: 2026-07-22 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, active-directory, ntds, credential-dumping, password-attacks, post-exploitation, netexec, kerbrute]
---

Entender o Active Directory (AD) e como atacá-lo é uma habilidade essencial para qualquer pentester. Muitas organizações usam Windows em toda a sua infraestrutura e dependem do Active Directory para gerenciar usuários, computadores e permissões.

## Ataque de Dicionário Contra o Active Directory

Antes de lançar um ataque, é importante entender a estrutura da organização alvo. Realizar um ataque de dicionário sem conhecer nenhum usuário válido pode gerar ruído desnecessário.

O primeiro passo é pesquisar a empresa e identificar o padrão de nomes de usuário. Isso geralmente pode ser feito procurando contas de funcionários associadas ao domínio da empresa. Depois de coletar uma lista de nomes de funcionários, podemos usar o [Username Anarchy](https://github.com/urbanadventurer/username-anarchy) para gerar variações comuns de nome de usuário a partir desses nomes:

```bash
./username-anarchy -i names.txt
```

Com uma lista de possíveis nomes de usuário, podemos usar o [Kerbrute](https://github.com/ropnop/kerbrute) para enumerar usuários válidos do Active Directory:

```bash
./kerbrute userenum --dc <IP> --domain <DOMAIN> <WORDLIST>
```

![Enumerando usuários válidos do Active Directory com o Kerbrute](/assets/img/posts/attacking-ad-ntds/kerbrute-userenum.png)
_Enumerando usuários válidos do domínio com o Kerbrute_

Assim que tivermos uma lista de usuários válidos, podemos realizar um ataque de dicionário de senhas contra o Active Directory usando o NetExec:

```bash
netexec smb <IP> -u <USER> -p <WORDLIST>
```

![NetExec realizando um ataque de dicionário de senhas](/assets/img/posts/attacking-ad-ntds/netexec-password-attack.png)
_Quebrando uma senha válida com o NetExec_

Fique atento que algumas organizações aplicam políticas de bloqueio de conta após um certo número de tentativas de login malsucedidas. Porém, isso não vem habilitado por padrão no Active Directory.

## Capturando o NTDS.dit

O **NT Directory Services (NTDS)** é o serviço de diretório usado pelo Active Directory para organizar e gerenciar recursos de rede. O arquivo `NTDS.dit` é o banco de dados mais importante do domínio — ele armazena nomes de usuário do domínio, hashes de senha e outras informações críticas do diretório. Capturar esse arquivo nos permite extrair os hashes de senha e quebrá-los offline.

Com um nome de usuário e senha válidos, podemos nos conectar ao alvo usando o Evil-WinRM, que fornece uma sessão remota do PowerShell:

```bash
evil-winrm -i <IP> -u <USER> -p <PASSWORD>
```

Depois de conectado, o primeiro passo é verificar os privilégios do usuário usando `net localgroup` e `net user <username>`. Privilégios administrativos ou de Domain Admin são necessários para copiar o arquivo `NTDS.dit`.

Poderíamos criar uma Volume Shadow Copy (VSS) para copiar o arquivo, mas a forma mais rápida de extrair o banco de dados `NTDS.dit` é executando o NetExec a partir da máquina de ataque:

```bash
netexec smb <IP> -u <USER> -p <PASSWORD> -M ntdsutil
```

![Extraindo o NTDS.dit remotamente com o NetExec](/assets/img/posts/attacking-ad-ntds/netexec-ntds-dump.png)
_Extraindo o banco de dados NTDS.dit remotamente com o NetExec_

Parabéns! Agora você tem os hashes de senha do domínio, que podem ser quebrados offline usando ferramentas como o Hashcat ou o John the Ripper.