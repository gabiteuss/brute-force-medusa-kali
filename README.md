# 🔐 Ataque de Força Bruta de Senhas com Medusa e Kali Linux

![Kali Linux](https://img.shields.io/badge/Kali%20Linux-2026.2-557C94?logo=kalilinux&logoColor=white)
![Medusa](https://img.shields.io/badge/Ferramenta-Medusa-red)
![Alvo](https://img.shields.io/badge/Alvo-Metasploitable%202-orange)
![Ambiente](https://img.shields.io/badge/Ambiente-Laborat%C3%B3rio%20Isolado-success)

> Desafio de Projeto da **DIO** — *Criando um Ataque de Brute Force de Senhas com Medusa e Kali Linux*.
> Documentação prática de uma cadeia completa de ataque a serviços de rede (SMB e FTP), feita em laboratório próprio e autorizado, com foco em **entender a falha para saber como defender**.

---

## ⚠️ Aviso ético e legal

Todo o conteúdo deste repositório foi executado em um **laboratório isolado, próprio e autorizado**:

- Máquinas virtuais no **VirtualBox**, em rede **Host-Only** (`192.168.56.0/24`), **sem acesso à internet**.
- O alvo é uma máquina **propositalmente vulnerável** (Metasploitable 2), criada justamente para treino.
- **Nada aqui é feito contra sistemas de terceiros.**

> 🚫 Testar força bruta, varredura ou acesso a sistemas que não são seus, **sem autorização por escrito**, é **crime no Brasil** (Lei 12.737/2012 — "Lei Carolina Dieckmann" — e art. 154-A do Código Penal).
>
> **Regra de ouro:** só se testa o que é seu ou o que você tem permissão explícita para testar.

---

## 🎯 Objetivos de aprendizagem

- Compreender como funcionam **ataques de força bruta** em serviços de rede (FTP, SMB, Web).
- Entender o conceito de **password spraying** e por que ele é mais discreto que a força bruta clássica.
- Usar **Kali Linux + Medusa** para auditoria de senhas em ambiente controlado.
- Reconhecer **vulnerabilidades comuns** (credenciais padrão, serviços desatualizados) e propor **mitigações**.
- Documentar o processo de forma clara e estruturada, usando o **GitHub como portfólio**.

---

## 🧪 Ambiente do laboratório

| Componente | Descrição |
| --- | --- |
| **Virtualizador** | VirtualBox |
| **Atacante** | Kali Linux 2026.2 |
| **Alvo** | Metasploitable 2 — IP `192.168.56.101` |
| **Rede** | Host-Only `192.168.56.0/24` (isolada, sem internet) |

### Topologia

```
┌─────────────────────────┐        Rede Host-Only        ┌─────────────────────────┐
│   Kali Linux 2026.2      │     192.168.56.0/24          │   Metasploitable 2      │
│   (Atacante)             │ ───────────────────────────► │   192.168.56.101        │
│   Nmap · Medusa          │     (isolada, sem internet)  │   (Alvo vulnerável)     │
│   smbclient · ftp        │                              │   FTP · SMB · etc.      │
└─────────────────────────┘                              └─────────────────────────┘
```

A cadeia completa do ataque foi:

**Enumeração → criação de wordlists → ataque com Medusa (SMB e FTP) → acesso e pós-exploração → lição de autorização.**

---

## 1️⃣ Enumeração / varredura de rede (Nmap)

Antes de atacar, é preciso saber **quem está na rede** e **quais serviços** estão rodando.

### Descobrir hosts ligados

```bash
nmap -sn 192.168.56.100-200 | grep 192 > varredura.txt
```

- `-sn` = *ping scan*: mostra **só os hosts ligados**, sem escanear portas.
- `grep 192` = filtra só as linhas com IP.
- `> varredura.txt` = salva o resultado num arquivo.

### Detectar sistema operacional e versões de serviço

```bash
nmap -O 192.168.56.101      # -O (maiúsculo) = detecta o Sistema Operacional
sudo nmap -sV 192.168.56.101 # -sV = detecta a versão dos serviços
```

> 💡 **Observação do laboratório:** na rede **NAT**, a detecção de SO (`-O`) deu errado. Trocando para **Host-Only**, o Nmap acertou (Linux 2.6.x). O `-O` precisa de pelo menos **1 porta aberta e 1 fechada** para funcionar bem.

### ⚠️ Cuidado com flags parecidas no Nmap

| Flag | O que faz |
| --- | --- |
| `-sV` | Detecta a **versão dos serviços** |
| `-O` (maiúsculo) | Detecta o **Sistema Operacional** |
| `-v` (minúsculo) | Modo **verboso** |
| `-oN arquivo.txt` | Salva a saída em **formato legível** |
| `-T` (maiúsculo) | **Velocidade** do scan (ex.: `-T4`) |

---

## 2️⃣ Serviços vulneráveis encontrados

A varredura revelou vários serviços antigos e vulneráveis no alvo:

| Porta | Serviço | Observação |
| --- | --- | --- |
| 21 | **vsftpd 2.3.4** (FTP) | Backdoor histórica famosa |
| 139 / 445 | **Samba 3.0.20** (SMB) | Versão antiga, vulnerável a RCE |
| 6667 | **UnrealIRCd** | Backdoor conhecida |

Estes serviços serão os alvos do ataque de força bruta com a Medusa.

---

## 3️⃣ Criação das wordlists

Força bruta precisa de **listas**: uma de **usuários** e uma de **senhas**.

### Lista de senhas

```bash
echo -e "password\n123456\nWecolme123\nmsfadmin" > senhas_spray.txt
```

- `echo -e` = o `-e` faz o `echo` **interpretar o `\n`** como "pular linha".
- `\n` = cada quebra vira **uma senha por linha**.
- `> senhas_spray.txt` = salva tudo no arquivo.

### Lista de usuários

Um arquivo `smb_users.txt` com os nomes de usuário a testar (ex.: `root`, `admin`, `msfadmin`...). A ideia é **cruzar usuários × senhas**.

---

## 4️⃣ Password spraying com Medusa (SMB)

```bash
medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

### O que cada flag faz

| Flag | Significado |
| --- | --- |
| `-h 192.168.56.101` | **Host** (o alvo) |
| `-U smb_users.txt` | Arquivo com a lista de **usuários** (U maiúsculo = arquivo) |
| `-P senhas_spray.txt` | Arquivo com a lista de **senhas** (P maiúsculo = arquivo) |
| `-M smbnt` | **Módulo** a usar — `smbnt` é o módulo de **SMB** |
| `-t 2` | Nº de **logins em paralelo por alvo** (threads) |
| `-T 50` | Nº de **alvos** testados em paralelo |

> 💡 **Dica de letra:** `-U` e `-P` (maiúsculos) apontam para um **arquivo**. `-u` e `-p` (minúsculos) seriam **um único** usuário ou senha.

### O que é "password spraying"?

- **Força bruta clássica:** muitas senhas contra **um único usuário**. Barulhento e costuma travar a conta (*lockout*).
- **Password spraying:** **poucas senhas** comuns contra **vários usuários**. Mais **discreto** e evita travar contas.

A ideia não é adivinhar a senha difícil de uma pessoa. Basta que **alguém** na lista use uma senha fraca.

### Resultado: credencial encontrada 🔑

```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

- **Usuário:** `msfadmin` · **Senha:** `msfadmin`

> 💡 **Lição:** é uma **credencial padrão** nunca trocada (usuário igual à senha) — um dos erros mais comuns do mundo real e dos mais fáceis de explorar.

---

## 5️⃣ Força bruta no FTP com Medusa

O mesmo princípio vale para o FTP (vsftpd 2.3.4), trocando apenas o módulo:

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 6
```

- `-M ftp` = módulo de **FTP**.

Resultado: as mesmas credenciais fracas deram acesso ao FTP (`msfadmin:msfadmin`). Isso mostra como uma senha padrão reutilizada abre **várias portas** de uma vez.

---

## 6️⃣ Acesso e pós-exploração SMB (smbclient)

Com a credencial na mão, dá para explorar os compartilhamentos.

### Listar os shares disponíveis

```bash
smbclient -L //192.168.56.101/tmp -U msfadmin
```

- `-L` = **listar** os shares (só mostra a lista e sai).
- `-U msfadmin` = autentica como o usuário (vai pedir a senha).

O servidor se identificou como **Samba 3.0.20-Debian** — versão antiga e vulnerável.

### Entrar de verdade num share (sem o `-L`)

```bash
smbclient //192.168.56.101/msfadmin -U msfadmin
```

Repare na **mudança de prompt** — isso confunde muita gente:

- `kali@kali:~$` → é o **terminal do Linux** (shell normal).
- `smb: \>` → é o **prompt de dentro do smbclient**. Você está "dentro" do compartilhamento remoto.

> 💡 **Ponto que confunde:** comandos como `get` são **internos do smbclient**, não do shell do Linux. Só funcionam dentro do prompt `smb: \>`.

### Dentro do share: listar e baixar

```
smb: \> ls
smb: \> get .profile                    # ✅ funcionou (baixou o arquivo)
smb: \> get .sudo_as_admin_successful   # ✅ funcionou
smb: \> get .mysql_history              # ❌ NT_STATUS_ACCESS_DENIED
```

---

## 7️⃣ A lição central: autenticação ≠ autorização

Este foi o aprendizado mais importante do laboratório.

- **Autenticação** = provar **quem você é** (a credencial `msfadmin:msfadmin` funcionou).
- **Autorização** = o que você **tem permissão** de fazer depois de entrar.

Mesmo com credencial **válida**, o `get .mysql_history` foi **negado**. Por quê? Porque as **permissões do Linux** (dono, grupo, `rwx`) **continuam valendo** por cima do SMB. **Logar não dá acesso a tudo.**

> 💡 **No pentest isso importa:** documentar **o que você conseguiu** (`.profile`) **e o que foi negado** (`.mysql_history`) faz parte do trabalho. O relatório precisa mostrar o alcance **real** do acesso, não só "consegui entrar".

### Acesso pelo cliente FTP nativo

```bash
ftp 192.168.56.101
# login: msfadmin
ftp> ls
ftp> pwd    # /home/msfadmin
```

> 💡 Dentro do prompt `ftp>` só funcionam **comandos de FTP** (`ls`, `cd`, `get`, `put`, `pwd`). Comandos de terminal como `echo` dão "Invalid command".

---

## 🛡️ Vulnerabilidades encontradas e mitigações

| Vulnerabilidade | Risco | Como corrigir / mitigar |
| --- | --- | --- |
| **Credencial padrão** (`msfadmin:msfadmin`) | Acesso imediato por força bruta | Trocar toda senha padrão por uma **forte e única** |
| **Samba 3.0.20 desatualizado** | Exploits conhecidos (RCE) | **Atualizar** o serviço para versão suportada |
| **vsftpd 2.3.4** com senha fraca | Acesso ao FTP e aos arquivos | Atualizar o serviço + política de senhas fortes |
| **Permissões amplas em shares** | Exfiltração de dados | Aplicar **menor privilégio** (permissões `rwx` restritas) |
| **Serviços legados** (telnet, IRCd antigo) | Superfície de ataque grande | **Desativar** o que não é usado |
| **Sem monitoramento de login** | Ataque passa despercebido | Monitorar/alertar sobre **muitas falhas de login** seguidas |

> A própria permissão correta no `.mysql_history` foi o que **barrou** a exfiltração. É a prova prática de que o **princípio do menor privilégio** funciona.

---

## 📸 Capturas de tela

As evidências visuais ficam na pasta `/images`:

<!-- Substitua os caminhos abaixo pelos seus prints -->
<!--
| Etapa | Print |
| --- | --- |
| Varredura com Nmap | ![Nmap](images/01-nmap.png) |
| Credencial encontrada (Medusa) | ![Medusa](images/02-medusa.png) |
| Acesso ao share (smbclient) | ![smbclient](images/03-smbclient.png) |
-->

> 💡 **Dica:** crie uma pasta `images/` no repositório e coloque aqui os prints de cada etapa (varredura, `ACCOUNT FOUND` da Medusa, o prompt `smb: \>` baixando arquivos). Depois, descomente a tabela acima e ajuste os nomes dos arquivos.

---

## 📁 Estrutura sugerida do repositório

```
.
├── README.md                 # este arquivo
├── wordlists/
│   ├── smb_users.txt         # lista de usuários
│   └── senhas_spray.txt      # lista de senhas
└── images/                   # capturas de tela das etapas
    ├── 01-nmap.png
    ├── 02-medusa.png
    └── 03-smbclient.png
```

---

## ✅ Conclusão e aprendizados

- A **força bruta** não depende de "quebrar" criptografia: ela explora **senhas fracas e padrão**. O elo mais fraco quase sempre é humano.
- O **password spraying** é mais discreto que a força bruta clássica, porque distribui poucas tentativas entre muitos usuários, evitando o *lockout*.
- Uma única credencial padrão (`msfadmin:msfadmin`) abriu **SMB e FTP** ao mesmo tempo — reutilização de senha multiplica o estrago.
- **Autenticação ≠ autorização:** entrar não é o mesmo que ter acesso a tudo. As permissões do sistema operacional seguem valendo.
- Documentar **o que funcionou e o que foi negado** é parte essencial do trabalho de um pentester — o relatório é o entregável que importa.

> Estudar ataque serve para **saber defender**. Cada falha encontrada aqui vira uma recomendação concreta de segurança.

---

*Desafio desenvolvido para a trilha de segurança da DIO, em ambiente de laboratório isolado e autorizado.*
