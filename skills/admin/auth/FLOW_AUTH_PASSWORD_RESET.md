---
id: FLOW_AUTH_PASSWORD_RESET
title: "Auth - Fluxo de Redefinição de Senha"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - password
  - reset
depends_on:
  - AUTH_API_REFERENCE
  - FLOW_AUTH_SIGNIN
---

# Fluxo de Redefinição de Senha

## Visão Geral

Fluxo em duas etapas que envia um e-mail de redefinição com um token de tempo limitado e, em seguida, permite que o usuário defina uma nova senha e faça login automaticamente. O usuário não precisa estar autenticado.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| E-mail registrado | O usuário deve ter uma conta existente | Fluxo de registro |
| `tenantId` | Escopo do tenant (opcional, resolvido via e-mail) | Consulta de tenant |

---

## Fluxo

```
Etapa 1: Solicitar e-mail de redefinição    POST /auth/request-password-reset
         ↓ (usuário clica no link do e-mail)
Etapa 2: Definir nova senha                 POST /auth/reset-password
         ↓ (login automático)
         JWT retornado
```

### Etapa 1: Solicitar Redefinição de Senha

Opcionalmente, resolva o tenant primeiro (igual ao login):

```
GET /auth/user-tenants?email=user@example.com
```

Em seguida, solicite o e-mail de redefinição:

```
POST /auth/request-password-reset
Content-Type: application/json
```

**Requisição Mínima:**

```json
{
  "email": "user@example.com"
}
```

**Requisição Completa (exemplo de produção):**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "callbackUrl": "https://myapp.com/auth/reset-password"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `email` | string | Sim | E-mail do usuário |
| `tenantId` | UUID | Não | Escopo do tenant (recomendado para multi-tenant) |
| `callbackUrl` | URL | Não | URL da sua página de redefinição de senha |

**Resposta (201):** Corpo vazio (e-mail enviado)

**Formato do link no e-mail:**

```
{callbackUrl}?email={email}&token={token};{expireTimestamp}
```

Exemplo: `https://myapp.com/auth/reset-password?email=user@example.com&token=abc123;1711900000`

### Etapa 2: Definir Nova Senha

Extraia `email` e `token` dos parâmetros de query do link no e-mail.

**Verificação de expiração no cliente (opcional, mas recomendada):**

```javascript
// Formato do token: "actualToken;unixTimestamp"
const [actualToken, expireStr] = tokenParam.split(';');
const expireMs = parseInt(expireStr) * 1000;
if (Date.now() > expireMs) {
  // Token expirado — exibir UI de "link expirado"
  // Redirecionar usuário para solicitar um novo e-mail de redefinição
}
```

**Definir a nova senha:**

```
POST /auth/reset-password
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "token": "abc123;1711900000",
  "password": "NewP@ssw0rd",
  "confirmation": "NewP@ssw0rd"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `email` | string | Sim | E-mail válido | E-mail do usuário (do parâmetro de query) |
| `token` | string | Sim | Token de verificação | String completa do token incluindo expiração (do parâmetro de query) |
| `password` | string | Sim | 8-32 caracteres, maiúsculas/minúsculas + dígito | Nova senha |
| `confirmation` | string | Sim | Deve ser igual a `password` | Confirmação da senha |

**Resposta (200):** `SignInResponse` — o usuário faz **login automaticamente** após uma redefinição de senha bem-sucedida.

```json
{
  "token": "new-access-token...",
  "refreshToken": "new-refresh-token...",
  "data": {
    "sub": "uuid-user-id",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "roles": ["user"],
    "verified": true,
    "emailVerified": true,
    "type": "user"
  }
}
```

**Efeito colateral:** Se o e-mail do usuário ainda não estava verificado, ele é automaticamente marcado como verificado.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Token inválido | Token expirado, já utilizado ou malformado | Solicite um novo e-mail de redefinição |
| 400 | Erro de validação | Senha fraca ou `confirmation` não corresponde | Corrija os campos de senha |
| 404 | Usuário não encontrado | E-mail não registrado | Direcione para o registro |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Token contém ponto-e-vírgula e quebra o parsing de URL | Use `decodeURIComponent()` ao ler dos parâmetros de query. A string completa `token;timestamp` deve ser enviada para a API |
| 2 | Usuário clica em link expirado | Verifique o timestamp no lado do cliente antes de chamar a API. Exiba uma UI de "solicitar novo link" em vez de um erro da API |
| 3 | `callbackUrl` não fornecida na Etapa 1 | O e-mail não conterá um link. Sempre inclua `callbackUrl` apontando para sua página de redefinição de senha |
| 4 | Token funciona apenas uma vez | Após a redefinição bem-sucedida, o token é invalidado. Uma segunda chamada `POST /auth/reset-password` com o mesmo token falhará |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|----------------|----------|
| Login | O usuário pode fazer login após a redefinição (ou usar o JWT do login automático) | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) |
| Registro | Se o usuário não possui uma conta | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) |
