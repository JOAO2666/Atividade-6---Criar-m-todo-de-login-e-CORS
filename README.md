# Atividade 6 - Criar método de login e CORS (SENAI)

Esta atividade consiste na implementação do **método de Login com geração de Token JWT (JSON Web Token)**, configuração da autenticação e autorização **JwtBearer** no pipeline da aplicação (`Program.cs`) e proteção dos endpoints restritos (`PUT` e `DELETE`) com `[Authorize]` no projeto **Exo.WebApi**, utilizando **.NET 6.0**, **Entity Framework Core** e **SQL Server**, com validação das rotas HTTP através do **Insomnia** (ou **Postman**).

---

## 📁 Estrutura do Projeto

```plaintext
├── cria-db.sql                          # Script SQL para criação do banco ExoApi e tabelas Projetos/Usuarios
├── Insomnia_ExoApi_Collection.json      # Coleção pronta para importação no Insomnia (testes com e sem Bearer Token)
├── Exo.WebApi/
│   ├── Contexts/
│   │   └── ExoContext.cs                # Contexto do Entity Framework Core (DbSet<Projeto> e DbSet<Usuario>)
│   ├── Controllers/
│   │   ├── ProjetosController.cs        # Controller REST de Projetos (/api/projetos)
│   │   └── UsuariosController.cs        # Controller REST de Usuários com POST de Login JWT e [Authorize] em PUT/DELETE
│   ├── Models/
│   │   ├── Projeto.cs                   # Entidade Projeto (Id, NomeDoProjeto, Area, Status)
│   │   └── Usuario.cs                   # Entidade Usuario (Id, Email, Senha)
│   ├── Repositories/
│   │   ├── ProjetoRepository.cs         # Camada Repository de Projetos
│   │   └── UsuarioRepository.cs         # Camada Repository de Usuários (inclui método Login(email, senha))
│   ├── Properties/
│   │   └── launchSettings.json          # Configurações de execução e portas (https://localhost:7154)
│   ├── appsettings.json                 # Configurações da aplicação
│   ├── Exo.WebApi.csproj                # Arquivo de projeto .NET 6.0 (com pacote Microsoft.AspNetCore.Authentication.JwtBearer)
│   └── Program.cs                       # Configuração de DI, Autenticação JwtBearer, Swagger e middlewares
└── README.md
```

---

## 🛠️ Tecnologias Utilizadas

- **C# / .NET 6.0 (ou .NET 7.0)**
- **ASP.NET Core Web API**
- **JWT Bearer Authentication (`Microsoft.AspNetCore.Authentication.JwtBearer` 6.0.0)**
- **Entity Framework Core 6.0 (`Microsoft.EntityFrameworkCore.SqlServer`)**
- **SQL Server / SQL Server Management Studio (SSMS)**
- **Insomnia / Postman** (Testes de autenticação e autorização Bearer Token)

---

## 🗄️ 1. Configuração do Banco de Dados (SSMS)

1. Abra o **SQL Server Management Studio (SSMS)**.
2. Abra e execute o arquivo [`cria-db.sql`](./cria-db.sql) para criar o banco **`ExoApi`**, a tabela **`Projetos`** e a tabela **`Usuarios`** com os registros iniciais (`email@sp.br` e `email_dois@sp.br`, ambos com senha `1234`).
3. Em [`Exo.WebApi/Contexts/ExoContext.cs`](./Exo.WebApi/Contexts/ExoContext.cs), ajuste a string de conexão caso sua instância do SQL Server seja diferente de `localhost\SQLEXPRESS01`:
   ```csharp
   optionsBuilder.UseSqlServer("Server=localhost\\SQLEXPRESS01;Database=ExoApi;Trusted_Connection=True;");
   ```

---

## 🔐 2. Implementações Realizadas

### 2.1. Método `Login` no `UsuarioRepository.cs`
Valida as credenciais de `email` e `senha` no banco de dados:
```csharp
public Usuario Login(string email, string senha)
{
    return _context.Usuarios.FirstOrDefault(u => u.Email == email && u.Senha == senha);
}
```

### 2.2. Geração de Token JWT e `[Authorize]` no `UsuariosController.cs`
- **`POST /api/usuarios`**: Recebe `email` e `senha` no corpo da requisição, consulta `_usuarioRepository.Login(usuario.Email, usuario.Senha)` e:
  - Caso não encontre o usuário, retorna **`404 NotFound("E-mail ou senha inválidos!")`**.
  - Caso encontre, gera um **`JwtSecurityToken`** assinado com `HmacSha256` (`exoapi-chave-autenticacao`), emissor/destinatário `exoapi.webapi` e expiração de 30 minutos, retornando **`200 OK`** com `{ "token": "..." }`.
- **`PUT /api/usuarios/{id}`** e **`DELETE /api/usuarios/{id}`**: Protegidos com o atributo **`[Authorize]`**, exigindo envio de um token JWT válido via cabeçalho `Authorization: Bearer <token>`.

### 2.3. Configuração de Autenticação JWT no `Program.cs`
- Registro do esquema `"JwtBearer"` via `builder.Services.AddAuthentication(...).AddJwtBearer("JwtBearer", ...)`, validando `Issuer`, `Audience`, `Lifetime`, `IssuerSigningKey` (`exoapi-chave-autenticacao`) e `ClockSkew` de 30 minutos.
- Habilitação dos middlewares `app.UseAuthentication();` e `app.UseAuthorization();` entre `app.UseRouting();` e `app.UseEndpoints(...)`.

---

## ▶️ 3. Como Executar a API

No terminal, dentro da pasta `Exo.WebApi`:

```bash
dotnet restore
dotnet build
dotnet run
```

A aplicação ficará disponível em:
- **HTTPS**: `https://localhost:7154`
- **HTTP**: `http://localhost:5038`

---

## 🧪 4. Roteiro de Testes no Insomnia

Você pode importar diretamente o arquivo [`Insomnia_ExoApi_Collection.json`](./Insomnia_ExoApi_Collection.json) no Insomnia ou seguir o passo a passo abaixo:

### Passo 1 — Testar Rotas Protegidas SEM Token (`401 Unauthorized`)
- **`PUT`** `https://localhost:7154/api/usuarios/2` (sem token) → Retorna **`401 Unauthorized`**.
- **`DELETE`** `https://localhost:7154/api/usuarios/2` (sem token) → Retorna **`401 Unauthorized`**.

### Passo 2 — Gerar Token de Acesso via Login (`POST /api/usuarios`)
- **Método**: `POST`
- **URL**: `https://localhost:7154/api/usuarios`
- **Body (JSON)**:
  ```json
  {
    "email": "email@sp.br",
    "senha": "1234"
  }
  ```
- **Resposta esperada**: **`200 OK`** contendo `{ "token": "eyJhbGciOiJIUzI1NiIs..." }`. Copie o valor do token.

### Passo 3 — Atualizar Usuário COM Token Bearer (`PUT /api/usuarios/2`)
- **Método**: `PUT`
- **URL**: `https://localhost:7154/api/usuarios/2`
- **Auth**: `Bearer Token` → Cole o token copiado no Passo 2.
- **Body (JSON)**:
  ```json
  {
    "email": "email_novo@sp.br",
    "senha": "aaaaa1234"
  }
  ```
- **Resposta esperada**: **`204 No Content`**.
- Confirme com **`GET`** `https://localhost:7154/api/usuarios/` (**`200 OK`**).

### Passo 4 — Deletar Usuário COM Token Bearer (`DELETE /api/usuarios/2`)
- **Método**: `DELETE`
- **URL**: `https://localhost:7154/api/usuarios/2`
- **Auth**: `Bearer Token` → Com o mesmo token ativo.
- **Resposta esperada**: **`204 No Content`**.
- Confirme com **`GET`** `https://localhost:7154/api/usuarios/` (**`200 OK`**) que o usuário de `id: 2` foi removido.
