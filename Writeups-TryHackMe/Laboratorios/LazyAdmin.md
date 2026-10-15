# Relatório de Auditoria de Segurança: CTF LazyAdmin

**Data:** 16 de Junho de 2026

**Alvo:** 10.65.146.126

**Objetivo:** Avaliação de vulnerabilidades, exploração e escalonamento de privilégios.

## 1. Resumo Executivo

O presente relatório documenta a análise de segurança realizada no ambiente "LazyAdmin". O processo compreendeu desde a enumeração inicial de serviços até o escalonamento de privilégios para o nível de superusuário (`root`). Identificou-se uma cadeia de falhas que permitiu a exposição de credenciais, execução remota de código (RCE) e a exploração de uma configuração incorreta de permissões (`sudo`).

## 2. Metodologia e Descobertas Técnicas

### 2.1. Fase de Enumeração e Reconhecimento

Durante a fase inicial, foram identificados os serviços expostos e a estrutura da aplicação web.

- **Comandos de Reconhecimento:**
    
    - `nmap -sC -sV -Pn 10.65.146.126`
        
    - `nmap -sC -sV -Pn -p- 10.65.146.126`
        
- **Descobertas:**
    
    - **Diretório Base:** `http://10.65.146.126/content/`
        
    - **CMS Identificado:** SweetRice (Versão 1.5.1)
        

### 2.2. Análise de Vulnerabilidades e Exploração (Foothold)

Identificou-se uma vulnerabilidade de _Backup Disclosure_ através do `searchsploit` (PoC 40718.txt), que apontou para a exposição de arquivos sensíveis.

- **Vetor de Ataque:** _Backup Disclosure_ para obtenção de credenciais.
    
- **URL de Exposição:** `http://10.65.146.126/content/inc/mysql_backup`
    
- **Análise do Banco de Dados:**
    
    - O arquivo de backup revelou a tabela `%--%_options` (Linha 14).
        
    - Formato: _PHP Serialized Array_.
        
    - **Credenciais Extraídas:**
        
        - **Usuário:** `manager`
            
        - **Hash (MD5):** `42f749ade7f9e195bf475f37a44cafcb`
            
        - **Senha Decodificada:** `Password123`
            

### 2.3. Execução Remota de Código (RCE)

Com as credenciais administrativas, realizou-se o acesso ao painel (`/content/as/`).

- **Método:** Exploração da funcionalidade de gerenciamento de anúncios ("Ads") para _Arbitrary File Upload_.
    
- **Ação:** Upload de _Reverse Shell_ (PHP) e execução via acesso direto ao caminho: `http://10.65.146.126/content/inc/ads/<arquivo_payload>.php`.
    
- **Resultado:** Acesso inicial ao sistema com o usuário `www-data`.
    

## 3. Pós-Exploração e Escalonamento de Privilégios

Após a obtenção do acesso inicial, focou-se na elevação de privilégios para o usuário `root`.

- **Enumeração:** O comando `sudo -l` revelou a seguinte configuração:
    
    - `User www-data may run the following commands on THM-Chal: (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl`
        
- **Análise:** O script `/home/itguy/backup.pl` executava incondicionalmente o arquivo `/etc/copy.sh`.
    
- **Exploração:**
    
    1. Verificou-se que o usuário `www-data` possuía permissão de escrita (`w`) no arquivo `/etc/copy.sh`.
        
    2. Injetou-se comandos arbitrários no arquivo `/etc/copy.sh`.
        
    3. A execução via `sudo /usr/bin/perl /home/itguy/backup.pl` resultou na execução do comando injetado com privilégios de superusuário (`root`).
        

## 4. Recomendações de Mitigação (Defesa)

Para mitigar as vulnerabilidades exploradas, recomenda-se a implementação das seguintes práticas:

|**Categoria**|**Recomendação**|
|---|---|
|**Proteção de Ativos**|Mover diretórios de backup e arquivos sensíveis para fora da _Web Root_, impedindo o acesso via navegador.|
|**Hardening do CMS**|Desabilitar a edição de código no painel administrativo e implementar _white-listing_ rigoroso para uploads de arquivos, validando _Magic Bytes_ e aplicando restrições de permissão (`noexec`).|
|**Princípio do Menor Privilégio**|Auditar o `/etc/sudoers`. Scripts administrativos não devem chamar arquivos que permitem escrita por usuários comuns.|
|**Gestão de Senhas**|Migrar do algoritmo MD5 para hashes modernos e seguros (ex: `bcrypt`, `Argon2`) para armazenamento de senhas.|

_Relatório finalizado com sucesso. O alvo foi totalmente comprometido._