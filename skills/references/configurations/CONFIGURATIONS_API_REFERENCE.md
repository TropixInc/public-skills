---
id: CONFIGURATIONS_API_REFERENCE
title: "Configurations - Referência da API"
module: offpix/configurations
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - contexts
  - tenant-context
  - tenant-input
  - api-reference
---

# Referência da API de Configurações

Referência completa de endpoints para o módulo de Configurações da W3Block. Este módulo abrange três recursos relacionados — Contexts (definições de formulário), Tenant Contexts (configurações específicas do tenant para esses formulários) e Tenant Inputs (campos individuais de formulário) — todos servidos pelo backend PixwayID.

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

A maioria dos endpoints requer papéis Admin ou SuperAdmin. Endpoints públicos (consultas por slug) são indicados individualmente.

---

## Enums

### ContextType

| Valor | Descrição |
|-------|-----------|
| `user_properties` | Formulário de coleta de dados KYC / perfil do usuário |
| `form` | Formulário genérico (pesquisas, coleta de dados customizada) |

### DataTypesEnum (Tipos de Campo de Entrada)

| Valor | Categoria | Descrição |
|-------|-----------|-----------|
| `text` | Entrada | Campo de texto livre |
| `email` | Entrada | Endereço de email com validação |
| `phone` | Entrada | Número de telefone |
| `cpf` | Entrada | Número do documento CPF brasileiro |
| `url` | Entrada | URL com validação |
| `date` | Entrada | Seletor de data |
| `birthdate` | Entrada | Data de nascimento (pode incluir validação de idade) |
| `user_name` | Entrada | Nome de exibição do usuário |
| `file` | Upload | Upload de arquivo genérico |
| `image` | Upload | Upload de arquivo de imagem |
| `identification_document` | Upload | Documento de identidade governamental (frente/verso) |
| `multiface_selfie` | Upload | Selfie biométrica com documento |
| `simple_select` | Seleção | Dropdown com opções estáticas |
| `dynamic_select` | Seleção | Dropdown com opções dinâmicas |
| `checkbox` | Toggle | Checkbox booleano |
| `simple_location` | Localização | Entrada de endereço / localização |
| `commerce_product` | Commerce | Seletor de produto (requer `data.productIds`) |
| `iframe` | Embed | Conteúdo iframe incorporado |
| `separator` | Layout | Separador visual (não é um campo de dados, ignorado na validação) |

### TenantContextNotificationType

| Valor | Descrição |
|-------|-----------|
| `kyc_approval_request` | Notificação enviada quando um usuário submete KYC para aprovação |
| `kyc_require_user_review` | Notificação pedindo ao usuário para revisar/reenviar |
| `kyc_user_approved` | Notificação enviada ao usuário na aprovação |
| `kyc_admin_approved` | Notificação enviada ao admin na aprovação |
| `kyc_user_rejected` | Notificação enviada ao usuário na rejeição |
| `kyc_admin_rejected` | Notificação enviada ao admin na rejeição |

---

## Entidades e Relacionamentos

```
Context (1) ──→ (N) TenantContext ──→ (N) TenantInput
   │                     │
   │                     └── Vincula um Context a um Tenant com configuração (aprovação, notificações)
   │
   └── Definição mestre do formulário (slug, type, maxSubmissions)

TenantInput: Campo individual do formulário dentro de um TenantContext (label, type, order, mandatory)
```

**Conceitos-chave:**
- Um **Context** é um modelo de formulário reutilizável (ex: "signup", "kyc-verification")
- Um **Tenant Context** vincula um Context a um tenant específico, adicionando configurações como aprovação automática, layout de tela e configurações de notificação
- **Tenant Inputs** são os campos/perguntas individuais dentro de um formulário, ordenados por `order` e opcionalmente agrupados por `step`
- Contexts podem ser **globais** (`tenantId = null`) ou **específicos do tenant**

---

## Endpoints: Contexts (Apenas SuperAdmin)

Caminho base: `/contexts`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/contexts` | Bearer | SuperAdmin | Criar uma nova definição de context |
| 2 | GET | `/contexts` | Bearer | SuperAdmin, Admin | Listar todos os contexts (global + tenant) |
| 3 | PATCH | `/contexts/:id` | Bearer | SuperAdmin | Atualizar um context |
| 4 | DELETE | `/contexts/:id` | Bearer | SuperAdmin | Exclusão suave de um context |

### POST /contexts

Criar uma nova definição mestre de context.

**Requisição:**
```json
{
  "description": "KYC Verification",
  "slug": "kyc-verification",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `description` | string | Sim | — | Nome legível por humanos |
| `slug` | string | Sim | — | Identificador seguro para URL. Minúsculas, alfanumérico + hífens. Deve corresponder a `^[a-z0-9]+(?:-[a-z0-9]+)*$` |
| `type` | ContextType | Não | `user_properties` | Tipo do formulário |
| `maxSubmissions` | integer | Não | `1` | Máximo de vezes que um usuário pode submeter este formulário (mín: 1) |
| `isPreOrder` | boolean | Não | `false` | Se é um context de pré-pedido |
| `tenantId` | UUID | Não | `null` | Se null, o context é global |

**Resposta (201):**
```json
{
  "id": "ctx-uuid",
  "description": "KYC Verification",
  "slug": "kyc-verification",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "tenantId": null,
  "createdAt": "2026-03-31T10:00:00Z",
  "updatedAt": "2026-03-31T10:00:00Z"
}
```

**Erros:**
| Status | Erro | Causa |
|--------|------|-------|
| 409 | DuplicatedContextException | Combinação slug + tenantId já existe |

---

## Endpoints: Tenant Contexts

Caminho base: `/tenants/:tenantId/tenant-context`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/tenants/:tenantId/tenant-context` | Bearer | Admin, Integration | Criar associação tenant-context |
| 2 | GET | `/tenants/:tenantId/tenant-context` | Bearer | Admin, User | Listar tenant contexts (paginado, filtrável) |
| 3 | GET | `/tenants/:tenantId/tenant-context/:tenantContextId` | Bearer | Admin, User | Obter um tenant context |
| 4 | GET | `/tenants/:tenantId/tenant-context/slug/:slug` | Bearer | Admin, User | Obter tenant context por slug |
| 5 | PATCH | `/tenants/:tenantId/tenant-context/:tenantContextId` | Bearer | Admin | Atualizar tenant context |
| 6 | GET | `/tenants/:tenantId/tenant-context/:tenantContextId/approvers` | Bearer | Admin, User | Listar aprovadores KYC do context |

### POST /tenants/:tenantId/tenant-context

Vincular um context existente a um tenant.

**Requisição Mínima:**
```json
{
  "contextId": "ctx-uuid",
  "active": true
}
```

**Requisição Completa:**
```json
{
  "contextId": "ctx-uuid",
  "active": true,
  "data": {
    "requireSpecificApprover": true,
    "alternativeSignUp": false,
    "screenConfig": {
      "postKycUrl": "https://example.com/success",
      "skipConfirmation": false,
      "position": "center",
      "textContainer": {
        "mainText": "Complete your profile",
        "subtitleText": "Enter your information",
        "auxiliarText": "This is required"
      }
    },
    "profileScreen": {
      "hidden": false
    }
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `contextId` | UUID | Sim | ID do context a associar |
| `active` | boolean | Sim | Se este formulário está atualmente ativo |
| `data` | object | Não | Configuração específica do tenant (layout de tela, regras de aprovação, etc.) |

**Resposta (201):** `TenantContextDto` (ver formato de resposta abaixo)

**Nota:** Se um tenant-context para o mesmo `contextId + tenantId` já existir, este endpoint **atualiza** em vez de criar um duplicado.

### GET /tenants/:tenantId/tenant-context

Listar todos os tenant contexts com paginação e filtros.

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | 1 | Número da página |
| `limit` | integer | 10 | Itens por página |
| `search` | string | — | Buscar em description/slug |
| `sortBy` | string | `createdAt` | Coluna de ordenação |
| `orderBy` | `ASC` \| `DESC` | `DESC` | Direção da ordenação |
| `active` | boolean | — | Filtrar por status ativo |
| `type` | ContextType[] | — | Filtrar por tipo de context |
| `preOrder` | boolean | — | Filtrar por isPreOrder |

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "tc-uuid",
      "contextId": "ctx-uuid",
      "tenantId": "tenant-uuid",
      "active": true,
      "data": { "..." : "..." },
      "autoApprove": false,
      "requireSpecificApprover": false,
      "blockCommerceDeliver": false,
      "approverWhitelistIds": null,
      "context": {
        "id": "ctx-uuid",
        "slug": "signup",
        "description": "SignUp Form",
        "type": "user_properties",
        "maxSubmissions": 1,
        "isPreOrder": false
      },
      "createdAt": "2026-03-31T10:00:00Z",
      "updatedAt": "2026-03-31T10:00:00Z"
    }
  ],
  "meta": {
    "totalItems": 1,
    "totalPages": 1,
    "currentPage": 1,
    "itemsPerPage": 10
  }
}
```

### GET /tenants/:tenantId/tenant-context/slug/:slug

Buscar um tenant context pelo slug do context subjacente. Útil para formulários conhecidos como `signup` ou `admin-notification-settings`.

**Resposta (200):** Um único `TenantContextDto` com `context` aninhado.

### GET /tenants/:tenantId/tenant-context/:tenantContextId/approvers

Retorna a lista de usuários com papel `kycApprover` para este context. Com cache de 300 segundos.

**Resposta (200):**
```json
{
  "approvers": [
    {
      "id": "user-uuid",
      "email": "approver@example.com",
      "name": "John Doe"
    }
  ]
}
```

**Erros:**
| Status | Erro | Causa |
|--------|------|-------|
| 404 | TenantContextNotFoundException | Context não encontrado |
| 400 | TenantContextDisabledException | Context não está ativo |
| 400 | TenantContextCantHaveApproversException | Context tem `autoApprove = true` |

---

## Endpoints: Admin Tenant Contexts

Caminho base: `/tenants/:tenantId/admin/tenant-context`

Estes endpoints criam/atualizam tanto o **context** quanto o **tenant-context** atomicamente em uma única transação.

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/tenants/:tenantId/admin/tenant-context` | Bearer | Admin, Integration | Criar context + tenant-context em um passo |
| 2 | PATCH | `/tenants/:tenantId/admin/tenant-context/:tenantContextId` | Bearer | Admin | Atualizar context + tenant-context atomicamente |
| 3 | GET | `/tenants/:tenantId/admin/tenant-context/:tenantContextId/inputs` | Bearer | Admin | Listar inputs de um tenant context |

### POST /tenants/:tenantId/admin/tenant-context

Cria tanto o Context subjacente quanto o TenantContext em uma única transação. Este é o endpoint que o frontend usa ao criar novos formulários.

**Requisição Mínima:**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "active": true
}
```

**Requisição Completa (exemplo de produção do frontend):**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": true,
  "data": {
    "requireSpecificApprover": true,
    "alternativeSignUp": false,
    "profileScreen": {
      "hidden": false
    },
    "screenConfig": {
      "postKycUrl": "https://example.com/verify-success",
      "skipConfirmation": false,
      "position": "left",
      "textContainer": {
        "mainText": "Verify Your Identity",
        "subtitleText": "Upload your documents",
        "auxiliarText": "This step is mandatory"
      }
    }
  }
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `slug` | string | Sim | — | Identificador seguro para URL (minúsculas, alfanumérico + hífens) |
| `description` | string | Sim | — | Nome legível do formulário |
| `type` | ContextType | Não | `user_properties` | Tipo do formulário |
| `maxSubmissions` | integer | Não | `1` | Máximo de submissões do usuário |
| `isPreOrder` | boolean | Não | `false` | Flag de pré-pedido |
| `active` | boolean | Sim | — | Se o formulário está ativo |
| `data` | object | Não | `null` | Configuração de tela, regras de aprovação (ver schema de data abaixo) |

**Schema do objeto `data` (nome no frontend → campo da API):**

| Nome no Frontend | Caminho da API | Tipo | Descrição |
|------------------|----------------|------|-----------|
| Exigir Aprovador Específico | `data.requireSpecificApprover` | boolean | Exigir um aprovador KYC específico |
| Cadastro Alternativo | `data.alternativeSignUp` | boolean | Habilitar fluxo de cadastro alternativo |
| Ocultar Tela de Perfil | `data.profileScreen.hidden` | boolean | Pular etapa de perfil no formulário |
| URL de Redirecionamento (Pós-KYC) | `data.screenConfig.postKycUrl` | string | URL para redirecionar após conclusão do formulário |
| Pular Confirmação | `data.screenConfig.skipConfirmation` | boolean | Pular etapa de confirmação |
| Posição do Formulário | `data.screenConfig.position` | `"left"` \| `"center"` | Alinhamento do formulário na tela |
| Texto Principal | `data.screenConfig.textContainer.mainText` | string | Título do formulário |
| Subtítulo | `data.screenConfig.textContainer.subtitleText` | string | Subtítulo do formulário |
| Texto de Ajuda | `data.screenConfig.textContainer.auxiliarText` | string | Instruções adicionais |

### PATCH /tenants/:tenantId/admin/tenant-context/:tenantContextId

Mesmo corpo que POST (exceto que `type` não pode ser alterado). Atualiza tanto context quanto tenant-context atomicamente.

**Nota:** Apenas SuperAdmin pode atualizar contexts **globais** (tenantId = null). Tentar atualizar um context global como Admin lança `UnableToChangeGlobalContextException` (403).

### GET /tenants/:tenantId/admin/tenant-context/:tenantContextId/inputs

Listar todos os inputs pertencentes a este tenant context.

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | 1 | Número da página |
| `limit` | integer | 10 | Itens por página |
| `search` | string | — | Buscar em label/description |

**Resposta (200):** Lista paginada de `TenantInputEntityDto`.

---

## Endpoints: Tenant Inputs (Campos de Formulário)

Caminho base: `/tenants/:tenantId/tenant-input`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/tenants/:tenantId/tenant-input` | Bearer | SuperAdmin, Admin, Integration | Criar um novo campo de formulário |
| 2 | PATCH | `/tenants/:tenantId/tenant-input/:inputId` | Bearer | SuperAdmin, Admin, Integration | Atualizar um campo de formulário |
| 3 | GET | `/tenants/:tenantId/tenant-input/slug/:slug` | **None (Público)** | — | Obter inputs ativos por slug do context |
| 4 | GET | `/tenants/:tenantId/tenant-input` | Bearer | SuperAdmin, Admin, Integration | Listar inputs (paginado) |
| 5 | GET | `/tenants/:tenantId/tenant-input/:inputId` | Bearer | SuperAdmin, Admin, Integration | Obter um input |

### POST /tenants/:tenantId/tenant-input

Criar um novo campo de formulário dentro de um context.

**Requisição Mínima:**
```json
{
  "contextId": "ctx-uuid",
  "label": "Email Address",
  "description": "Enter your email",
  "type": "email",
  "order": 1,
  "mandatory": true,
  "active": true
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "contextId": "ctx-uuid",
  "label": "Email Address",
  "description": "Enter your email address",
  "type": "email",
  "order": 1,
  "mandatory": true,
  "active": true,
  "attributeName": "userEmail",
  "step": 1,
  "options": null,
  "data": null
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `contextId` | UUID | Sim | — | Context ao qual este input pertence |
| `label` | string | Sim | — | Rótulo do campo exibido ao usuário |
| `description` | string | Sim | — | Texto de ajuda / placeholder do campo |
| `type` | DataTypesEnum | Sim | — | Tipo do campo (ver enum acima) |
| `order` | integer | Sim | — | Ordem de exibição (crescente) |
| `mandatory` | boolean | Sim | — | Se este campo é obrigatório |
| `active` | boolean | Sim | — | Se este campo está atualmente visível |
| `attributeName` | string | Não | Mesmo que `type` | Chave de atributo customizada para o valor submetido |
| `step` | number | Não | `null` | Número da etapa/página do formulário para formulários multi-etapas |
| `options` | array | Não | `[]` | Para `simple_select` / `dynamic_select`: `[{ "label": "Option A", "value": "a" }]` |
| `data` | object | Não | `null` | Configuração específica do tipo (ex: `{ "productIds": ["uuid"] }` para `commerce_product`) |

**Efeitos colaterais:** Se o novo input for `mandatory` e `active`, todos os usuários existentes são sinalizados para revisão através do sistema users-contexts.

### PATCH /tenants/:tenantId/tenant-input/:inputId

Atualizar um campo de formulário existente. Não é possível alterar `contextId` ou `type`.

**Efeitos colaterais:** Mesmo que POST — se `mandatory` ou `active` mudar para `mandatory + active`, os usuários são sinalizados para revisão.

### GET /tenants/:tenantId/tenant-input/slug/:slug (Público)

Retorna todos os inputs **ativos** para um context identificado por slug, ordenados por `order ASC`. Este é o endpoint usado para renderizar formulários para usuários finais.

**Resposta (200):**
```json
[
  {
    "id": "input-uuid-1",
    "contextId": "ctx-uuid",
    "tenantId": "tenant-uuid",
    "label": "Full Name",
    "description": "Enter your full name",
    "type": "text",
    "order": 1,
    "mandatory": true,
    "active": true,
    "attributeName": "fullName",
    "step": 1,
    "options": [],
    "data": null,
    "createdAt": "2026-03-31T10:00:00Z",
    "updatedAt": "2026-03-31T10:00:00Z"
  },
  {
    "id": "input-uuid-2",
    "contextId": "ctx-uuid",
    "tenantId": "tenant-uuid",
    "label": "Email Address",
    "description": "Enter your email",
    "type": "email",
    "order": 2,
    "mandatory": true,
    "active": true,
    "attributeName": "email",
    "step": 1,
    "options": [],
    "data": null,
    "createdAt": "2026-03-31T10:00:00Z",
    "updatedAt": "2026-03-31T10:00:00Z"
  }
]
```

**Erros:**
| Status | Erro | Causa |
|--------|------|-------|
| 404 | ContextBySlugNotFoundException | Nenhum context com este slug |
| 404 | TenantContextNotFoundException | Context existe mas não está vinculado a este tenant |
| 400 | TenantContextDisabledException | Tenant context está inativo |

---

## Validação Específica por Tipo: commerce_product

Ao criar/atualizar um input com `type: "commerce_product"`, o campo `data` é validado:

```json
{
  "data": {
    "productIds": ["product-uuid-1", "product-uuid-2"]
  }
}
```

- `productIds` deve ser um array não vazio de UUIDs válidos
- Cada ID de produto é verificado no serviço Commerce — IDs inválidos lançam `InvalidTenantDataException`
