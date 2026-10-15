Registrei aqui todas as etapas e descobertas da resolução deste desafio CTF (_CyberHeroes_ do TryHackMe):

## Relatório de Atividades: Desafio CyberHeroes

### 1. Enumeração e Reconhecimento

- **Identificação do Alvo:** Mapeamento do IP `10.66.182.58` e verificação de que se tratava de uma página web com formulário de login utilizando o método HTTP POST.
    
- **Varredura de Diretórios:** Tentativa de enumeração com _Gobuster_ e análise de vulnerabilidades com _Nikto_.
    
- **Descoberta de Directory Listing:** Identificação de que o servidor permitia a listagem de diretórios na pasta `/assets/vendor/`, embora os arquivos presentes fossem apenas bibliotecas padrão de front-end (Bootstrap, etc.).
    

### 2. Análise Estática e Inspeção de Código

- **Inspeção do Front-End:** Verificação do código-fonte da página de login (`login.html`) usando as ferramentas de desenvolvedor (F12) e visualização do código HTML/CSS e das funções JavaScript carregadas.
    
- **Descoberta da Lógica de Validação:** Encontro da função JavaScript `authenticate()`, que realizava a verificação das credenciais diretamente no navegador do cliente, em vez de enviar uma requisição assíncrona antes da hora.
    

### 3. Engenharia Reversa e Obfuscação

- **Análise da Condição:** O código continha a seguinte validação:
    
    `if (a.value=="h3ck3rBoi" & b.value==RevereString("54321@terceSrepuS"))`
    
- **Desofuscação:** Compreensão de que a senha estava invertida pela função `RevereString`. Ao reverter a string `"54321@terceSrepuS"`, cheguei à senha real: **`SuperSecret@12345`**.
    
- **Propósito Educacional:** Discussão sobre como técnicas de ofuscação de código no front-end são utilizadas em CTFs para introduzir conceitos de análise de código e engenharia reversa para iniciantes.
    

### Credenciais Finais Mapeadas

- **Usuário:** `h3ck3rBoi`
    
- **Senha:** `SuperSecret@12345`
    

Basta inserir esses dados na página de login para que a requisição seja validada com sucesso e a _flag_ seja exibida na tela!