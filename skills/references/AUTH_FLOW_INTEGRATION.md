---
id: AUTH_FLOW_INTEGRATION
title: "Guia de Integração de Autenticação Cross-Module"
module: references
version: "1.0.0"
type: reference
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
tags:
  - auth
  - integration
  - cross-module
---

# Guia de Integração de Autenticação Cross-Module

Guia para integração direta com as APIs W3Block **sem o SDK React**. Cobre obtenção de tokens, headers, refresh, resolução de tenant e rate limits.

> **Para integração via SDK React:** Consulte [SETUP_API_PATTERNS.md](../_setup/SETUP_API_PATTERNS.md).
> **Referência completa de endpoints:** Consulte [AUTH_API_REFERENCE.md](./auth/AUTH_API_REFERENCE.md).

---

## URLs Base dos Serviços

| Serviço | URL de Produção | Swagger | Responsabilidade |
|---------|-----------------|---------|------------------|
| **PixwayID** (Identity) | `https://pixwayid.w3block.io` | [/docs](https://pixwayid.w3block.io/docs/) | Auth, usuários, KYC, settings, billing |
| **KEY** (Registry) | `https://api.w3block.io` | [/docs](https://api.w3block.io/docs/) | Contratos, tokens, loyalty, tokenização, whitelist |
| **Commerce** | `https://commerce.w3block.io` | [/docs](https://commerce.w3block.io/docs/) | Produtos, pedidos, pagamentos, promoções |
| **Pass** | `https://pass.w3block.io` | [/docs](https://pass.w3block.io/docs/) | Token passes, benefícios, operadores |
| **PDF** | `https://pdf.w3block.io` | [/docs](https://pdf.w3block.io/docs/) | Geração de certificados e tickets PDF |

> **Staging:** Entre em contato com seu representante W3Block para obter URLs de staging.

---

## 1. Obtendo Tokens de Acesso

### 1.1 — Login de Usuário (senha)

```bash
curl -X POST https://pixwayid.w3block.io/auth/signin \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "MyP@ssw0rd",
    "tenantId": "uuid-do-seu-tenant"
  }'
```

**Resposta (200):**

```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6InJlZnJlc2gifQ...",
  "data": {
    "sub": "uuid-user-id",
    "iss": "uuid-tenant-id",
    "exp": 1711900000,
    "type": "user",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "roles": ["user"],
    "verified": true
  }
}
```

> **Nota sobre `tenantId`:** Se omitido e o email existir em múltiplos tenants, a API retorna **409 Conflict** com a lista de tenants. Use `GET /auth/user-tenants?email=...` para resolver antes.

### 1.2 — Login de Tenant (API Key — server-to-server)

Para integrações backend/server-to-server, use credenciais de API do tenant:

```bash
curl -X POST https://pixwayid.w3block.io/auth/signin/tenant \
  -H "Content-Type: application/json" \
  -d '{
    "key": "sua-api-key",
    "secret": "seu-api-secret",
    "tenantId": "uuid-do-tenant"
  }'
```

**Resposta (200):** Mesma estrutura, mas com `data.type = "tenant"` e `data.roles` contendo `TenantRoleEnum` (ex: `["integration"]`).

### 1.3 — Login via Código OTP

```bash
# Passo 1: Solicitar código
curl -X POST https://pixwayid.w3block.io/auth/signin/request-code \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "tenantId": "uuid-do-tenant"
  }'

# Passo 2: Usar o código recebido por email
curl -X POST https://pixwayid.w3block.io/auth/signin/code \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "code": "123456",
    "tenantId": "uuid-do-tenant"
  }'
```

### 1.4 — Login via Google/Apple OAuth

| Método | Endpoint | Uso |
|--------|----------|-----|
| Redirect (web) | `GET /auth/{tenantId}/signin/google` | Redireciona para tela de consentimento |
| Token direto (mobile/SPA) | `POST /auth/signin/google` | Envia `credential` (Google ID Token JWT) |
| Redirect (web) | `GET /auth/{tenantId}/signin/apple` | Redireciona para Apple |
| Token direto (mobile/SPA) | `POST /auth/signin/apple` | Envia `credential` (Apple ID Token JWT) |

---

## 2. Usando o Token em Requisições

### 2.1 — Header de Autenticação

Todos os endpoints autenticados exigem:

```
Authorization: Bearer {token}
```

Exemplo:

```bash
curl -X GET https://api.w3block.io/{companyId}/token-collections \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..."
```

### 2.2 — Estrutura do JWT Decodificado

O `token` é um JWT RS256 com o seguinte payload:

```json
{
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
}
```

**Campos importantes para a lógica da aplicação:**
- `exp` — expiração em Unix timestamp (segundos). Multiplique por 1000 para comparar com `Date.now()`
- `roles` — determina permissões do usuário
- `tenantId` — escopo do tenant (equivalente a `companyId` nas URLs)
- `type` — `"user"` para login de usuário, `"tenant"` para login via API key

### 2.3 — Verificação de Token (server-side)

Para verificar tokens em seu backend, busque a chave pública:

```bash
curl https://pixwayid.w3block.io/auth/jwks.json
```

Use o JWKS retornado para validar a assinatura RS256 do JWT. O JWKS tem cache de 10 minutos no CDN.

---

## 3. Refresh Token

Tokens de acesso expiram rapidamente. Use o refresh token para obter novos tokens sem re-autenticar:

```bash
curl -X POST https://pixwayid.w3block.io/auth/refresh-token \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6InJlZnJlc2gifQ..."
  }'
```

**Resposta (200):**

```json
{
  "token": "novo-access-token...",
  "refreshToken": "novo-refresh-token..."
}
```

**Comportamento importante:**
- O refresh token antigo é invalidado com atraso de 30 segundos (grace period para requisições em andamento)
- O refresh token contém um `tokenHash` (SHA-256 do access token original)
- Se o refresh falhar, o usuário precisa fazer login novamente

### Fluxo recomendado de refresh:

```
1. Faça a requisição normalmente
2. Se receber 401 (token expirado):
   a. Chame POST /auth/refresh-token com o refreshToken salvo
   b. Atualize ambos os tokens no storage
   c. Repita a requisição original com o novo access token
3. Se o refresh também falhar → redirecione para login
```

---

## 4. Logout

```bash
curl -X POST https://pixwayid.w3block.io/auth/logout \
  -H "Authorization: Bearer {token-atual}"
```

O token é colocado em blacklist via SHA-256 hash no cache. O TTL da blacklist = tempo restante até a expiração natural do token.

---

## 5. `tenantId` vs `companyId`

Na prática, **são o mesmo valor** (UUID do tenant). A diferença é apenas de nomenclatura:

| Contexto | Nome usado | Exemplo |
|----------|-----------|---------|
| Headers e body de auth | `tenantId` | `"tenantId": "uuid"` no POST /auth/signin |
| Path params nas APIs de negócio | `companyId` | `GET /{companyId}/token-collections` |
| JWT payload | `tenantId` | `data.tenantId` no SignInResponse |
| SDK React | `companyId` | `useCompanyConfig().companyId` |

**Regra prática:** Use `tenantId` quando falando com PixwayID (auth). Use `companyId` nos path params de KEY, Commerce e Pass.

---

## 6. Rate Limiting

| Escopo | Limite padrão | Endpoints com limite diferente |
|--------|--------------|-------------------------------|
| Global | 120 req/min | — |
| Login por senha | 60 req/min | `POST /auth/signin` |
| Código OTP | 10 req/min | `POST /auth/signin/request-code`, `POST /auth/signin/code` |
| OAuth | 10 req/min | Todos os endpoints de Google/Apple |
| Signup tenant | 10 req/min | `POST /auth/signup/tenant`, `PATCH /auth/signup/tenant/finish` |

**Resposta quando excede o limite:**

```json
{
  "statusCode": 429,
  "message": "Too Many Requests"
}
```

**Estratégia recomendada:** Implemente exponential backoff com jitter. Comece com 1s e dobre a cada retry, até no máximo 3 tentativas.

---

## 7. Fluxo Completo de Integração

### Para uma aplicação frontend (SPA/Mobile):

```
1. POST /auth/signin (ou /signup) → obter token + refreshToken
2. Armazenar tokens de forma segura (httpOnly cookies ou secure storage)
3. Incluir "Authorization: Bearer {token}" em toda requisição
4. Ao receber 401 → POST /auth/refresh-token → atualizar tokens
5. Se refresh falhar → redirecionar para login
6. No logout → POST /auth/logout + limpar tokens locais
```

### Para uma integração backend (server-to-server):

```
1. POST /auth/signin/tenant com API key + secret → obter token
2. Armazenar token em memória (não persista em disco)
3. Incluir "Authorization: Bearer {token}" em toda requisição
4. Monitorar exp do JWT → refresh antes de expirar
5. Em caso de erro de auth → re-autenticar com API key
```

---

## 8. Armadilhas Comuns

| Armadilha | Solução |
|-----------|---------|
| Omitir `tenantId` no signin e receber 409 | Sempre passe `tenantId` se você sabe o tenant, ou use `GET /auth/user-tenants` para resolver |
| Comparar `exp` do JWT com `Date.now()` sem multiplicar por 1000 | `exp` está em segundos, `Date.now()` em milissegundos. Multiplique `exp * 1000` |
| Usar o refresh token antigo após refresh | Após refresh, descarte **ambos** os tokens antigos. O refresh antigo é invalidado em 30s |
| Enviar `Authorization` header em endpoints públicos | O header é **validado** mesmo em endpoints públicos. Se o token estiver expirado, a requisição falha |
| Confundir JWT de usuário com JWT de tenant | Verifique `data.type` — `"user"` para login de usuário, `"tenant"` para API key |
