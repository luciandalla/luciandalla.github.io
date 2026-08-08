---
title: "Processo de Autenticação do Linux"
date: 2026-07-24 10:00:00 -0300
categories: [Conceitos, Linux]
tags: [linux, pam, credential-dumping, john-the-ripper, hashcat, post-exploitation]
---

Distribuições baseadas em Linux suportam vários mecanismos de autenticação. Um dos mais usados é o Pluggable Authentication Modules (PAM). Esse módulo geralmente fica localizado em `/usr/lib/x86_64-linux-gnu/security/` e interage com os arquivos `/etc/passwd` e `/etc/shadow`. A biblioteca PAM também pode impedir que usuários reutilizem senhas antigas. Essas senhas anteriores ficam armazenadas no arquivo `/etc/security/opasswd`. Privilégios de administrador (root) são necessários para ler esse arquivo.

## /etc/passwd

O arquivo `/etc/passwd` contém informações sobre todos os usuários do sistema. O arquivo é legível por todos os usuários. Cada linha do arquivo segue a estrutura `[username]:[password]:[userID]:[groupID]:[GECOS]:[home-directory]:[default-shell]`.

![Permissões e conteúdo do arquivo /etc/passwd](/assets/img/posts/linux-authentication-process/passwd-file.png)
_Permissões e conteúdo do arquivo /etc/passwd_

Em casos raros, o arquivo armazena os hashes de senha — isso acontece só em sistemas antigos. Normalmente, o campo vem preenchido com `x`. Em sistemas Linux modernos, os hashes de senha ficam armazenados no arquivo `/etc/shadow`.

## /etc/shadow

O arquivo `/etc/shadow` é responsável pelo armazenamento e gerenciamento de senhas. Todo usuário registrado em `/etc/passwd` precisa ter uma entrada correspondente nesse arquivo, ou será considerado inválido. A estrutura do arquivo é `[username]:[password]:[last-change]:[min-age]:[max-age]:[warning-period]:[inactivity-period]:[expiration-date]:[reserved-field]`.

![Exemplo de dados no arquivo /etc/shadow](/assets/img/posts/linux-authentication-process/shadow-file.png)
_Exemplo de dados no arquivo /etc/shadow_

Se o campo de senha contiver um caractere como `!` ou `*`, o usuário não consegue fazer login usando uma senha Unix. Porém, outros métodos de autenticação ainda podem ser usados. Se o campo de senha estiver vazio, nenhuma senha é exigida para o login.

O campo de senha também segue um formato específico, do qual conseguimos extrair informações adicionais: `$<id>$<salt>$<hashed>`. O `id` especifica qual algoritmo criptográfico de hash foi usado:

| ID | Algoritmo |
|----|-----------|
| `1` | MD5 |
| `2a` | Blowfish |
| `5` | SHA-256 |
| `6` | SHA-512 |
| `sha1` | SHA1crypt |
| `y` | Yescrypt |
| `gy` | Gost-yescrypt |
| `7` | Scrypt |

## Quebrando Credenciais do Linux

Assim que tivermos acesso root numa máquina Linux, podemos coletar os hashes de senha dos usuários e tentar quebrá-los usando vários métodos para recuperar as senhas em texto plano.

Para isso, podemos usar uma ferramenta chamada `unshadow`, que já vem incluída no John the Ripper (JtR). Ela funciona combinando os arquivos `/etc/passwd` e `/etc/shadow` num único arquivo adequado para a quebra:

```bash
sudo unshadow /etc/passwd /etc/shadow > /tmp/unshadowed.hashes
```

Com o arquivo unshadowed pronto, podemos quebrá-lo usando o Hashcat ou outra ferramenta:

```bash
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```