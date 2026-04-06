---
id: AUTH_SKILL_INDEX
title: "Índice de Skills de Autenticação"
module: offpix/auth
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
---

# Índice de Skills de Autenticação

Documentação do módulo de autenticação do W3Block Identity Service (PixwayID). Abrange registro de usuários, login (5 métodos), redefinição de senha, OAuth, gerenciamento de tokens e onboarding de tenants.

**Serviço:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [AUTH_API_REFERENCE](./AUTH_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums e schemas de autenticação | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) | 1.0.0 | Login por senha, código OTP, magic token, chave de API do tenant | Implementado | Implementar qualquer método de login |
| 3 | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) | 1.0.0 | Registro de usuário + onboarding de tenant (múltiplas etapas) | Implementado | Implementar fluxo de registro |
| 4 | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) | 1.0.0 | Solicitar e-mail de redefinição + definir nova senha | Implementado | Implementar recuperação de senha |
| 5 | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) | 1.0.0 | Google & Apple OAuth (redirecionamento + token direto) | Implementado | Implementar login social |

---

## Guia Rápido

### Para uma implementação básica de autenticação:

```
1. Leia: FLOW_AUTH_SIGNUP.md              → Implementar registro de usuário
2. Leia: FLOW_AUTH_SIGNIN.md              → Implementar login (escolha seu método)
3. Leia: FLOW_AUTH_PASSWORD_RESET.md      → Implementar recuperação de senha
4. Consulte: AUTH_API_REFERENCE.md        → Para schemas, enums e casos especiais
```

### Implementação mínima orientada a API (3 chamadas):

```
1. POST /auth/signup          → Registrar usuário
2. POST /auth/signin          → Fazer login (obter JWT)
3. POST /auth/refresh-token   → Manter sessão ativa
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | 409 ao fazer login sem `tenantId` | Chame `GET /auth/user-tenants?email=X` primeiro para resolver o tenant |
| 2 | Validação de senha inconsistente | Tenants com modo `passwordless` possuem regras diferentes. Verifique a configuração do tenant |
| 3 | Token temporário do cadastro de tenant usado para chamadas de API | Use-o apenas para `PATCH /auth/signup/tenant/finish`. Obtenha um JWT real após isso |
| 4 | Renovação de token falha silenciosamente | Monitore o `RefreshAccessTokenError` e force a reautenticação |
| 5 | Código OTP exige `tenantId` mas login por senha não | Projete a UX para coletar o tenant antes de exibir a opção de OTP |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|--------------|-----------|
| Fazer login com e-mail + senha | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Fluxo A |
| Fazer login com código OTP | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Fluxo B |
| Fazer login com magic link (gerado por admin) | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Fluxo C |
| Autenticar servidor-a-servidor (chave de API) | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Fluxo D |
| Registrar um usuário em tenant existente | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) — Fluxo A |
| Criar um novo tenant + usuário admin | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) — Fluxo B |
| Redefinir uma senha esquecida | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) |
| Adicionar login com Google | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) — Google |
| Adicionar login com Apple | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) — Apple |
| Renovar um token prestes a expirar | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Renovação de Token |
| Verificar uma assinatura JWT externamente | [AUTH_API_REFERENCE](./AUTH_API_REFERENCE.md) — Endpoint JWKS |

---

## Matriz: Endpoints x Documentos

| Endpoint | Método | Login | Registro | Redefinição de Senha | OAuth | Ref. API |
|----------|--------|:-----:|:--------:|:--------------------:|:-----:|:--------:|
| `/auth/signin` | POST | **X** | | | | **X** |
| `/auth/signin/code` | POST | **X** | | | | **X** |
| `/auth/signin/request-code` | POST | **X** | | | | **X** |
| `/auth/signin/magic-token` | POST | **X** | | | | **X** |
| `/auth/signin/generate-magic-token` | POST | **X** | | | | **X** |
| `/auth/signin/tenant` | POST | **X** | | | | **X** |
| `/auth/signup` | POST | | **X** | | | **X** |
| `/auth/signup/tenant` | POST | | **X** | | | **X** |
| `/auth/signup/tenant/finish` | PATCH | | **X** | | | **X** |
| `/auth/verify-sign-up` | GET | | **X** | | | **X** |
| `/auth/request-confirmation-email` | POST | | **X** | | | **X** |
| `/auth/request-password-reset` | POST | | | **X** | | **X** |
| `/auth/reset-password` | POST | | | **X** | | **X** |
| `/auth/refresh-token` | POST | **X** | | | | **X** |
| `/auth/logout` | POST | **X** | | | | **X** |
| `/auth/user-tenants` | GET | **X** | | **X** | | **X** |
| `/auth/{tenantId}/signin/google` | GET | | | | **X** | **X** |
| `/auth/{tenantId}/signin/google/code` | GET | | | | **X** | **X** |
| `/auth/signin/google` | POST | | | | **X** | **X** |
| `/auth/{tenantId}/signin/apple` | GET | | | | **X** | **X** |
| `/auth/{tenantId}/signin/apple/code` | POST | | | | **X** | **X** |
| `/auth/signin/apple` | POST | | | | **X** | **X** |
| `/auth/jwks.json` | GET | | | | | **X** |
| `/billing/plans` | GET | | **X** | | | **X** |
| `/billing/{id}/plan` | PATCH | | **X** | | | **X** |
| `/billing/{id}/credit-card` | PATCH | | **X** | | | **X** |
| `/billing/{id}/state` | GET | | **X** | | | **X** |

---

## Diagrama de Fluxo

```
                         ┌─────────────────────────────────────────────────┐
                         │           MÓDULO DE AUTH W3BLOCK                 │
                         │                                                 │
  ┌──────────────┐       │  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
  │ Novo Usuário │──────→│  │ Registro │───→│ Verificar│───→│  Login   │  │
  └──────────────┘       │  │ (A ou B) │    │  E-mail  │    │          │  │
                         │  └──────────┘    └──────────┘    └────┬─────┘  │
  ┌──────────────┐       │                                       │        │
  │Usuário Exist.│──────→│  ┌──────────────────────────────┐     │        │
  └──────────────┘       │  │ Login (5 métodos)            │     │        │
                         │  │ • Senha                      │─────┤        │
                         │  │ • Código OTP                 │     │        │
  ┌──────────────┐       │  │ • Magic Token                │     │        │
  │ Esqueceu     │──────→│  │ • Chave de API do Tenant     │     │        │
  │ a Senha      │       │  │ • Google / Apple OAuth       │     │        │
  └──────────────┘       │  └──────────────────────────────┘     │        │
                         │                                       ▼        │
  ┌──────────────┐       │  ┌──────────┐               ┌──────────────┐  │
  │ Redefinir    │──────→│  │ Solicitar│──→ e-mail ──→ │ Redefinir +  │  │
  │ Senha        │       │  │ Redefin. │               │ Login Auto   │  │
  └──────────────┘       │  └──────────┘               └──────────────┘  │
                         │                                       │        │
                         │                                       ▼        │
                         │                              ┌──────────────┐  │
                         │                              │  JWT Token   │  │
                         │                              │  + Refresh   │  │
                         │                              └──────────────┘  │
                         └─────────────────────────────────────────────────┘
```
