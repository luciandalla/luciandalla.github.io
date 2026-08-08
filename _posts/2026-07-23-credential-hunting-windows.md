---
title: "Caçando Credenciais no Windows"
date: 2026-07-23 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, credential-hunting, lazagne, findstr, post-exploitation]
---

Assim que tivermos acesso a uma máquina Windows alvo, é importante caçar credenciais por todo o sistema de arquivos. Muitos sistemas operacionais têm recursos de busca nativos, que facilitam a procura por senhas em documentos. O primeiro passo é se perguntar como o usuário da máquina usa o sistema, para focar a busca com objetivos claros.

Alguns termos-chave podem ser usados na nossa busca:

- `password`
- `passphrase`
- `key`
- `username`
- `user account`
- `creds`
- `users`
- `passkey`
- `configuration`
- `dbcredential`
- `dbpassword`
- `pwd`
- `login`
- `credentials`

## Caça Manual de Credenciais

Se tivermos acesso à interface gráfica, é natural começar a busca usando o recurso de pesquisa nativo do Windows.

![Buscando arquivos relacionados a credenciais com o Windows Search](/assets/img/posts/credential-hunting-windows/windows-search.png)
_Buscando credenciais com o recurso de pesquisa nativo do Windows_

Também podemos usar o `findstr` para buscar padrões em vários tipos de arquivo. Tendo em mente os termos-chave comuns, podemos usar variações desse comando para descobrir credenciais num alvo Windows:

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

## Descobrindo com o LaZagne

O [LaZagne](https://github.com/AlessandroZ/LaZagne) é composto por módulos, cada um voltado a um software diferente na busca por senhas. Alguns dos módulos mais comuns estão descritos abaixo:

- **browsers**: extrai senhas de vários navegadores, incluindo Chromium, Firefox, Microsoft Edge e Opera
- **chats**: extrai senhas de várias aplicações de chat, incluindo Skype
- **mails**: vasculha caixas de e-mail em busca de senhas, incluindo Outlook e Thunderbird
- **memory**: extrai senhas da memória, mirando o KeePass e o LSASS
- **sysadmin**: extrai senhas dos arquivos de configuração de várias ferramentas de sysadmin, como OpenVPN e WinSCP
- **windows**: extrai credenciais específicas do Windows, mirando segredos LSA, o Credential Manager e mais
- **wifi**: extrai credenciais de WiFi

Assim que o LaZagne estiver instalado na máquina alvo, podemos executar o seguinte comando para rodar todos os módulos incluídos: `start LaZagne.exe all`

![Extraindo credenciais armazenadas com o LaZagne](/assets/img/posts/credential-hunting-windows/lazagne-execution.png)
_Extraindo credenciais com o LaZagne_