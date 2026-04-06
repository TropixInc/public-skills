---
id: FLOW_AUTH_SIGNIN
title: "Auth - Fluxos de Login"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - signin
  - login
depends_on:
  - AUTH_API_REFERENCE
---

# Fluxos de Login

## Visão Geral

O W3Block suporta 5 métodos de login: senha, código OTP, magic token, chave de API do tenant e Google/Apple OAuth. Todos os métodos retornam a mesma `SignInResponse` com tokens de acesso e de renovação. Este documento cobre os fluxos de senha, código e magic token. Os fluxos OAuth são documentados separadamente.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Conta de usuário | Usuário registrado em um tenant | [Fluxo de Registro](./FLOW_AUTH_SIGNUP.md) |
| `tenantId` | UUID do tenant de destino | Consulta de tenant ou configuração do ambiente |
| URL Base | URL do serviço de identidade | `https://pixwayid.w3block.io` (prod). Um ambiente de staging também está disponível — entre em contato com a W3Block para detalhes. |

## Entidades e Relacionamentos

```
Tenant (1) ──→ (N) User ──→ (N) UserCode (OTP/magic tokens)
                     │
                     └──→ (N) AccessLog (audit trail)
```

Um único e-mail pode existir em múltiplos tenants. Quando `tenantId` não é fornecido, a API resolve a conta correta — ou retorna um conflito 409 se existirem múltiplas.

---

## Fluxo A: Login por Senha

O fluxo mais comum. Opcionalmente precedido por uma consulta de tenant.

### Etapa 1 (Opcional): Resolver Tenant

Se o usuário pode ter contas em múltiplos tenants, resolva primeiro:

```
GET /auth/user-tenants?email=user@example.com
```

**Resposta:**

```json
{
  "email": "user@example.com",
  "tenants": [
    { "id": "uuid-tenant-1", "name": "Company A" },
    { "id": "uuid-tenant-2", "name": "Company B" }
  ]
}
```

- Se **1 tenant**: use seu `id` como `tenantId` automaticamente.
- Se **2+ tenants**: apresente uma UI de seleção, depois use o `id` escolhido.
- Se **0 tenants / 404**: e-mail não registrado.

### Etapa 2: Fazer Login

```
POST /auth/signin
Content-Type: application/json
```

**Requisição Mínima:**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd"
}
```

**Requisição Completa (com tenant):**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "tenantId": "uuid-tenant-id"
}
```

**Resposta (200):**

```json
{
  "token": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "data": {
    "sub": "uuid-user-id",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "verified": true,
    "emailVerified": true,
    "phoneVerified": false,
    "type": "user"
  }
}
```

### Etapa 3: Armazenar Tokens

Armazene `token` (acesso) e `refreshToken` de forma segura. O token de acesso é usado como `Authorization: Bearer {token}` para todas as requisições autenticadas.

**Detalhes do token:**
- Algoritmo: RS256 (assimétrico — verifique via `/auth/jwks.json`)
- Expiração do token de acesso: configurada por implantação (tipicamente 15 minutos)
- Expiração do token de renovação: configurada por implantação (vida mais longa)
- O token de renovação possui `typ: "refresh"` no cabeçalho JWT

---

## Fluxo B: Login por Código OTP

Fluxo em duas etapas: solicitar um código de 6 dígitos, depois fazer login com ele.

### Etapa 1: Solicitar Código

```
POST /auth/signin/request-code
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id"
}
```

> **Nota:** `tenantId` é **obrigatório** para este endpoint, diferente do login por senha.

**Resposta (201):** Corpo vazio. Um código de 6 dígitos é enviado para o e-mail do usuário.

### Etapa 2: Fazer Login com Código

```
POST /auth/signin/code
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "code": "123456",
  "tenantId": "uuid-tenant-id"
}
```

**Resposta (200):** `SignInResponse` (igual ao login por senha)

**Efeito colateral:** Se o e-mail do usuário ainda não estava verificado, ele é automaticamente marcado como verificado ao fazer login com código com sucesso.

---

## Fluxo C: Login por Magic Token

Usado para login iniciado por admin (ex.: impersonação, onboarding de integração). Requer um JWT de admin ou integração para gerar o token.

### Etapa 1: Gerar Magic Token (Admin)

```
POST /auth/signin/generate-magic-token
Authorization: Bearer {adminToken}
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "expirationInMinutes": 60
}
```

**Resposta (201):**

```json
{
  "token": "uuid-magic-token",
  "expiresAt": "2026-03-31T13:00:00.000Z"
}
```

### Etapa 2: Fazer Login com Magic Token

```
POST /auth/signin/magic-token
Content-Type: application/json
```

```json
{
  "token": "uuid-magic-token",
  "tenantId": "uuid-tenant-id"
}
```

**Resposta (200):** `SignInResponse`

---

## Fluxo D: Login por Chave de API do Tenant

Para integrações servidor-a-servidor. Retorna um **JWT de tenant** com roles `TenantRoleEnum` em vez de um JWT de usuário.

```
POST /auth/signin/tenant
Content-Type: application/json
```

```json
{
  "key": "tenant-api-key",
  "secret": "tenant-api-secret",
  "tenantId": "uuid-tenant-id"
}
```

**Resposta (200):** `SignInResponse` com `data.type = "tenant"` e `data.roles` contendo valores de `TenantRoleEnum`.

---

## Renovação de Token

Quando o token de acesso se aproxima da expiração, troque o token de renovação por um novo par:

```
POST /auth/refresh-token
Content-Type: application/json
```

```json
{
  "refreshToken": "eyJhbG..."
}
```

**Resposta (200):**

```json
{
  "token": "new-access-token...",
  "refreshToken": "new-refresh-token..."
}
```

**Importante:** O token de renovação antigo é colocado em lista negra com um atraso de 30 segundos. Implemente a renovação de token de forma proativa (ex.: em 50% do tempo de vida do token de acesso) em vez de esperar por erros 401.

---

## Logout

Coloca o token de acesso atual em lista negra:

```
POST /auth/logout
Authorization: Bearer {accessToken}
```

**Resposta (201):** Corpo vazio.

Após o logout, o token de acesso é imediatamente inválido. Qualquer requisição subsequente usando-o receberá um 401.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 401 | Não autorizado | Credenciais inválidas, token expirado ou token em lista negra | Verifique as credenciais ou renove o token |
| 404 | Não encontrado | E-mail não registrado em nenhum tenant | Direcione o usuário para o registro |
| 409 | Conflito | E-mail existe em múltiplos tenants, sem `tenantId` | Use `/auth/user-tenants` para resolver, depois inclua `tenantId` |
| 429 | Muitas requisições | Limite de taxa excedido | Aguarde e tente novamente (verifique o cabeçalho `Retry-After`) |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | 409 ao fazer login sem `tenantId` | Sempre chame `/auth/user-tenants` primeiro se não souber o tenant. Se houver exatamente um resultado, use-o automaticamente |
| 2 | Renovação de token falha silenciosamente | Monitore o `RefreshAccessTokenError` na resposta de renovação. Force logout e reautentique |
| 3 | Código OTP expirado | Os códigos têm TTL curto. Implemente um botão de reenviar com tempo de espera (recomendado: 60 segundos) |
| 4 | `tenantId` obrigatório para `/request-code` mas opcional em outros | Isso é intencional — códigos OTP são vinculados ao tenant. Sempre inclua `tenantId` para fluxos baseados em código |
| 5 | Validação de senha difere por tenant | Tenants com modo `passwordless` não aplicam regras de senha. Verifique a configuração do tenant ao construir UIs multi-tenant |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|----------------|----------|
| Registro | Criar uma conta de usuário antes de fazer login | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) |
| Redefinição de Senha | Redefinir senha antes de fazer login | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) |
| Google/Apple OAuth | Métodos alternativos de login | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) |
| Renovação de Token | Manter sessão após o login | Este documento (seção Renovação de Token) |
