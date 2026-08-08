---
title: "Caçando Credenciais na Rede"
date: 2026-07-29 10:00:00 -0300
categories: [Conceitos, Redes]
tags: [network, credential-dumping, credential-hunting, wireshark, pcredz, snaffler, powerhuntshares, manspider, netexec]
---

Muitas aplicações modernas usam TLS para proteger dados em trânsito, mas sistemas legados, mal configurados ou de teste ainda podem depender de protocolos não criptografados, expondo informações sensíveis. Atacantes podem explorar esses protocolos em texto plano para capturar credenciais do tráfego de rede usando ferramentas como Wireshark ou Pcredz.

## Tráfego de Rede

### Wireshark

O Wireshark é um analisador de pacotes poderoso que permite a inspeção eficiente de tráfego de rede ao vivo ou capturado através do seu mecanismo flexível de filtros. Ao aplicar filtros ou buscar por strings específicas, como "password" em tráfego HTTP não criptografado, analistas conseguem identificar rapidamente credenciais expostas e informações sensíveis.

![Caçando credenciais com o Wireshark](/assets/img/posts/credential-hunting-network/wireshark.png)
_Caçando credenciais com o Wireshark_

### Pcredz

O [Pcredz](https://github.com/lgandx/PCredz) é uma ferramenta de extração de credenciais que analisa tráfego de rede ao vivo ou capturas de pacotes para identificar credenciais expostas, hashes de autenticação e outros dados sensíveis em múltiplos protocolos. Ele suporta a extração de informações como FTP, SMTP, POP, IMAP, HTTP, SNMP, NTLM, Kerberos e dados de cartão de crédito a partir do tráfego de rede. Ao processar um arquivo PCAP ou monitorar uma interface ao vivo, o Pcredz automatiza rapidamente a descoberta de credenciais que, de outra forma, exigiriam análise manual.

![Caçando credenciais com o Pcredz](/assets/img/posts/credential-hunting-network/pcredz.png)
_Caçando credenciais com o Pcredz_

## Compartilhamentos de Rede

Compartilhamentos de rede corporativos frequentemente contêm informações valiosas, mas credenciais mal armazenadas e arquivos de configuração podem torná-los um alvo atraente para atacantes. Uma caça de credenciais eficaz envolve buscar por palavras-chave sensíveis, tipos de arquivo comuns, e priorizar estrategicamente compartilhamentos de alto valor, como os usados por equipes de TI. Ferramentas como MANSPIDER, Snaffler e NetExec automatizam esse processo, facilitando a descoberta de segredos expostos em compartilhamentos de rede Windows e Linux.

### Caçando a partir do Windows

#### Snaffler

O [Snaffler](https://github.com/SnaffCon/Snaffler) é uma ferramenta em C# que descobre automaticamente compartilhamentos de rede acessíveis num ambiente de Active Directory e os vasculha em busca de arquivos que contenham possíveis credenciais ou outras informações sensíveis. Ele usa regras de detecção nativas para identificar arquivos e padrões interessantes, embora revisão manual costume ser necessária para filtrar falsos positivos. Opções como `-u`, `-i` e `-n` ajudam a refinar as buscas mirando usuários do Active Directory ou compartilhamentos de rede específicos.

![Caçando credenciais com o Snaffler](/assets/img/posts/credential-hunting-network/snaffler.png)
_Caçando credenciais com o Snaffler_

#### PowerHuntShares

O [PowerHuntShares](https://github.com/NetSPI/PowerHuntShares) é uma ferramenta em PowerShell que automatiza a descoberta e avaliação de compartilhamentos SMB em todo um domínio Windows, mesmo quando executada a partir de uma máquina fora do domínio. Ela identifica compartilhamentos acessíveis, analisa permissões, detecta possíveis riscos de segurança e gera um relatório HTML completo para fácil revisão. Embora seja bastante eficaz para enumeração de compartilhamentos em grande escala, os scans podem levar um tempo considerável em ambientes corporativos grandes.

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public
```

### Caçando a partir do Linux

#### MANSPIDER

O [MANSPIDER](https://github.com/blacklanternsecurity/MANSPIDER) é uma ferramenta baseada em Linux que varre remotamente compartilhamentos SMB em busca de arquivos sensíveis e credenciais, sem exigir acesso a uma máquina no domínio. Ele suporta buscas baseadas em conteúdo, baixa os arquivos correspondentes para análise posterior, e costuma ser executado através do seu container Docker oficial para simplificar a implantação. Suas opções flexíveis de configuração o tornam uma solução eficaz para caçar credenciais em ambientes Windows a partir de uma máquina de ataque Linux.

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider 10.129.234.121 -c 'passw' -u 'user' -p 'password!'
```

#### NetExec

O NetExec consegue buscar em compartilhamentos de rede SMB por informações sensíveis usando sua opção `--spider`, permitindo tanto a inspeção de nome de arquivo quanto de conteúdo. Ao buscar por palavras-chave como "passw", ele ajuda a identificar rapidamente arquivos que podem conter credenciais expostas ou outros segredos. Essa capacidade torna o NetExec uma ferramenta versátil para automatizar a caça de credenciais durante avaliações de redes Windows.

```bash
nxc smb 10.129.234.121 -u user -p 'password!' --spider IT --content --pattern "passw"
```