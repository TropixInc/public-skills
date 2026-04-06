---
id: FLOW_AUTH_OAUTH
title: "Auth - Fluxos OAuth Google & Apple"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - oauth
  - google
  - apple
depends_on:
  - AUTH_API_REFERENCE
  - FLOW_AUTH_SIGNIN
---

# Fluxos OAuth Google & Apple

## Visão Geral

O W3Block suporta Google e Apple OAuth como métodos alternativos de login. Ambos suportam dois padrões de integração: **fluxo de redirecionamento** (troca de código no lado do servidor) e **fluxo de token direto** (envio de credencial no lado do cliente). Ambos criam contas de usuário automaticamente no primeiro login.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `tenantId` | UUID do tenant de destino | Configuração do ambiente |
| OAuth configurado | Google/Apple OAuth deve estar configurado para o tenant | Painel administrativo ou API de configuração do tenant |
| URL Base | URL do serviço de identidade | `https://pixwayid.w3block.io` (prod) |

---

## Google OAuth

### Fluxo A: Redirecionamento (Authorization Code)

Ideal para aplicações web. O backend gerencia a troca OAuth.

#### Etapa 1: Redirecionar para o Google

Navegue o navegador do usuário para:

```
GET /auth/{tenantId}/signin/google
```

O backend redireciona para a tela de consentimento do Google com o client ID e escopos corretos configurados para o tenant.

#### Etapa 2: Tratar o Callback

Após o usuário consentir, o Google redireciona de volta para seu aplicativo com um parâmetro de query `code`. Faça a troca:

```
GET /auth/{tenantId}/signin/google/code?code={authorizationCode}
```

Parâmetro de query opcional: `referrer={referralCode}`

**Resposta (200):**

```json
{
  "token": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "data": {
    "sub": "uuid-user-id",
    "tenantId": "uuid-tenant-id",
    "email": "user@gmail.com",
    "name": "John Doe",
    "roles": ["user"],
    "type": "user"
  },
  "isNewUser": true
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `isNewUser` | boolean | `true` se este é o primeiro login do usuário (a conta acabou de ser criada) |

**Use `isNewUser`** para redirecionar novos usuários para uma página de completar perfil ou onboarding.

### Fluxo B: Token Direto (SPA / Mobile)

Para aplicativos que obtêm o Google ID token no lado do cliente (ex.: via Google Sign-In SDK).

```
POST /auth/signin/google
Content-Type: application/json
```

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id",
  "referrer": "REFERRAL_CODE"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `credential` | string | Sim | JWT do Google ID token |
| `tenantId` | UUID | Sim | Tenant de destino |
| `referrer` | string | Não | Código de indicação |

**Resposta (200):** Igual ao fluxo de redirecionamento.

---

## Apple OAuth

### Fluxo A: Redirecionamento (Authorization Code)

#### Etapa 1: Redirecionar para a Apple

Navegue o navegador do usuário para:

```
GET /auth/{tenantId}/signin/apple
```

O backend redireciona para a tela de consentimento da Apple. A configuração do Apple OAuth é carregada dinamicamente por tenant.

#### Etapa 2: Tratar o Callback

Após o consentimento, a Apple redireciona de volta. Faça a troca do código:

```
POST /auth/{tenantId}/signin/apple/code
```

**Resposta (200):**

```json
{
  "token": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "data": { ... },
  "isNewUser": true,
  "isPrivateEmail": true
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `isNewUser` | boolean | `true` se a conta acabou de ser criada |
| `isPrivateEmail` | boolean | `true` se o usuário escolheu o e-mail de retransmissão privado da Apple |

### Fluxo B: Token Direto (Mobile)

Para aplicativos que usam o Apple Sign-In SDK:

```
POST /auth/signin/apple
Content-Type: application/json
```

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id"
}
```

Aceita tanto o campo `credential` quanto `id_token` — ambos esperam um JWT do Apple ID token.

**Resposta (200):** Igual ao fluxo de redirecionamento.

---

## Comportamento de Auto-Registro

Tanto o Google quanto o Apple OAuth criam contas de usuário automaticamente no primeiro login:

1. O backend verifica o token/código OAuth
2. Extrai o e-mail e informações de perfil do token
3. Verifica se um usuário com este e-mail existe no tenant de destino
4. **Se existir:** Vincula o provedor OAuth à conta existente e faz login
5. **Se não existir:** Cria um novo usuário com as informações do perfil OAuth e faz login

O comportamento de resolução de conflitos (etapa 4) é configurável por tenant via `thirdPartyAuthConflictResolution`:

| Valor | Comportamento |
|-------|---------------|
| `ALWAYS_UPDATE` | Sempre atualiza o perfil do usuário com os dados do OAuth |
| `UPDATE_WHEN_JUST_HAVE_PROVIDER_AUTH` | Atualiza apenas se o usuário não tiver senha definida |
| `IGNORE` | Nunca atualiza os dados de perfil existentes |

---

## E-mail de Retransmissão Privado da Apple

Quando `isPrivateEmail: true`, o usuário optou por ocultar seu e-mail real. A Apple fornece um endereço de retransmissão privado como `abc123@privaterelay.appleid.com`. Trate isso da seguinte forma:

- Armazene o e-mail de retransmissão como o e-mail do usuário (ele encaminha para a caixa de entrada real)
- Não o exiba como um e-mail "real" na sua interface
- Opcionalmente, solicite ao usuário que forneça seu e-mail real nas configurações de perfil

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Credencial inválida | Token OAuth expirado, malformado ou de client errado | Reautentique com o provedor OAuth |
| 400 | Configuração de tenant inválida | OAuth não configurado para este tenant | Configure o Google/Apple OAuth nas configurações do tenant |
| 409 | Conflito de e-mail | E-mail já existe com um método de autenticação diferente | Depende da configuração `thirdPartyAuthConflictResolution` |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Google OAuth não funciona para um tenant | Verifique se o tenant possui client ID/secret do Google OAuth configurados |
| 2 | Apple OAuth requer configuração por tenant | A Apple exige uma chave privada por aplicativo. Cada tenant pode precisar de sua própria configuração no Apple Developer |
| 3 | `isNewUser` é false no primeiro login com Google | Se o usuário já se registrou com e-mail+senha, o Google OAuth vincula à conta existente |
| 4 | URL de redirecionamento incompatível | A URL de callback deve estar registrada no console de desenvolvedor do Google/Apple |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|----------------|----------|
| Login | Alternativa ao login por senha | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) |
| Registro | OAuth cria contas automaticamente | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) |
