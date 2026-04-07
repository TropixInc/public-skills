---
id: FLOW_SETTINGS_TENANT_CONFIGURATION
title: "Configuracoes - Configuracao do Tenant"
module: offpix/settings
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - settings
  - tenant
  - configuration
  - auth-providers
  - kyc
  - passwordless
  - email
depends_on:
  - SETTINGS_SKILL_INDEX
  - SETTINGS_API_REFERENCE
---

# Configuracao do Tenant

## Visao Geral

Este fluxo cobre o gerenciamento do perfil do tenant e a configuracao de integracoes e funcionalidades: provedores de autenticacao (Google OAuth, Apple OAuth, passwordless), configuracoes de KYC, servico de e-mail, notificacoes push (OneSignal), mensagens WhatsApp (Twilio), deteccao de fraude (ClearSale) e exclusao automatica de conta. Tambem abrange o controle de funcionalidades via verificacao do plano de cobranca.

## Pre-requisitos

| Requisito | Descricao | Como obter |
|-----------|-----------|------------|
| `tenantId` | UUID do Tenant | Fluxo de autenticacao / configuracao do ambiente |
| Bearer token | Token de acesso JWT (role Admin) | Autenticacao no servico ID |
| Credenciais do provedor | Chaves/segredos de API para OAuth, e-mail, etc. | Paineis dos provedores externos |

## Entidades e Relacionamentos

```
TenantEntity (1) -----> (1) TenantConfigurationsEntity
     |
     +-----> (1) TenantPlanEntity -----> (1) BillingPlanEntity
                                               |
                                               +-- enabledFeatures: BillingFeature[]
```

**Conceitos-chave:**
- Cada tenant possui exatamente uma TenantConfigurationsEntity (criada no primeiro POST)
- Os campos de configuracao sao independentes -- voce pode atualizar um sem afetar os outros
- A disponibilidade de funcionalidades depende do plano de cobranca do tenant (`enabledFeatures`)
- TenantEntity armazena o perfil (nome, documento, pais); TenantConfigurationsEntity armazena as integracoes

---

## Fluxo: Gerenciamento do Perfil do Tenant

### Passo 1: Obter Detalhes do Tenant

**Endpoint:**

| Metodo | Caminho | Autenticacao | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/tenant/{tenantId}` | Bearer token (Admin) | - |

**Resposta (200):**
```json
{
  "id": "tenant-uuid",
  "name": "My Company LLC",
  "document": "12345678000190",
  "countryCode": "BR",
  "wallets": [
    {
      "address": "0x1234...abcd",
      "type": "operator"
    }
  ],
  "clientId": "client-uuid",
  "info": {
    "website": "https://example.com",
    "industry": "retail"
  },
  "roles": ["admin"],
  "operatorAddress": "0x1234...abcd",
  "setupDone": true
}
```

**Observacoes:**
- `wallets` contem as carteiras blockchain associadas ao tenant
- `setupDone` indica se o tenant completou o onboarding inicial
- `operatorAddress` e o operador blockchain padrao para transacoes

### Passo 2: Atualizar Perfil do Tenant

**Endpoint:**

| Metodo | Caminho | Autenticacao | Content-Type |
|--------|---------|--------------|-------------|
| PUT | `/tenant/profile/{tenantId}` | Bearer token (Admin) | application/json |

**Requisicao Minima:**
```json
{
  "name": "My Company"            // obrigatorio
}
```

**Requisicao Completa:**
```json
{
  "name": "My Company LLC",       // obrigatorio
  "document": "12345678000190",   // opcional -- registro empresarial (CNPJ, EIN, etc.)
  "countryCode": "BR",           // opcional -- ISO 3166-1 alpha-2
  "info": {                       // opcional -- metadados livres
    "website": "https://example.com",
    "industry": "retail",
    "contactEmail": "admin@example.com"
  }
}
```

**Resposta (200):** TenantEntity atualizada.

**Referencia de Campos:**

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibicao do tenant |
| `document` | string | Nao | Numero de registro empresarial |
| `countryCode` | string | Nao | Codigo de pais ISO |
| `info` | object | Nao | Objeto de metadados livres |

**Observacoes:**
- Este e um PUT (substituicao completa), nao um PATCH -- envie todos os campos que deseja manter
- `wallets`, `roles` e `operatorAddress` nao sao modificaveis por este endpoint

---

## Fluxo: Configuracao do Tenant (Integracoes)

### Passo 3: Obter Configuracao Atual

**Endpoint:**

| Metodo | Caminho | Autenticacao | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/tenant/configurations/{tenantId}` | Bearer token (Admin) | - |

**Resposta (200):**
```json
{
  "tenantId": "tenant-uuid",
  "kyc": null,
  "signUp": null,
  "passwordless": { "enabled": false },
  "googleOAuth": null,
  "appleOAuth": null,
  "oneSignal": null,
  "twilioWhatsApp": null,
  "clearSale": null,
  "emailService": null,
  "automaticAccountExclusion": false
}
```

**Observacoes:**
- Campos `null` significam que a integracao nao esta configurada
- Sempre busque a configuracao atual antes de atualizar para entender o estado corrente

### Passo 4: Atualizar Configuracao

**Endpoint:**

| Metodo | Caminho | Autenticacao | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/tenant/configurations/{tenantId}` | Bearer token (Admin) | application/json |

Este endpoint utiliza comportamento de upsert: cria a configuracao na primeira chamada e atualiza nas chamadas subsequentes.

#### 4a: Habilitar Autenticacao Passwordless

**Requisicao Minima:**
```json
{
  "passwordless": {
    "enabled": true
  }
}
```

#### 4b: Configurar Google OAuth

**Requisicao:**
```json
{
  "googleOAuth": {
    "clientId": "123456789-abc.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxxxxxxxxx",
    "enabled": true
  }
}
```

#### 4c: Configurar Apple OAuth

**Requisicao:**
```json
{
  "appleOAuth": {
    "clientId": "com.example.app",
    "teamId": "ABC123DEF4",
    "keyId": "KEY123",
    "enabled": true
  }
}
```

#### 4d: Configurar KYC

**Requisicao:**
```json
{
  "kyc": {
    "provider": "clear_sale",
    "enabled": true,
    "requiredForPurchase": false
  }
}
```

#### 4e: Configurar Servico de E-mail

**Requisicao:**
```json
{
  "emailService": {
    "provider": "sendgrid",
    "apiKey": "SG.xxxxxxxxxxxx",
    "fromEmail": "noreply@example.com",
    "fromName": "My Company"
  }
}
```

#### 4f: Configurar Notificacoes Push (OneSignal)

**Requisicao:**
```json
{
  "oneSignal": {
    "appId": "onesignal-app-uuid",
    "apiKey": "onesignal-rest-api-key"
  }
}
```

#### 4g: Configurar WhatsApp (Twilio)

**Requisicao:**
```json
{
  "twilioWhatsApp": {
    "accountSid": "ACxxxxxxxxxxxx",
    "authToken": "auth-token-here",
    "fromNumber": "+15551234567"
  }
}
```

#### 4h: Configurar Deteccao de Fraude (ClearSale)

**Requisicao:**
```json
{
  "clearSale": {
    "enabled": true,
    "apiKey": "clearsale-api-key"
  }
}
```

#### 4i: Habilitar Exclusao Automatica de Conta

**Requisicao:**
```json
{
  "automaticAccountExclusion": true
}
```

#### Configuracao Completa (todos os campos):

```json
{
  "kyc": {
    "provider": "clear_sale",
    "enabled": true,
    "requiredForPurchase": false
  },
  "signUp": {
    "requireEmailVerification": true,
    "allowedDomains": ["example.com"]
  },
  "passwordless": {
    "enabled": true
  },
  "googleOAuth": {
    "clientId": "123456789-abc.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxxxxxxxxx",
    "enabled": true
  },
  "appleOAuth": {
    "clientId": "com.example.app",
    "teamId": "ABC123DEF4",
    "keyId": "KEY123",
    "enabled": true
  },
  "oneSignal": {
    "appId": "onesignal-app-uuid",
    "apiKey": "onesignal-rest-api-key"
  },
  "twilioWhatsApp": {
    "accountSid": "ACxxxxxxxxxxxx",
    "authToken": "auth-token-here",
    "fromNumber": "+15551234567"
  },
  "clearSale": {
    "enabled": true,
    "apiKey": "clearsale-api-key"
  },
  "emailService": {
    "provider": "sendgrid",
    "apiKey": "SG.xxxxxxxxxxxx",
    "fromEmail": "noreply@example.com",
    "fromName": "My Company"
  },
  "automaticAccountExclusion": false
}
```

**Resposta (201):** TenantConfigurationsEntity criada/atualizada.

**Observacoes:**
- Voce pode enviar um payload parcial com apenas os campos que deseja atualizar
- Campos JSONB (`kyc`, `signUp`) sao armazenados como estao -- o backend nao faz merge de objetos aninhados
- Definir um campo como `null` desabilita aquela integracao
- Campos sensiveis (chaves de API, segredos) sao armazenados criptografados

---

## Fluxo: Controle de Funcionalidades

### Passo 5: Verificar se uma Funcionalidade esta Habilitada

Use este endpoint para controlar elementos de UI ou operacoes com base no plano de cobranca do tenant.

**Endpoint:**

| Metodo | Caminho | Autenticacao | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/billing/{tenantId}/is-feature-enabled/{feature}` | Bearer token (Admin/User/Operator) | - |

**Exemplo de Requisicao:**
```
GET /billing/tenant-uuid/is-feature-enabled/COMMERCE
```

**Resposta (200):**
```json
{
  "enabled": true
}
```

**Verificacoes Comuns de Funcionalidades:**

| Funcionalidade | Caso de Uso |
|----------------|-------------|
| `COMMERCE` | Exibir/ocultar vitrine e gerenciamento de produtos |
| `LOYALTY` | Exibir/ocultar funcionalidades de programa de fidelidade |
| `KYC` | Habilitar/desabilitar fluxos de verificacao KYC |
| `MINT_ERC_721` | Permitir mintagem de NFT |
| `MINT_ERC_20` | Permitir mintagem de ERC-20 |
| `WHITE_LABEL` | Habilitar opcoes de marca white-label |
| `CUSTOM_DOMAIN` | Exibir configuracoes de dominio personalizado |
| `FULL_TOKEN_PASS` | Habilitar funcionalidades avancadas de token pass |
| `CUSTODIAL_WALLET` | Habilitar gerenciamento de carteira custodial |
| `EXTERNAL_ECOMMERCE_INTEGRATION` | Exibir opcoes de integracao com e-commerce externo |

**Observacoes:**
- Este endpoint esta disponivel para as roles Admin, User e Operator -- nao apenas para admins
- A verificacao e feita contra `BillingPlanEntity.features.enabledFeatures` do plano atual do tenant
- Limites personalizados na TenantPlanEntity nao afetam a disponibilidade de funcionalidades
- Use isto como uma guarda antes de exibir UI especifica de funcionalidade ou fazer chamadas de API especificas

---

## Tratamento de Erros

| Status | Erro | Causa | Resolucao |
|--------|------|-------|-----------|
| 400 | VALIDATION_ERROR | Valores de campo invalidos (JSON incorreto, tipos errados) | Valide a estrutura do payload antes de enviar |
| 401 | UNAUTHORIZED | Token ausente ou expirado | Reautentique |
| 403 | FORBIDDEN | Nao-admin tentando atualizar configuracao/perfil | Garanta a role Admin |
| 404 | TENANT_NOT_FOUND | `tenantId` invalido | Verifique se o tenant existe |
| 404 | CONFIGURATION_NOT_FOUND | GET de configuracao antes de qualquer POST (nenhuma configuracao existe ainda) | Crie a configuracao primeiro com POST |
| 422 | INVALID_FEATURE | Nome de funcionalidade desconhecido em is-feature-enabled | Verifique os valores do enum BillingFeature |

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | PUT no perfil apaga campos nao incluidos | PUT e uma substituicao completa; sempre inclua todos os campos que deseja manter |
| 2 | OAuth nao funciona apos atualizacao da configuracao | Verifique se `enabled: true` esta definido junto com as credenciais |
| 3 | Verificacao de funcionalidade retorna false para uma funcionalidade configurada | A disponibilidade de funcionalidade depende do plano de cobranca, nao da configuracao. Faca upgrade do plano se necessario |
| 4 | Configuracao de KYC salva mas fluxo de KYC nao aparece | Verifique se a funcionalidade de cobranca `KYC` esta habilitada no plano E se `kyc.enabled` esta `true` na configuracao |
| 5 | GET de configuracoes retorna 404 | Nenhuma configuracao existe ainda; faca POST primeiro para criar uma |
| 6 | Atualizacao parcial de configuracao sobrescreve campos JSONB | Campos JSONB (kyc, signUp) sao substituidos inteiramente; envie o objeto completo, nao apenas subcampos alterados |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Autenticacao | Obrigatorio para todos os endpoints; configuracao OAuth afeta fluxos de autenticacao | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Gerenciamento de Cobranca | Plano determina funcionalidades disponiveis | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Submissao de KYC | Configuracao de KYC controla fluxos de verificacao | [FLOW_CONTACTS_KYC_SUBMISSION](../contacts/FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Configuracoes | Configuracao de contexto/formulario complementa a configuracao do tenant | [CONFIGURATIONS_SKILL_INDEX](../configurations/CONFIGURATIONS_SKILL_INDEX.md) |
