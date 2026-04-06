---
id: FLOW_CONFIGURATIONS_INPUT_MANAGEMENT
title: "Configurações - Gerenciamento de Input (Campo de Formulário)"
module: offpix/configurations
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - tenant-input
  - form-fields
  - kyc
depends_on:
  - CONFIGURATIONS_API_REFERENCE
  - FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
---

# Gerenciamento de Input (Campo de Formulário)

## Visão Geral

Inputs são os campos individuais dentro de um formulário de context — campos de texto, uploads de arquivo, selects, solicitações de documentos, etc. Este fluxo cobre a criação, atualização, ordenação e recuperação de campos de formulário. No frontend, isso é gerenciado em **KYC > Form Designer > [Context] > Inputs** (`/dash/kyc/inputs?id={contextId}`).

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de auth / configuração do ambiente |
| `contextId` | UUID do context ao qual adicionar campos | [Ciclo de Vida do Context](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

## Entidades

```
TenantInput
├── label         (nome de exibição mostrado ao usuário)
├── description   (texto de ajuda / placeholder)
├── type          (DataTypesEnum: text, email, file, etc.)
├── order         (integer: sequência de exibição)
├── mandatory     (boolean: este campo é obrigatório?)
├── active        (boolean: este campo é exibido?)
├── step          (number: agrupamento por página/etapa do formulário)
├── attributeName (chave personalizada para o valor enviado)
├── options       (array: para campos do tipo select)
└── data          (object: configuração específica do tipo)
```

**19 tipos de input** em 5 categorias:

| Categoria | Tipos |
|-----------|-------|
| **Entrada de texto** | `text`, `email`, `phone`, `cpf`, `url`, `user_name` |
| **Data** | `date`, `birthdate` |
| **Upload de arquivo** | `file`, `image`, `identification_document`, `multiface_selfie` |
| **Seleção** | `simple_select`, `dynamic_select`, `checkbox` |
| **Especial** | `simple_location`, `commerce_product`, `iframe`, `separator` |

---

## Fluxo: Adicionar Campos a um Formulário

### Etapa 1: Criar um Campo de Formulário

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/tenants/{tenantId}/tenant-input` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "contextId": "ctx-uuid",
  "label": "Full Name",
  "description": "Enter your full legal name",
  "type": "text",
  "order": 1,
  "mandatory": true,
  "active": true
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "contextId": "ctx-uuid",
  "label": "Full Name",
  "description": "Enter your full legal name",
  "type": "text",
  "order": 1,
  "mandatory": true,
  "active": true,
  "attributeName": "fullName",
  "step": 1,
  "options": null,
  "data": null
}
```

**Resposta (201):**
```json
{
  "id": "input-uuid",
  "contextId": "ctx-uuid",
  "tenantId": "tenant-uuid",
  "label": "Full Name",
  "description": "Enter your full legal name",
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
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `contextId` | UUID | Sim | — | Context ao qual este campo pertence |
| `label` | string | Sim | — | Rótulo do campo exibido ao usuário |
| `description` | string | Sim | — | Texto de ajuda ou placeholder |
| `type` | DataTypesEnum | Sim | — | Tipo do campo (não pode ser alterado após a criação) |
| `order` | integer | Sim | — | Ordem de exibição (baseada em 1, ascendente) |
| `mandatory` | boolean | Sim | — | Se o campo é obrigatório para o envio do formulário |
| `active` | boolean | Sim | — | Se o campo está atualmente visível |
| `attributeName` | string | Não | Mesmo que `type` | Chave personalizada para o valor enviado. Use isto para dar nomes significativos às respostas |
| `step` | number | Não | `null` | Agrupar campos em etapas/páginas do formulário |
| `options` | array | Não | `[]` | Para tipos select: `[{ "label": "Exibição", "value": "armazenado" }]` |
| `data` | object | Não | `null` | Configuração específica do tipo |

**Observações:**
- Se `attributeName` não for fornecido, o padrão será o valor de `type` (ex.: "email", "text"). Defina um valor personalizado para distinguir múltiplos campos do mesmo tipo
- Quando um input **obrigatório + ativo** é criado, todos os usuários existentes são sinalizados para revisão no sistema de users-contexts. Isso pode disparar notificações de reenvio de KYC

### Etapa 2: Adicionar Mais Campos

Repita a Etapa 1 para cada campo, incrementando o valor de `order`:

```
Order 1: Full Name (text, obrigatório)
Order 2: Email (email, obrigatório)
Order 3: Phone (phone, opcional)
Order 4: ID Document Front (identification_document, obrigatório)
Order 5: Selfie (multiface_selfie, obrigatório)
```

---

## Exemplos Específicos por Tipo

### Campo Select (Dropdown)

```json
{
  "contextId": "ctx-uuid",
  "label": "Country",
  "description": "Select your country",
  "type": "simple_select",
  "order": 3,
  "mandatory": true,
  "active": true,
  "attributeName": "country",
  "options": [
    { "label": "Brazil", "value": "BR" },
    { "label": "United States", "value": "US" },
    { "label": "Portugal", "value": "PT" }
  ]
}
```

### Seletor de Produto de Comércio

```json
{
  "contextId": "ctx-uuid",
  "label": "Select a Plan",
  "description": "Choose your subscription plan",
  "type": "commerce_product",
  "order": 6,
  "mandatory": true,
  "active": true,
  "attributeName": "selectedPlan",
  "data": {
    "productIds": ["product-uuid-1", "product-uuid-2"]
  }
}
```

**Validação:** Cada ID de produto em `productIds` é verificado no serviço Commerce. IDs inválidos lançam `InvalidTenantDataException` (400).

### Upload de Documento

```json
{
  "contextId": "ctx-uuid",
  "label": "Government ID",
  "description": "Upload front and back of your ID",
  "type": "identification_document",
  "order": 4,
  "mandatory": true,
  "active": true,
  "attributeName": "govId"
}
```

### Separador (Elemento de Layout)

```json
{
  "contextId": "ctx-uuid",
  "label": "--- Documents Section ---",
  "description": "Upload the following documents",
  "type": "separator",
  "order": 3,
  "mandatory": false,
  "active": true
}
```

**Observação:** Separadores são ignorados durante a validação do formulário — são elementos apenas visuais.

---

## Fluxo: Atualizar um Campo

### Etapa 1: Atualizar o Input

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/tenants/{tenantId}/tenant-input/{inputId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "label": "Updated Label",
  "description": "Updated description",
  "order": 2,
  "mandatory": false,
  "active": true,
  "attributeName": "updatedAttr",
  "step": 2,
  "options": null,
  "data": null
}
```

**Observações:**
- `contextId` e `type` **não podem ser alterados** após a criação
- Se a atualização tornar um campo `mandatory + active` (e não era antes), todos os usuários existentes são sinalizados para revisão

---

## Fluxo: Recuperar Campos do Formulário (Público)

Este endpoint é usado para renderizar formulários para usuários finais. Ele retorna apenas campos **ativos** ordenados por `order ASC`.

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenants/{tenantId}/tenant-input/slug/{slug}` | **Nenhuma (Público)** |

**Resposta (200):**
```json
[
  {
    "id": "input-uuid-1",
    "label": "Full Name",
    "type": "text",
    "order": 1,
    "mandatory": true,
    "active": true,
    "attributeName": "fullName",
    "step": 1,
    "options": [],
    "data": null
  },
  {
    "id": "input-uuid-2",
    "label": "Email Address",
    "type": "email",
    "order": 2,
    "mandatory": true,
    "active": true,
    "attributeName": "email",
    "step": 1,
    "options": [],
    "data": null
  }
]
```

**Observações:**
- Apenas inputs ativos são retornados
- Os resultados são ordenados por `order ASC`
- Se o context ou tenant-context não for encontrado ou estiver inativo, a API retorna erros apropriados (404 ou 400)

---

## Fluxo: Listar Inputs (Admin, Paginado)

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenants/{tenantId}/admin/tenant-context/{tenantContextId}/inputs` | Bearer (Admin) |

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | 1 | Número da página |
| `limit` | integer | 10 | Itens por página (frontend usa 4 no mobile) |
| `search` | string | — | Buscar no label/descrição |

**Uso no frontend:** A página de inputs (`/dash/kyc/inputs?id={contextId}`) busca inputs ordenados por `order ASC` e os renderiza como uma lista editável.

---

## Formulários Multi-Etapas

Agrupe campos em etapas usando o campo `step`:

```
Etapa 1: Informações Pessoais
  ├── Order 1: Full Name (text)
  ├── Order 2: Email (email)
  └── Order 3: Phone (phone)

Etapa 2: Documentos
  ├── Order 4: ID Document (identification_document)
  └── Order 5: Selfie (multiface_selfie)

Etapa 3: Adicional
  └── Order 6: Country (simple_select)
```

O campo `step` é usado para:
- Agrupar campos em páginas/abas do formulário no frontend
- Validar que os documentos obrigatórios existem por etapa (via `checkAllRequiredDocumentsHaveBeenSubmitted`)

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 404 | TenantParamsNotFoundException | Input não encontrado | Verifique o inputId |
| 400 | InvalidTenantDataException | Validação específica do tipo falhou | Verifique o campo `data` para o tipo específico (ex.: productIds válidos para commerce_product) |
| 404 | ContextBySlugNotFoundException | Slug não encontrado (endpoint público) | Verifique a grafia do slug e o tenant |
| 404 | TenantContextNotFoundException | Context não vinculado ao tenant | Crie a associação tenant-context primeiro |
| 400 | TenantContextDisabledException | Context está inativo (endpoint público) | Ative o context |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | `attributeName` assume o padrão da string do tipo | Sempre defina um `attributeName` personalizado quando tiver múltiplos campos do mesmo tipo (ex.: dois campos `text`). Caso contrário, ambos compartilhariam a chave "text" |
| 2 | Adicionar campo obrigatório dispara revisão de usuários | Criar ou atualizar um campo para `mandatory + active` sinaliza todos os usuários existentes para reenvio. Planeje a estrutura do formulário antes de criar inputs |
| 3 | `type` não pode ser alterado após a criação | Se precisar de um tipo diferente, exclua o input antigo e crie um novo |
| 4 | Endpoint público por slug retorna vazio para campos inativos | Apenas inputs ativos são retornados. Campos desativados ficam ocultos para usuários finais, mas ainda visíveis nas views de admin |
| 5 | Validação de produto de comércio falha | Para o tipo `commerce_product`, `data.productIds` deve conter UUIDs válidos de produtos existentes no serviço Commerce |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Ciclo de Vida do Context | Crie o context antes de adicionar inputs | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Ativação de Cadastro | Alternar visibilidade do formulário de cadastro | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) |
| Referência da API | Detalhes completos de endpoints e enums | [CONFIGURATIONS_API_REFERENCE](./CONFIGURATIONS_API_REFERENCE.md) |
