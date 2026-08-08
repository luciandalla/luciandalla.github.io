---
title: "Extraindo Credenciais do LSASS no Windows"
date: 2026-07-20 10:00:00 -0300
categories: [Conceitos, Windows]
tags: [windows, credential-dumping, lsass, mimikatz, pypykatz, hashcat, post-exploitation]
---

O **LSASS** (Local Security Authority Subsystem Service) é um processo central do Windows responsável por aplicar políticas de segurança, gerenciar a autenticação de usuários e armazenar material sensível de credenciais em memória.

Extrair o LSASS é parecido com extrair o banco de dados SAM: primeiro, criamos um arquivo com o dump de memória extraído e, depois de transferi-lo para a máquina de ataque, conseguimos quebrar as credenciais offline.

Abaixo estão duas técnicas para extrair o LSASS.

## Método 1: Gerenciador de Tarefas

Abra o Gerenciador de Tarefas, selecione a aba **Processos**, encontre e clique com o botão direito em **Local Security Authority Process**, e depois selecione **Criar arquivo de despejo**. Um arquivo chamado `lsass.DMP` é criado e salvo em `%temp%`.

![Criando um dump do LSASS pelo Gerenciador de Tarefas](/assets/img/posts/dumping-lsass-credentials/task-manager-dump.png)
_Criando um arquivo de dump pelo Gerenciador de Tarefas_

## Método 2: Rundll32.exe & Comsvcs.dll

Esse método é mais flexível porque não exige interface gráfica. Porém, algumas soluções de antivírus podem identificá-lo como malicioso. O primeiro passo é descobrir o PID do `lsass.exe`:

```powershell
tasklist /svc
```

ou, no PowerShell:

```powershell
Get-Process lsass
```

Com uma sessão elevada do PowerShell, podemos executar o seguinte comando para criar o arquivo de dump:

```powershell
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Com esse comando, executamos o `rundll32.exe` para chamar uma função exportada da `comsvcs.dll`, que por sua vez chama a função `MiniDumpWriteDump` (`MiniDump`) para despejar a memória do processo LSASS num local específico (`C:\lsass.dmp`).

## Extraindo Credenciais com o Pypykatz

Na máquina de ataque, podemos usar o **Pypykatz**, uma ferramenta poderosa que consegue extrair credenciais do arquivo `.dmp`. O Pypykatz é uma implementação em Python do Mimikatz (que só roda no Windows).

```bash
pypykatz lsa minidump <file.dmp>
```

![Extraindo credenciais do arquivo de dump com o pypykatz](/assets/img/posts/dumping-lsass-credentials/pypykatz-execution.png)
_Rodando o pypykatz contra o dump do LSASS_

A ferramenta consegue extrair dados de diferentes protocolos de autenticação, como:

- **MSV**: usado por versões mais recentes do Windows
- **WDIGEST**: usado por versões mais antigas do Windows, frequentemente armazenando credenciais em texto plano
- **Kerberos**
- **DPAPI** (masterkey)

## Quebrando o Hash

Depois de extrair o hash, podemos usar o hashcat para quebrá-lo.

![hashcat quebrando o hash extraído](/assets/img/posts/dumping-lsass-credentials/hashcat-cracked.png)
_Senha recuperada após quebrar o hash extraído com o hashcat_