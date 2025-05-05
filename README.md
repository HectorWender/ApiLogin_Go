# API de Autenticação em Go

Esta API fornece endpoints para autenticação (login/logout) de usuários com diferentes papéis (funcionário, cliente, patrocinador), utilizando JSON Web Tokens (JWT) para gerenciamento de sessão e seguindo princípios de Clean Architecture.

## Funcionalidades

*   **Login:** Autentica usuários com base em nome de usuário e senha, retornando um JWT em caso de sucesso.
*   **Logout:** Endpoint para invalidar sessões (implementação básica, depende da estratégia de invalidação de token).
*   **Autenticação JWT:** Utiliza JWT para proteger endpoints e gerenciar sessões.
*   **Controle de Roles:** Permite restringir o acesso a endpoints específicos com base no papel do usuário (employee, client, sponsor) embutido no JWT.
*   **Clean Architecture:** Estrutura organizada em camadas (domain, usecase, infrastructure, api) para melhor manutenibilidade e testabilidade.
*   **Hashing de Senha:** Armazena senhas de forma segura usando bcrypt.
*   **Repositório em Memória:** Inclui uma implementação de repositório de usuários em memória para facilitar testes (facilmente substituível).
*   **Configuração via Variáveis de Ambiente:** Carrega configurações importantes (chave secreta JWT, porta do servidor) de variáveis de ambiente (com fallback para `.env` local).
*   **Logging Estruturado:** Configuração básica de log para incluir data, hora e arquivo de origem.

## Estrutura do Projeto

O projeto segue a estrutura da Clean Architecture:

```
/auth-api
|-- cmd/server/main.go       # Ponto de entrada, inicialização e injeção de dependências
|-- internal/
|   |-- domain/              # Entidades e regras de negócio (User, Role)
|   |-- usecase/             # Lógica da aplicação (AuthUseCase, TokenService interfaces)
|   |   |-- auth/            # Caso de uso de autenticação
|   |   `-- token/           # Interface e implementação JWT
|   |-- infrastructure/      # Detalhes externos (BD, config, hash)
|   |   |-- persistence/     # Implementações de repositório (memory)
|   |   |-- config/          # Carregamento de configuração
|   |   `-- hash/            # Hashing de senha (bcrypt)
|   `-- api/                 # Camada de apresentação (HTTP)
|       |-- handler/         # Handlers HTTP (AuthHandler)
|       |-- middleware/      # Middlewares (AuthMiddleware)
|       `-- router/          # Configuração de rotas
|-- go.mod                   # Dependências do Go
|-- go.sum
|-- auth-server              # Binário compilado
`-- README.md                # Este arquivo
```

## Configuração

A API utiliza variáveis de ambiente para configuração. Crie um arquivo `.env` na raiz do projeto para desenvolvimento local (opcional, mas recomendado):

```dotenv
# .env

# Chave secreta para assinar os JWTs (MUITO IMPORTANTE: Use uma chave segura e longa em produção)
JWT_SECRET="sua-chave-secreta-super-segura-aqui"

# Porta em que o servidor irá rodar
SERVER_PORT=8080

# (Opcional) Duração da validade do JWT em horas
JWT_EXPIRATION_HOURS=1

# (Opcional) Emissor (issuer) do JWT
JWT_ISSUER="sua-api"
```

**Importante:** Nunca comite chaves secretas diretamente no código ou em arquivos `.env` públicos. Em produção, utilize mecanismos seguros de gerenciamento de segredos.

## Como Executar

1.  **Clone o repositório:**
    ```bash
    # (Se aplicável, após receber o código)
    # git clone <url_do_repositorio>
    # cd auth-api
    ```

2.  **Instale as dependências:**
    ```bash
    go mod tidy
    ```

3.  **Compile o projeto:**
    ```bash
    go build -o auth-server ./cmd/server/main.go
    ```

4.  **Configure as variáveis de ambiente:**
    *   Crie o arquivo `.env` conforme a seção "Configuração".
    *   Ou exporte as variáveis diretamente no seu terminal:
        ```bash
        export JWT_SECRET="sua-chave-secreta-super-segura-aqui"
        export SERVER_PORT=8080
        # export JWT_EXPIRATION_HOURS=1
        # export JWT_ISSUER="sua-api"
        ```

5.  **Execute o servidor:**
    ```bash
    ./auth-server
    ```

O servidor estará rodando em `http://localhost:8080` (ou na porta configurada).

## Endpoints da API

*   **`POST /login`** (Público)
    *   Autentica um usuário.
    *   **Corpo da Requisição (JSON):**
        ```json
        {
          "username": "employee1",
          "password": "password123"
        }
        ```
        *(Usuários de teste pré-configurados no repositório em memória: `employee1`/`password123`, `client1`/`clientpass`, `sponsor1`/`sponsorpass`)*
    *   **Resposta de Sucesso (200 OK - JSON):**
        ```json
        {
          "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
        }
        ```
    *   **Resposta de Erro (401 Unauthorized):** Credenciais inválidas.

*   **`POST /logout`** (Protegido - Qualquer Role)
    *   Endpoint para indicar logout. A invalidação real do token depende da estratégia (não implementada com blacklist nesta versão).
    *   **Cabeçalho Obrigatório:** `Authorization: Bearer <seu_jwt_token>`
    *   **Resposta de Sucesso (200 OK - JSON):**
        ```json
        {
          "message": "Logout successful (placeholder)"
        }
        ```
    *   **Resposta de Erro (401 Unauthorized):** Token ausente, inválido ou expirado.

*   **`GET /admin/dashboard`** (Protegido - Role: `employee`)
    *   Endpoint de exemplo acessível apenas por funcionários.
    *   **Cabeçalho Obrigatório:** `Authorization: Bearer <seu_jwt_token>`
    *   **Resposta de Sucesso (200 OK):** `Welcome, Employee!`
    *   **Resposta de Erro (401 Unauthorized):** Token ausente, inválido ou expirado.
    *   **Resposta de Erro (403 Forbidden):** Token válido, mas a role não é `employee`.

*   **`GET /profile`** (Protegido - Role: `client` ou `sponsor`)
    *   Endpoint de exemplo acessível por clientes ou patrocinadores.
    *   **Cabeçalho Obrigatório:** `Authorization: Bearer <seu_jwt_token>`
    *   **Resposta de Sucesso (200 OK):** `Welcome, client!` ou `Welcome, sponsor!`
    *   **Resposta de Erro (401 Unauthorized):** Token ausente, inválido ou expirado.
    *   **Resposta de Erro (403 Forbidden):** Token válido, mas a role não é `client` nem `sponsor`.

## Próximos Passos / Melhorias

*   Implementar um repositório de banco de dados real (PostgreSQL, MongoDB).
*   Adicionar Refresh Tokens para melhorar a experiência do usuário e a segurança.
*   Implementar uma estratégia de invalidação de token para logout (ex: blacklist em Redis).
*   Adicionar validação de entrada mais robusta para as requisições.
*   Escrever testes unitários e de integração.
*   Melhorar o tratamento de erros e os códigos de status HTTP.
*   Utilizar um framework web mais robusto como Gin ou Echo, se necessário.

