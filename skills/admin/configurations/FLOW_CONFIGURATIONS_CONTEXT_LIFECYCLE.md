---
id: FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
title: "Configurações - Ciclo de Vida do Context"
module: offpix/configurations
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - contexts
  - tenant-context
  - forms
  - kyc
depends_on:
  - AUTH_API_REFERENCE
  - CONFIGURATIONS_API_REFERENCE
---

# Ciclo de Vida do Context

## Visão Geral

Um Context no W3Block representa uma definição de formulário (KYC, pesquisas, formulários de onboarding). Este fluxo cobre a criação, configuração e gerenciamento de contexts pelo painel administrativo. O ciclo de vida completo é: criar um context + associação com tenant (uma única chamada de API) e, em seguida, adicionar campos de formulário (inputs) a ele.

No frontend, este módulo é acessado em **KYC > Form Designer** (`/dash/kyc/form-designer`).

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | Token de acesso JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de auth / configuração do ambiente |
| Role Admin | O usuário deve ter role Admin ou SuperAdmin | Configuração do tenant |

## Entidades e Relacionamentos

```
Context (1) ──→ (N) TenantContext ──→ (N) TenantInput
  │                     │                    │
  │ slug, type,         │ active, data,      │ label, type, order,
  │ maxSubmissions      │ approval config,   │ mandatory, options,
  │                     │ notifications      │ step, attributeName
  │                     │
  │                     └── Vincula context ao tenant com configuração de tela
  │
  └── Definição mestre do formulário (pode ser global ou específica do tenant)
```

**Tipos de Context:**
- `user_properties` — Formulários de KYC / perfil do usuário (exibidos no frontend como formulários de "onboarding")
- `form` — Formulários genéricos (pesquisas, coleta de dados personalizada)

---

## Fluxo A: Criar um Novo Formulário (Context + Associação com Tenant)

O endpoint admin cria tanto o Context quanto o TenantContext em uma única transação. Esta é a abordagem recomendada.

### Etapa 1: Criar Context via Endpoint Admin

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/tenants/{tenantId}/admin/tenant-context` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "active": true
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": true,
  "data": {
    "requireSpecificApprover": false,
    "alternativeSignUp": false,
    "profileScreen": {
      "hidden": false
    },
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": false,
      "position": "center",
      "textContainer": {
        "mainText": "Verify Your Identity",
        "subtitleText": "Upload your documents to continue",
        "auxiliarText": "This process takes about 5 minutes"
      }
    }
  }
}
```

**Resposta (201):**
```json
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
    "slug": "kyc-verification",
    "description": "KYC Verification Form",
    "type": "user_properties",
    "maxSubmissions": 1,
    "isPreOrder": false
  },
  "createdAt": "2026-03-31T10:00:00Z",
  "updatedAt": "2026-03-31T10:00:00Z"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `slug` | string | Sim | — | Identificador seguro para URL (`^[a-z0-9]+(?:-[a-z0-9]+)*$`) |
| `description` | string | Sim | — | Nome do formulário (frontend: título do formulário) |
| `type` | `user_properties` \| `form` | Não | `user_properties` | Tipo do formulário |
| `maxSubmissions` | integer | Não | `1` | Quantas vezes um usuário pode enviar (mín: 1) |
| `isPreOrder` | boolean | Não | `false` | Flag de context de pré-venda |
| `active` | boolean | Sim | — | Se o formulário está habilitado |
| `data` | object | Não | `null` | Configuração de tela e regras de aprovação |

**Observações:**
- O slug deve ser único por tenant. Se um context com o mesmo slug já existir para este tenant, a API lança `DuplicatedContextException` (409)
- Após a criação, o frontend redireciona para a página de inputs para adicionar campos ao formulário
- O objeto `data` é armazenado como JSONB — você pode incluir quaisquer campos personalizados, mas o frontend utiliza especificamente o schema documentado acima

### Etapa 2: Adicionar Campos ao Formulário (Inputs)

Após criar o context, adicione campos a ele. Veja [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md).

---

## Fluxo B: Atualizar um Formulário Existente

### Etapa 1: Buscar o Context Atual

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenants/{tenantId}/tenant-context/{tenantContextId}` | Bearer (Admin) |

**Resposta (200):** `TenantContextDto` completo com objeto `context` aninhado.

### Etapa 2: Atualizar Context + Tenant Context

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/tenants/{tenantId}/admin/tenant-context/{tenantContextId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "slug": "kyc-verification",
  "description": "Updated KYC Form",
  "maxSubmissions": 3,
  "active": true,
  "data": {
    "requireSpecificApprover": true,
    "screenConfig": {
      "postKycUrl": "https://myapp.com/new-welcome",
      "position": "left"
    }
  }
}
```

**Observações:**
- `type` não pode ser alterado após a criação
- Apenas SuperAdmin pode atualizar contexts **globais** (aqueles com `tenantId = null`). Usuários Admin que tentarem isso recebem `UnableToChangeGlobalContextException` (403)
- A atualização é atômica — tanto o context quanto o tenant-context são atualizados em uma única transação

---

## Fluxo C: Listar e Filtrar Contexts

### Listar Todos os Contexts de um Tenant

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenants/{tenantId}/tenant-context` | Bearer (Admin, User) |

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `page` | integer | Número da página (padrão: 1) |
| `limit` | integer | Itens por página (padrão: 10) |
| `active` | boolean | Filtrar por status ativo |
| `type` | ContextType[] | Filtrar por tipo (`user_properties`, `form`) |
| `preOrder` | boolean | Filtrar por flag de pré-venda |
| `search` | string | Buscar na descrição/slug |
| `sortBy` | string | Coluna de ordenação (padrão: `createdAt`) |
| `orderBy` | `ASC` \| `DESC` | Direção de ordenação (padrão: `DESC`) |

**Padrões de uso no frontend:**
- Página Form Designer: busca todos os contexts, ordena o context de cadastro primeiro
- Página KYC Config: filtra especificamente o context de cadastro (`item.context.slug === 'signup'`)
- Página Clientes: filtra `active === true && type === 'user_properties'`

### Buscar por Slug

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenants/{tenantId}/tenant-context/slug/{slug}` | Bearer (Admin, User) |

Slugs comuns usados na plataforma:
- `signup` — o formulário principal de cadastro/registro
- `admin-notification-settings` — context de configuração de notificações

---

## Fluxo D: Configurar o Formulário de Cadastro (Caso Especial)

O formulário de cadastro tem um fluxo dedicado no frontend (`/dash/kyc/signup`). Ele usa o slug `signup` e tipo `user_properties`.

### Etapa 1: Verificar se o Context de Cadastro Existe

```
GET /tenants/{tenantId}/tenant-context?type=user_properties
```

Filtre a resposta por `item.context.slug === 'signup'`.

### Etapa 2a: Criar se Não Existir

```
POST /tenants/{tenantId}/admin/tenant-context
```

```json
{
  "slug": "signup",
  "description": "SignUp Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": false
}
```

### Etapa 2b: Atualizar se Existir

```
PATCH /tenants/{tenantId}/tenant-context/{tenantContextId}
```

```json
{
  "active": true,
  "data": {
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": true,
      "position": "center",
      "textContainer": {
        "mainText": "Welcome!",
        "subtitleText": "Complete your profile to get started"
      }
    }
  }
}
```

**Observação:** A atualização do formulário de cadastro usa o endpoint PATCH não-admin (`/tenant-context/{id}`) que atualiza apenas `active` e `data` — não os campos do context subjacente.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 409 | DuplicatedContextException | Slug + tenantId já existe | Use um slug diferente ou atualize o context existente |
| 404 | TenantContextNotFoundException | Context não encontrado ou não vinculado ao tenant | Verifique o tenantContextId e tenantId |
| 404 | ContextBySlugNotFoundException | Nenhum context com este slug | Verifique a grafia do slug |
| 403 | UnableToChangeGlobalContextException | Admin tentou atualizar um context global | Apenas SuperAdmin pode modificar contexts globais |
| 400 | TenantContextDisabledException | Context está inativo | Ative o context primeiro |
| 400 | TenantContextCantHaveApproversException | Context com auto-aprovação não tem aprovadores | Defina `autoApprove: false` para habilitar aprovadores |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Validação do slug falha | O slug deve ser minúsculo, alfanumérico apenas com hífens. Sem espaços, underscores ou caracteres especiais. Padrão: `^[a-z0-9]+(?:-[a-z0-9]+)*$` |
| 2 | Usando endpoint PATCH errado | O PATCH admin (`/admin/tenant-context/{id}`) atualiza tanto context + tenant-context. O PATCH comum (`/tenant-context/{id}`) atualiza apenas `active` e `data` |
| 3 | Atualização de context global falha para Admin | Contexts globais (tenantId = null) só podem ser modificados por SuperAdmin |
| 4 | Context aparece duplicado | O mesmo Context pode ser vinculado a múltiplos tenants via TenantContext. Cada tenant recebe sua própria configuração |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Auth | Necessário antes de qualquer operação admin | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| Gerenciamento de Inputs | Próximo passo após criar um context | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Ativação de Cadastro | Ativar/desativar formulário de cadastro | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) |
