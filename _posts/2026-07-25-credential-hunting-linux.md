---
title: "Caçando Credenciais no Linux"
date: 2026-07-25 10:00:00 -0300
categories: [Conceitos, Linux]
tags: [linux, credential-dumping, credential-hunting, mimipenguin, lazagne, firefox-decrypt, post-exploitation]
---

Caçar credenciais é um dos primeiros passos de pós-exploração, já que credenciais expostas podem levar rapidamente à escalada de privilégio. Credenciais valiosas podem ser encontradas em arquivos de configuração, bancos de dados, scripts, logs, histórico de comandos, memória ou chaveiros de navegador. Um entendimento aprofundado do papel e do ambiente do sistema alvo ajuda a priorizar onde buscar e aumenta as chances de descobrir credenciais úteis.

## Arquivos

### Arquivos de Configuração

O primeiro passo na enumeração do sistema é identificar arquivos de configuração, já que eles fornecem informações valiosas sobre a configuração do sistema e possíveis fraquezas de segurança. Restringir a busca às extensões de arquivo de configuração comuns torna o processo de enumeração mais eficiente e focado.

```bash
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done
```

### Bancos de Dados

Podemos aplicar a mesma abordagem de busca a outras extensões de arquivo, como as usadas por bancos de dados, e depois inspecionar os arquivos encontrados em busca de informações sensíveis.

```bash
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

### Notas

Buscar por notas pode revelar informações valiosas, incluindo credenciais, detalhes do sistema e documentação de processos internos. Como notas podem ter nomes arbitrários ou nenhuma extensão de arquivo, é importante buscar tanto por arquivos `.txt` quanto por arquivos sem extensão durante a enumeração.

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

### Scripts

Scripts frequentemente contêm informações sensíveis, incluindo credenciais fixas no código usadas para automatizar tarefas e processos do sistema. Examinar scripts pode revelar senhas ou outros segredos que podem ser aproveitados para escalada de privilégio ou movimentação lateral.

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

### Cronjobs

Cron jobs automatizam a execução de comandos e scripts, às vezes expondo credenciais fixas no código necessárias para tarefas agendadas. Revisar as configurações de cron do sistema e específicas de cada usuário pode revelar informações sensíveis e possíveis oportunidades de escalada de privilégio.

```bash
cat /etc/crontab
```

### Arquivos de Histórico

Arquivos de histórico fornecem informações valiosas sobre atividades anteriores do usuário e operações do sistema, frequentemente revelando comandos, credenciais ou ações administrativas. Revisar arquivos como `.bash_history`, `.bashrc` e `.bash_profile` pode revelar informações úteis para escalada de privilégio e enumeração do sistema.

```bash
tail -n5 /home/*/.bash*
```

### Arquivos de Log

Arquivos de log registram atividade do sistema, eventos de autenticação, comportamento de serviços e erros de aplicação, sendo uma fonte valiosa de informação durante a enumeração. Revisar logs pode revelar tentativas de login malsucedidas, tarefas agendadas, configurações de serviço e outros indicadores úteis para escalada de privilégio. Entender o propósito e o formato dos arquivos de log comuns ajuda a identificar informações relevantes de forma mais eficiente durante avaliações de segurança.

```bash
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null;fi;done
```

## Memória e Cache

### MimiPenguin

Credenciais usadas por aplicações e usuários autenticados podem ficar armazenadas em memória ou em chaveiros de navegador, tornando-as alvos valiosos durante a pós-exploração. Ferramentas como o [mimipenguin](https://github.com/huntergregal/mimipenguin) conseguem extrair essas credenciais de sistemas Linux, embora exijam privilégios de root para operar.

```bash
sudo python3 mimipenguin.py
```

### LaZagne

O [LaZagne](https://github.com/AlessandroZ/LaZagne) é uma ferramenta poderosa de extração de credenciais, capaz de recuperar senhas e hashes de diversas fontes, incluindo navegadores, SSH, Wi-Fi, serviços de nuvem, chaveiros e arquivos de configuração. Ele suporta uma ampla gama de armazenamentos de credenciais no Linux, tornando-o uma ferramenta eficaz para pós-exploração e caça de credenciais.

```bash
sudo python2.7 laZagne.py all
```

### Firefox Decrypt

O Firefox armazena credenciais salvas em arquivos criptografados como o logins.json, mas essas senhas ainda podem ser recuperadas se um atacante obtiver acesso ao perfil do usuário. O [Firefox Decrypt](https://github.com/unode/firefox_decrypt) é uma ferramenta especializada que extrai e descriptografa essas credenciais armazenadas, sendo valiosa para a caça de credenciais durante a pós-exploração.

```bash
python3.9 firefox_decrypt.py
```