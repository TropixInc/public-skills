---
id: FLOW_AUTH_SIGNUP
title: "Auth - Fluxos de Registro"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - signup
  - registration
depends_on:
  - AUTH_API_REFERENCE
---

# Fluxos de Registro

## Visão Geral

O W3Block suporta dois fluxos de registro: registrar um usuário em um **tenant existente** (Fluxo A) e criar um **novo tenant** com o proprietário como seu primeiro admin (Fluxo B). O Fluxo B é um processo de múltiplas etapas que inclui verificação de e-mail, conclusão de perfil e configuração de cobrança.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| URL Base | URL do serviço de identidade | `https://pixwayid.w3block.io` (prod) |
| `tenantId` | UUID do tenant de destino (apenas Fluxo A) | Configuração do ambiente ou consulta de tenant |

---

## Fluxo A: Registrar Usuário em Tenant Existente

Chamada de API única. Usado ao integrar usuários a um tenant já configurado.

### Etapa 1: Criar Usuário

```
POST /auth/signup
Content-Type: application/json
```

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
  "callbackUrl": "https://myapp.com/auth/verify-email",
  "verificationType": "numeric",
  "referrer": "REFERRAL_CODE",
  "utmParams": {
    "utm_source": "google",
    "utm_medium": "cpc",
    "utm_campaign": "launch"
  }
}
```

**Resposta (201):** `SignInResponse` — o usuário recebe um JWT imediatamente, mas `data.emailVerified` será `false` até que o e-mail seja confirmado.

### Etapa 2 (Opcional): Verificar E-mail

Se `callbackUrl` foi fornecida, o usuário recebe um e-mail. Para `verificationType: "numeric"`, ele contém um código de 6 dígitos; para `"invisible"`, um link clicável.

**Para verificar com um código:**

```
GET /auth/verify-sign-up?email=user@example.com&token=123456
```

**Para reenviar o e-mail/código de verificação:**

```
POST /auth/request-confirmation-email
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "verificationType": "numeric"
}
```

**Resposta da verificação:**

```json
{
  "verified": true,
  "emailVerified": true,
  "phoneVerified": false
}
```

### Caso Especial: Tenants Passwordless

Quando um tenant tem o modo `passwordless` habilitado:
- Os campos `password` e `confirmation` tornam-se **opcionais**
- Uma senha aleatória é gerada no lado do servidor
- O usuário faz login via código OTP ou magic token
- O comportamento de e-mail é controlado pela configuração `passwordless.sendEmailWhenRegister` do tenant

---

## Fluxo B: Criar Novo Tenant (Onboarding de Tenant)

Fluxo de múltiplas etapas que cria tanto um tenant quanto seu primeiro usuário admin.

```
Etapa 1: Registrar proprietário    POST /auth/signup/tenant
         ↓
Etapa 2: Verificar e-mail          GET /auth/verify-sign-up
         ↓
Etapa 3: Completar perfil          PATCH /auth/signup/tenant/finish
         ↓
Etapa 4: Definir plano de cobrança PATCH /billing/{companyId}/plan
         ↓
Etapa 5: Definir cartão de pagamento PATCH /billing/{companyId}/credit-card
```

### Etapa 1: Registrar Proprietário do Tenant

Cria um usuário inativo no tenant do sistema. O usuário fica inativo até a Etapa 3.

```
POST /auth/signup/tenant
Content-Type: application/json
```

**Requisição Mínima:**

```json
{
  "email": "owner@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd"
}
```

**Requisição Completa:**

```json
{
  "email": "owner@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd",
  "verificationType": "numeric",
  "name": "John Doe",
  "phone": "+5511999998888"
}
```

**Resposta (201):** `SignInResponse` — contém um **JWT temporário**. Armazene este token; ele é necessário para a Etapa 3.

> **Importante:** Este é um token temporário vinculado ao tenant do sistema. O usuário está inativo e não pode acessar nenhum recurso de tenant até que a Etapa 3 seja concluída.

### Etapa 2: Verificar E-mail

Usando `verificationType: "numeric"` (recomendado para melhor UX):

```
GET /auth/verify-sign-up?email=owner@example.com&token=123456
```

**Reenviar código:**

```
POST /auth/request-confirmation-email
Content-Type: application/json
```

```json
{
  "email": "owner@example.com",
  "verificationType": "numeric"
}
```

**Resposta:** `{ "verified": true, "emailVerified": true, "phoneVerified": false }`

### Etapa 3: Completar Perfil e Criar Tenant

Usa o JWT temporário da Etapa 1.

```
PATCH /auth/signup/tenant/finish
Authorization: Bearer {temporaryToken}
Content-Type: application/json
```

```json
{
  "tenantName": "My Company",
  "ownerName": "John Doe",
  "phone": "+5511999998888"
}
```

| Campo | Tipo | Obrigatório | Validação | Descrição |
|-------|------|-------------|-----------|-----------|
| `tenantName` | string | Sim | Mín. 3 caracteres | Nome da empresa/organização |
| `ownerName` | string | Sim | Mín. 5 caracteres | Nome completo do proprietário |
| `phone` | string | Sim | Telefone válido | Telefone do proprietário |

**Resposta (200):**

```json
{
  "tenant": {
    "id": "uuid-new-tenant-id",
    "name": "My Company"
  },
  "tenantSignIn": {
    "token": "new-access-token-scoped-to-new-tenant...",
    "refreshToken": "new-refresh-token...",
    "data": {
      "sub": "uuid-user-id",
      "tenantId": "uuid-new-tenant-id",
      "roles": ["admin", "user"],
      "type": "user"
    }
  }
}
```

> **Importante:** Após esta etapa, descarte o token temporário e use o novo `tenantSignIn.token`. O usuário agora é um admin ativo do novo tenant.

**Efeitos colaterais:**
- Nova entidade de tenant criada
- Usuário migrado do tenant do sistema para o novo tenant
- Roles do usuário definidas como `[admin, user]`
- Usuário marcado como ativo
- Configuração padrão do tenant despachada (assíncrona)

### Etapa 4: Definir Plano de Cobrança

```
PATCH /billing/{companyId}/plan
Authorization: Bearer {newAccessToken}
Content-Type: application/json
```

```json
{
  "planId": "uuid-plan-id"
}
```

Para obter os planos disponíveis:

```
GET /billing/plans
Authorization: Bearer {newAccessToken}
```

### Etapa 5: Definir Cartão de Pagamento (Stripe)

Primeiro tokenize o cartão via Stripe.js, depois envie o token para a API:

```
PATCH /billing/{companyId}/credit-card
Authorization: Bearer {newAccessToken}
Content-Type: application/json
```

```json
{
  "tokenId": "tok_1234567890",
  "cardId": "card_1234567890",
  "cardLastNumbers": "4242",
  "expiryMonth": 12,
  "expiryYear": 2028,
  "brand": "Visa"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokenId` | string | Sim | ID do token Stripe de `stripe.createToken()` |
| `cardId` | string | Sim | ID do cartão Stripe da resposta do token |
| `cardLastNumbers` | string | Sim | Últimos 4 dígitos do cartão |
| `expiryMonth` | number | Sim | Mês de expiração do cartão |
| `expiryYear` | number | Sim | Ano de expiração do cartão |
| `brand` | string | Sim | Bandeira do cartão (Visa, Mastercard, etc.) |

---

## Notificações por E-mail

| Evento | Template | Gatilho |
|--------|----------|---------|
| Confirmação de registro (link) | `signup.html` | `verificationType: "invisible"` |
| Confirmação de registro (código) | `signup-code.html` | `verificationType: "numeric"` |
| Confirmação de convite | `invite.html` | Usuário criado com role de admin |
| Completar perfil | `default.html` | Tipo de verificação padrão |

URLs de e-mail de confirmação incluem: `?token={token}&code={code}&expire={timestamp}&email={email}&tenantId={tenantId}`.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Erro de validação | E-mail inválido, senha fraca ou campos obrigatórios ausentes | Verifique as validações dos campos |
| 400 | Código de indicação inválido | O código `referrer` não corresponde a nenhum usuário no tenant | Verifique o código de indicação |
| 409 | E-mail já existe | Usuário já registrado neste tenant | Direcione para o login |
| 422 | Entidade não processável | `confirmation` não corresponde a `password` | Certifique-se de que ambos os campos sejam idênticos |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Token temporário da Etapa 1 usado para chamadas de API | Este token é apenas para a Etapa 3. Ele é vinculado ao tenant do sistema e o usuário está inativo |
| 2 | Pular verificação de e-mail antes da Etapa 3 | Embora tecnicamente possível, o `emailVerified` do usuário será `false`, o que pode bloquear funcionalidades |
| 3 | Usar verificação `invisible` em apps mobile | Use `numeric` — deep links para URLs de verificação por e-mail não são confiáveis em mobile |
| 4 | Regras de senha falhando em tenants passwordless | Verifique a configuração do tenant primeiro. Tenants passwordless geram senhas automaticamente |
| 5 | `callbackUrl` rejeitada | O host da URL deve estar na lista de hosts permitidos do tenant. Contate o admin para adicionar seu domínio à whitelist |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|----------------|----------|
| Login | Fazer login após o registro | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) |
| Redefinição de Senha | Se o usuário esquecer a senha | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) |
| OAuth | Registro alternativo via Google/Apple | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) |
