---
id: AUTH_API_REFERENCE
title: "Auth - Referência da API"
module: offpix/auth
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - identity
  - api-reference
---

# Referência da API de Autenticação

Referência completa de endpoints para o módulo de autenticação do Serviço de Identidade W3Block (PixwayID).

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://pixwayid.w3block.io` |
| Swagger | https://pixwayid.w3block.io/docs/ |

> **Nota:** Um ambiente de staging também está disponível para testes. Entre em contato com seu representante W3Block para obter as URLs de staging.

## Autenticação

Endpoints marcados como **Auth: Bearer** requerem:

```
Authorization: Bearer {accessToken}
```

Endpoints marcados como **Auth: None** são públicos. No entanto, se um header `Authorization` estiver presente em um endpoint público, o token ainda será validado — permitindo autenticação opcional.

## Limitação de Taxa

Todos os endpoints possuem limitação de taxa. Padrão: 120 requisições/minuto. Endpoints com limites mais restritos são indicados individualmente.

---

## Enums

### UserRoleEnum

| Valor | Descrição |
|-------|-----------|
| `superAdmin` | Super administrador em nível de plataforma |
| `admin` | Administrador do tenant |
| `operator` | Operador do tenant |
| `user` | Usuário regular |
| `loyaltyOperator` | Operador do programa de fidelidade |
| `commerceOrderReceiver` | Receptor de pedidos do commerce |
| `kycApprover` | Operador de aprovação KYC |
| `keyErc20Receiver` | Receptor de tokens ERC-20 |

### TenantRoleEnum

| Valor | Descrição |
|-------|-----------|
| `application` | Acesso em nível de aplicação |
| `administrator` | Papel de admin do tenant |
| `integration` | Acesso de integração/chave API |
| `contextApplication` | Aplicação com escopo de contexto |

### VerificationType

| Valor | Descrição |
|-------|-----------|
| `numeric` | Código numérico de 6 dígitos enviado por email |
| `invisible` | Verificação baseada em token via link por email |

### UserCodeType

| Valor | Descrição |
|-------|-----------|
| `signin` | Código OTP para login |
| `magicSignInToken` | Token mágico UUID para login |
| `loyalty` | Código do programa de fidelidade |
| `phone_verification` | Verificação de número de telefone |

### JwtType

| Valor | Descrição |
|-------|-----------|
| `user` | JWT de usuário (emitido no login do usuário) |
| `tenant` | JWT de tenant (emitido no login por chave API) |

---

## Schemas

### SignInResponse

Retornado por todos os endpoints de login em caso de sucesso.

```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6InJlZnJlc2gifQ...",
  "data": {
    "sub": "uuid-user-id",
    "iss": "uuid-tenant-id",
    "aud": ["uuid-tenant-id"],
    "exp": 1711900000,
    "iat": 1711899100,
    "type": "user",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "verified": true,
    "emailVerified": true,
    "phoneVerified": false
  },
  "profile": {
    "id": "uuid-user-id",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "verified": true
  }
}
```

### UserJwtPayload (`token` decodificado)

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `sub` | UUID | ID do usuário |
| `iss` | UUID | ID do tenant (emissor) |
| `aud` | UUID[] | Audiência (IDs de tenant) |
| `exp` | number | Expiração (unix timestamp) |
| `iat` | number | Emitido em (unix timestamp) |
| `type` | `"user"` | Tipo do token |
| `tenantId` | UUID | ID do tenant |
| `email` | string | Email do usuário |
| `name` | string? | Nome do usuário |
| `roles` | UserRoleEnum[] | Papéis do usuário |
| `verified` | boolean | Se o usuário está verificado |
| `emailVerified` | boolean | Se o email está verificado |
| `phoneVerified` | boolean | Se o telefone está verificado |

### TenantJwtPayload (`token` decodificado para login de tenant)

Mesmo que UserJwtPayload, mas com `type: "tenant"` e `roles: TenantRoleEnum[]`.

---

## Endpoints

### Login

#### POST `/auth/signin`

Login de usuário baseado em senha.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 60/min |
| **Content-Type** | application/json |

**Requisição Mínima:**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd"
}
```

**Requisição Completa:**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "tenantId": "uuid-tenant-id"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `email` | string | Sim | Email válido, máx 50 chars | Email do usuário (case-insensitive) |
| `password` | string | Sim | 8-32 chars | Senha do usuário |
| `tenantId` | UUID | Não | UUID válido | Escopo do tenant. Se omitido e o email existir em múltiplos tenants, retorna 409 |

**Resposta (200):** `SignInResponse`

**Erros:**

| Status | Causa | Resposta |
|--------|-------|----------|
| 401 | Credenciais inválidas | `{ "statusCode": 401, "message": "Unauthorized" }` |
| 409 | Email existe em múltiplos tenants, sem `tenantId` fornecido | `{ "statusCode": 409, "message": "Conflict", "tenants": [{ "id": "uuid", "name": "Tenant Name" }] }` |

---

#### POST `/auth/signin/code`

Login baseado em código OTP. Requer chamada prévia a `/auth/signin/request-code`.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "email": "user@example.com",
  "code": "123456",
  "tenantId": "uuid-tenant-id"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `email` | string | Sim | Email válido, máx 50 chars | Email do usuário |
| `code` | string | Sim | Exatamente 6 caracteres | Código OTP recebido por email |
| `tenantId` | UUID | Não | UUID válido | Escopo do tenant |

**Resposta (200):** `SignInResponse`

**Efeito colateral:** Se o email ainda não estiver verificado, ele é automaticamente marcado como verificado em caso de login por código bem-sucedido.

---

#### POST `/auth/signin/request-code`

Envia um código OTP de 6 dígitos para o email do usuário para login baseado em código.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `email` | string | Sim | Email válido | Email do usuário |
| `tenantId` | UUID | **Sim** | UUID válido | Escopo do tenant (obrigatório diferente de outros endpoints) |

**Resposta (201):** Corpo vazio (email enviado)

---

#### POST `/auth/signin/magic-token`

Login usando um token mágico UUID (gerado por admin/integração).

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "token": "uuid-magic-token",
  "tenantId": "uuid-tenant-id"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `token` | UUID | Sim | UUID válido | Token mágico de login |
| `tenantId` | UUID | Sim | UUID válido | Escopo do tenant |

**Resposta (200):** `SignInResponse`

---

#### POST `/auth/signin/generate-magic-token`

Gera um token mágico de login para um usuário. Apenas admin/integração.

| | |
|---|---|
| **Auth** | Bearer (SuperAdmin, Integration) |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "expirationInMinutes": 30
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `email` | string | Sim | Email válido | Email do usuário alvo |
| `tenantId` | UUID | Sim | UUID válido | Escopo do tenant |
| `expirationInMinutes` | number | Não | Inteiro, mín 1 | TTL do token (padrão: 30 min) |

**Resposta (201):**

```json
{
  "token": "uuid-magic-token",
  "expiresAt": "2026-03-31T12:30:00.000Z"
}
```

---

#### POST `/auth/signin/tenant`

Login usando chave e segredo da API do tenant. Retorna um **JWT de tenant** (não um JWT de usuário).

| | |
|---|---|
| **Auth** | None |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "key": "api-key-string",
  "secret": "api-secret-string",
  "tenantId": "uuid-tenant-id"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `key` | string | Sim | Chave API do tenant |
| `secret` | string | Sim | Segredo API do tenant |
| `tenantId` | UUID | Sim | ID do tenant |

**Resposta (200):** `SignInResponse` (com `data.type = "tenant"` e `data.roles = TenantRoleEnum[]`)

---

### Google OAuth

#### GET `/auth/{tenantId}/signin/google`

Redireciona para a tela de consentimento do Google OAuth.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |

**Parâmetros de Caminho:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `tenantId` | UUID | ID do tenant (determina a configuração do cliente OAuth) |

**Resposta (302):** Redirecionamento para `accounts.google.com`

---

#### GET `/auth/{tenantId}/signin/google/code`

Troca um código de autorização do Google por um JWT W3Block.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `code` | string | Sim | Código de autorização do Google |
| `referrer` | string | Não | Código de indicação |

**Resposta (200):** `SignInResponse` (com `isNewUser: boolean` adicional)

---

#### POST `/auth/signin/google`

Login direto usando um token de ID do Google JWT (para apps mobile/SPA).

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id",
  "referrer": "REFERRAL_CODE"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `credential` | string | Sim | Token de ID do Google (JWT) |
| `tenantId` | UUID | Sim | ID do tenant |
| `referrer` | string | Não | Código de indicação |

**Resposta (200):** `SignInResponse` (com `isNewUser: boolean`)

---

### Apple OAuth

#### GET `/auth/{tenantId}/signin/apple`

Redireciona para a tela de consentimento do Apple OAuth.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |

**Parâmetros de Caminho:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `tenantId` | UUID | ID do tenant |

**Resposta (302):** Redirecionamento para `appleid.apple.com`

---

#### POST `/auth/{tenantId}/signin/apple/code`

Troca um código de autorização da Apple por um JWT W3Block.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |

**Resposta (200):** `SignInResponse` (com `isNewUser: boolean`, `isPrivateEmail: boolean`)

---

#### POST `/auth/signin/apple`

Login direto usando um token de ID da Apple (para apps mobile/SPA).

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id"
}
```

Aceita tanto o campo `credential` quanto `id_token` (ambos são JWTs de token de ID da Apple).

**Resposta (200):** `SignInResponse` (com `isNewUser: boolean`, `isPrivateEmail: boolean`)

---

### Cadastro

#### POST `/auth/signup`

Registra um novo usuário dentro de um tenant existente.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 60/min |
| **Content-Type** | application/json |

**Requisição Mínima:**

```json
{
  "tenantId": "uuid-tenant-id",
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd"
}
```

**Requisição Completa (exemplo de produção):**

```json
{
  "tenantId": "uuid-tenant-id",
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd",
  "name": "John Doe",
  "phone": "+5511999998888",
  "i18nLocale": "pt-BR",
  "callbackUrl": "https://myapp.com/auth/verify",
  "verificationType": "numeric",
  "referrer": "REFERRAL_CODE",
  "utmParams": {
    "utm_source": "google",
    "utm_medium": "cpc",
    "utm_campaign": "launch"
  }
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `tenantId` | UUID | Sim | UUID válido | Tenant de destino |
| `email` | string | Sim | Email válido, máx 50 chars | Email do usuário (convertido para minúsculas) |
| `password` | string | Condicional | 8-32 chars, maiúsculas/minúsculas + dígito | Obrigatório a menos que o tenant seja passwordless |
| `confirmation` | string | Condicional | Deve coincidir com `password` | Confirmação da senha |
| `name` | string | Não | 1-250 chars | Nome de exibição do usuário |
| `phone` | string | Não | Número de telefone válido | Telefone do usuário |
| `i18nLocale` | enum | Não | `pt-BR` ou `en` | Idioma do email (padrão: `pt-BR`) |
| `callbackUrl` | URL | Não | URL válida, host permitido | URL de redirecionamento para verificação de email |
| `verificationType` | enum | Não | `numeric` ou `invisible` | Método de verificação (padrão: `invisible`) |
| `referrer` | string | Não | — | Código de indicação |
| `utmParams` | object | Não | — | Parâmetros de rastreamento UTM |

**Resposta (201):** `SignInResponse`

**Notas:**
- Se o tenant tem o modo `passwordless` habilitado, `password` e `confirmation` são opcionais — uma senha aleatória é gerada no servidor.
- Se `verificationType` é `numeric`, um código de 6 dígitos é enviado em vez de um link.
- Se a configuração do tenant exige um referenciador, o campo `referrer` se torna obrigatório.

---

#### POST `/auth/signup/tenant`

Etapa 1 da criação de tenant. Cria um usuário inativo no tenant do sistema.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "email": "owner@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd",
  "verificationType": "numeric"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `email` | string | Sim | Email do proprietário |
| `password` | string | Sim | 8-32 chars, maiúsculas/minúsculas + dígito |
| `confirmation` | string | Sim | Deve coincidir com a senha |
| `verificationType` | enum | Não | Padrão: `invisible` |
| `name` | string | Não | Nome do proprietário |
| `phone` | string | Não | Número de telefone |

**Resposta (201):** `SignInResponse` (JWT temporário — o usuário está inativo até a etapa 2)

---

#### PATCH `/auth/signup/tenant/finish`

Etapa 2 da criação de tenant. Cria o tenant e ativa o usuário.

| | |
|---|---|
| **Auth** | Bearer (token temporário da etapa 1) |
| **Limite de Taxa** | 10/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "tenantName": "My Company",
  "ownerName": "John Doe",
  "phone": "+5511999998888"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `tenantName` | string | Sim | Mín 3 chars | Nome da empresa/tenant |
| `ownerName` | string | Sim | Mín 5 chars | Nome completo do proprietário |
| `phone` | string | Sim | Telefone válido | Telefone do proprietário |

**Resposta (200):**

```json
{
  "tenant": {
    "id": "uuid-new-tenant-id",
    "name": "My Company"
  },
  "tenantSignIn": { /* SignInResponse - novo JWT com escopo no novo tenant */ }
}
```

**Efeitos colaterais:**
- Cria uma nova entidade `Tenant`
- Migra o usuário do tenant do sistema para o novo tenant
- Define os papéis do usuário como `[admin, user]`
- Marca o usuário como ativo
- Despacha jobs de configuração do tenant (criação de cliente, configuração padrão)

---

### Verificação de Email

#### GET `/auth/verify-sign-up`

Verifica o email de um usuário usando um token de verificação.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 120/min |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `email` | string | Sim | Email do usuário |
| `token` | string | Sim | Token de verificação (do link de email ou código de 6 dígitos) |

**Resposta (200):**

```json
{
  "verified": true,
  "emailVerified": true,
  "phoneVerified": false
}
```

---

#### POST `/auth/request-confirmation-email`

Reenvia o email/código de verificação de email.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 60/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "callbackUrl": "https://myapp.com/auth/verify",
  "verificationType": "numeric"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `email` | string | Sim | Email do usuário |
| `tenantId` | UUID | Não | Escopo do tenant |
| `callbackUrl` | URL | Não | URL de redirecionamento para link de email |
| `verificationType` | enum | Não | `numeric` ou `invisible` |

**Resposta (201):** Corpo vazio (email enviado)

---

### Redefinição de Senha

#### POST `/auth/request-password-reset`

Envia um email de redefinição de senha para o usuário.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 60/min |
| **Content-Type** | application/json |

**Requisição Mínima:**

```json
{
  "email": "user@example.com"
}
```

**Requisição Completa:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "callbackUrl": "https://myapp.com/auth/reset-password",
  "verificationType": "invisible"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `email` | string | Sim | Email do usuário |
| `tenantId` | UUID | Não | Escopo do tenant |
| `callbackUrl` | URL | Não | URL da página de redefinição de senha |
| `verificationType` | enum | Não | Padrão: `invisible` |

**Resposta (201):** Corpo vazio (email enviado)

**Nota:** O link do email contém `?email={email}&token={token};{expireTimestamp}`. Sua página de redefinição de senha deve analisar esses parâmetros.

---

#### POST `/auth/reset-password`

Consome o token de redefinição e define uma nova senha. Retorna um JWT (login automático).

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 60/min |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "email": "user@example.com",
  "token": "verification-token-string",
  "password": "NewP@ssw0rd",
  "confirmation": "NewP@ssw0rd"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `email` | string | Sim | Email válido | Email do usuário |
| `token` | string | Sim | Token de verificação | Token do email de redefinição |
| `password` | string | Sim | 8-32 chars, maiúsculas/minúsculas + dígito | Nova senha |
| `confirmation` | string | Sim | Deve coincidir com `password` | Confirmação da senha |

**Resposta (200):** `SignInResponse` (o usuário é automaticamente logado)

**Efeito colateral:** O email é marcado como verificado caso ainda não estivesse.

---

### Gerenciamento de Tokens

#### POST `/auth/refresh-token`

Troca um refresh token por um novo par de access + refresh token.

| | |
|---|---|
| **Auth** | None (refresh token no corpo) |
| **Content-Type** | application/json |

**Requisição:**

```json
{
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6InJlZnJlc2gifQ..."
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `refreshToken` | string | Sim | Refresh token JWT válido |

**Resposta (200):**

```json
{
  "token": "new-access-token...",
  "refreshToken": "new-refresh-token..."
}
```

**Notas:**
- O refresh token antigo é colocado em lista negra com um atraso de 30 segundos (permitindo que requisições em andamento sejam concluídas).
- Refresh tokens possuem `typ: "refresh"` no header do JWT.
- O refresh token contém um `tokenHash` (SHA-256 do access token com o qual foi emitido).

---

#### POST `/auth/logout`

Coloca o access token atual em lista negra.

| | |
|---|---|
| **Auth** | Bearer |

**Requisição:** Corpo vazio. O access token do header `Authorization` é colocado em lista negra.

**Resposta (201):** Corpo vazio

**Notas:**
- O token é colocado em lista negra através de seu hash SHA-256 no cache.
- TTL da lista negra = segundos restantes até a expiração natural do token.

---

### Consulta de Tenant

#### GET `/auth/user-tenants`

Lista os tenants onde um usuário possui conta. Útil para resolução de email multi-tenant.

| | |
|---|---|
| **Auth** | None |
| **Limite de Taxa** | 60/min |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `email` | string | Sim | Email do usuário |

**Resposta (200):**

```json
{
  "email": "user@example.com",
  "tenants": [
    { "id": "uuid-tenant-1", "name": "Company A" },
    { "id": "uuid-tenant-2", "name": "Company B" }
  ]
}
```

---

### JWKS

#### GET `/auth/jwks.json`

Retorna a chave pública RSA como JWKS para verificação de tokens.

| | |
|---|---|
| **Auth** | None |
| **Cache** | CDN com cache de 10 minutos |

**Resposta (200):**

```json
{
  "keys": [
    {
      "kty": "RSA",
      "n": "...",
      "e": "AQAB",
      "alg": "RS256",
      "use": "sig"
    }
  ]
}
```

**Caso de uso:** Serviços externos podem verificar JWTs da W3Block buscando este endpoint e validando a assinatura RS256.
