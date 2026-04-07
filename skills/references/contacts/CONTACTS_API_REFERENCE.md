---
id: CONTACTS_API_REFERENCE
title: "Contatos - Referência da API"
module: offpix/contacts
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contacts
  - users
  - kyc
  - documents
  - royalty
  - api-reference
---

# Referência da API de Contatos

Referência completa de endpoints para o módulo de Contatos da W3Block. Este módulo cobre gerenciamento de usuários (convidar, editar, listar), fluxos de submissão e aprovação de documentos KYC, elegibilidade a royalties e contatos externos. Servido pelo backend PixwayID com alguns endpoints nos backends Commerce/Registry.

## URLs Base

| Ambiente | Serviço | URL |
|----------|---------|-----|
| Produção | PixwayID (usuários, KYC) | `https://pixwayid.w3block.io` |
| Produção | Registry (royalty, contatos externos) | `https://api.w3block.io` |
| Produção | Commerce (provedores de pagamento) | `https://commerce.w3block.io` |
| Staging | PixwayID | *(ambiente de staging disponível — use a URL base de staging)* |
| Swagger | PixwayID | https://pixwayid.w3block.io/docs/ |
| Swagger | Registry | https://api.w3block.io/docs/ |

## Autenticação

Todos os endpoints requerem Bearer token a menos que explicitamente marcados como públicos:

```
Authorization: Bearer {accessToken}
```

---

## Enums

### UserRoleEnum

| Valor | Label no Frontend | Descrição |
|-------|-------------------|-----------|
| `superAdmin` | — | Super administrador em nível de plataforma |
| `admin` | Admin | Administrador do tenant |
| `operator` | Operador | Operador do tenant |
| `user` | Cliente | Usuário regular / cliente |
| `loyaltyOperator` | Operador de Fidelidade | Operador do programa de fidelidade |
| `commerceOrderReceiver` | Receptor ERC | Receptor de pedidos do commerce |
| `kycApprover` | — | Operador de aprovação KYC |
| `keyErc20Receiver` | — | Receptor de tokens ERC-20 |

### UserContextStatus (Status KYC)

| Valor | Label no Frontend | Cor | Descrição |
|-------|-------------------|-----|-----------|
| `APPROVED` | Aprovado | Verde | Todos os documentos aprovados |
| `DENIED` | Negado | Vermelho | Documentos rejeitados |
| `REQUIRED_REVIEW` | Revisão Necessária | Laranja | Aprovador solicitou reenvio de campos específicos |
| `CREATED` | Criado | Laranja | Todos os docs submetidos, aguardando aprovação |
| `DRAFT` | Rascunho | — | Submissão em andamento (incompleta) |

### KYCStatusType (Específico do frontend, de customer-infos)

| Valor | Cor | Descrição |
|-------|-----|-----------|
| `approved` | Verde | KYC aprovado |
| `denied` | Vermelho | KYC negado |
| `waitingInfo` | Laranja | Aguardando informações |
| `missingDocuments` | Laranja | Documentos obrigatórios ausentes |
| `processing` | Laranja | Processando |
| `created` | Laranja | Aguardando revisão |

### ContactTypes (Navegação do frontend)

| Valor | Rota do Frontend | Papéis Filtrados |
|-------|------------------|------------------|
| `clients` | `/dash/contacts/clients` | `user` |
| `team` | `/dash/contacts/team` | `admin`, `operator`, `loyaltyOperator`, `commerceOrderReceiver` |
| `partiner` | `/dash/contacts/partiners` | Parceiros/associados |
| `external` | — | Contatos de carteira externos (sem conta de usuário) |

---

## Entidades e Relacionamentos

```
User (1) ──→ (N) UsersContext ──→ (N) UsersDocument
  │                  │                     │
  │ email, name,     │ status, logs,       │ inputId, value,
  │ phone, roles,    │ approverUserId,     │ assetId, status
  │ wallets          │ utmParams           │
  │                  │                     │
  │                  └── Vincula usuário ao context com status de aprovação
  │                       (DRAFT → CREATED → APPROVED/DENIED)
  │
  ├──→ (N) Wallet (mainWallet + adicionais)
  └──→ (1) RoyaltyEligible (opcional)

TenantContext define o formulário → TenantInput define os campos
UsersContext rastreia a submissão de um usuário → UsersDocument armazena o valor de cada campo
```

---

## Endpoints: Gerenciamento de Usuários

### GET /users — Listar Usuários por Tenant

**Caminho:** `GET /{tenantId}/users` (via SDK: `sdk.api.users.getUsersByTenantId`)
**Auth:** Bearer (Admin)

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `role` | UserRoleEnum[] | `['user']` | Filtrar por papel(éis) |
| `orderBy` | `ASC` \| `DESC` | `DESC` | Direção da ordenação |
| `sortBy` | string | `createdAt` | Coluna de ordenação |
| `search` | string | — | Buscar em email/nome |
| `limit` | integer | `10` | Itens por página |
| `page` | integer | `1` | Número da página |

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "user-uuid",
      "tenantId": "tenant-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+5511999999999",
      "roles": ["user"],
      "verified": true,
      "verifiedEmailAt": "2026-01-01T00:00:00Z",
      "mainWalletId": "wallet-uuid",
      "mainWallet": {
        "id": "wallet-uuid",
        "address": "0x1234...abcd",
        "type": "vault",
        "status": "active"
      },
      "wallets": [],
      "createdAt": "2026-01-01T00:00:00Z",
      "updatedAt": "2026-01-01T00:00:00Z"
    }
  ],
  "meta": {
    "totalItems": 100,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 10,
    "currentPage": 1
  }
}
```

### GET /users/:id — Obter Perfil do Usuário

**Caminho:** `GET /{tenantId}/users/{userId}` (via SDK: `sdk.api.users.getProfileUserById`)
**Auth:** Bearer (Admin, User — usuário pode buscar apenas o próprio perfil)

**Resposta (200):** Mesmo formato do item da lista, com detalhes completos da carteira.

### POST /users/invite — Convidar Novo Usuário

**Caminho:** `POST /users/invite` (via SDK: `sdk.api.users.invite`)
**Auth:** Bearer (Admin)

**Requisição:**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "user",
  "royaltyEligible": false,
  "tenantId": "tenant-uuid",
  "i18nLocale": "pt-br",
  "generateRandomPassword": true,
  "sendEmail": true
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome completo do usuário |
| `email` | string | Sim | Email do usuário (deve ser único por tenant) |
| `role` | UserRoleEnum | Sim | Papel a atribuir |
| `tenantId` | UUID | Sim | Tenant de destino |
| `i18nLocale` | string | Não | Localidade para emails (`pt-br`, `en`) |
| `generateRandomPassword` | boolean | Sim | Sempre `true` — gera senha temporária |
| `sendEmail` | boolean | Sim | Sempre `true` — envia email de convite |

**Resposta (201):** Objeto de usuário com `id`.

**Erros:**
| Status | Erro | Causa |
|--------|------|-------|
| 409 | Conflito | Email já em uso para este tenant |

### PATCH /users/:id — Editar Usuário

**Caminho:** `PATCH /{tenantId}/users/{userId}` (via SDK: `sdk.api.users.update`)
**Auth:** Bearer (Admin)

**Requisição:**
```json
{
  "name": "Updated Name",
  "phone": "+5511888888888",
  "roles": ["user", "operator"]
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Não | Nome atualizado |
| `phone` | string | Não | Telefone atualizado |
| `roles` | UserRoleEnum[] | Não | Papéis atualizados |

### GET /users/find-by-any — Buscar Usuário por CPF ou ID

**Caminho:** `GET /{tenantId}/users/documents/find-user-by-any`
**Auth:** Bearer (Admin, Integration)

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `userId` | UUID | Buscar por ID do usuário |
| `cpf` | string | Buscar por CPF (11 dígitos, checksum válido) |

Um dos campos `userId` ou `cpf` é obrigatório. A busca por CPF pesquisa nos documentos KYC submetidos.

### GET /users/by-wallet — Buscar Usuário por Endereço de Carteira

**Caminho:** `GET /{tenantId}/users/wallets/by-address/{address}` (via SDK)
**Auth:** Bearer (Admin)

---

## Endpoints: Contextos de Usuário KYC

Caminho base: `/{tenantId}/users/contexts`

### GET /users/contexts/find — Listar Todas as Submissões de Contexto de Usuário

**Auth:** Bearer (Admin, User, KycApprover)

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | `1` | Número da página |
| `limit` | integer | `10` | Itens por página |
| `search` | string | — | Buscar em email/nome do usuário |
| `sortBy` | array | `[{column: 'createdAt', order: 'ASC'}]` | Configuração de ordenação |
| `userId[]` | UUID[] | — | Filtrar por IDs de usuário (apenas Admin) |
| `status[]` | UserContextStatus[] | — | Filtrar por status |
| `contextId[]` | UUID[] | — | Filtrar por IDs de context |
| `contextType[]` | ContextType[] | — | Filtrar por `user_properties` ou `form` |
| `preOrder` | boolean | — | Filtrar contexts de pré-pedido |
| `utmCampaign` | string | — | Filtrar por campanha UTM |
| `utmSource` | string | — | Filtrar por fonte UTM |
| `excludeSelfContexts` | boolean | — | KycApprover: excluir próprias submissões |

**Resposta (200):** Lista paginada de `UsersContextsEntity` com relações de usuário e context.

### GET /users/contexts/:userId — Obter Contexts do Usuário

**Auth:** Bearer (Admin, User — usuário pode ver apenas os próprios)

**Parâmetros de Query:** `page`, `limit`, `search`, `sortBy`, `status`, `contextId`

### GET /users/contexts/:userId/:userContextId — Obter Context Específico com Documentos

**Auth:** Bearer (Admin, User, KycApprover)

**Resposta (200):**
```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "CREATED",
  "approverUserId": null,
  "logs": [
    {
      "moderatorId": "admin-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "utmParams": null,
  "documents": [
    {
      "id": "doc-uuid",
      "inputId": "input-uuid",
      "status": "CREATED",
      "simpleValue": "john@example.com",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid",
        "label": "Email Address",
        "type": "email",
        "mandatory": true
      },
      "asset": null
    }
  ],
  "context": {
    "id": "ctx-uuid",
    "slug": "signup",
    "description": "SignUp Form",
    "type": "user_properties"
  },
  "user": {
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

---

## Endpoints: Submissão de Documentos KYC

### POST /users/documents/:userId/context/:contextId — Submeter Documentos

**Caminho:** `POST /{tenantId}/users/documents/{userId}/context/{contextId}`
**Auth:** Bearer (User, Admin, KycApprover)

**Requisição:**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-name",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-email",
      "value": "john@example.com"
    },
    {
      "inputId": "input-uuid-cpf",
      "value": "12345678901"
    },
    {
      "inputId": "input-uuid-file",
      "assetId": "asset-uuid"
    },
    {
      "inputId": "input-uuid-location",
      "value": {
        "region": "SP",
        "country": "BR",
        "placeId": "ChIJ...",
        "city": "Sao Paulo",
        "postal_code": "01000-000",
        "street_address_1": "Av Paulista"
      }
    }
  ],
  "currentStep": 1,
  "approverUserId": "approver-uuid",
  "userContextId": "existing-uc-uuid",
  "utmParams": {
    "utm_campaign": "onboarding",
    "utm_source": "email"
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `documents` | DocumentDto[] | Sim (mín: 1) | Array de submissões de campos |
| `documents[].inputId` | UUID | Sim | ID do TenantInput ao qual este valor pertence |
| `documents[].value` | string \| object | Condicional | Valor do campo (texto, objeto para tipos complexos) |
| `documents[].assetId` | UUID | Condicional | ID do asset do arquivo (para uploads de arquivo/imagem/documento) |
| `currentStep` | number | Não | Etapa atual do formulário (para validação multi-etapas) |
| `approverUserId` | UUID | Condicional | Obrigatório se o context tiver `requireSpecificApprover: true` |
| `userContextId` | UUID | Não | Anexar a context de usuário existente (para reenvio) |
| `utmParams` | object | Não | Parâmetros de rastreamento UTM |

**Resposta (201):** `UsersContextsEntity` (criado ou atualizado).

**Formatos de valor por tipo de entrada:**

| Tipo de Entrada | Formato do Valor | Exemplo |
|-----------------|-----------------|---------|
| `text`, `email`, `url`, `user_name` | string | `"john@example.com"` |
| `cpf` | string (11 dígitos) | `"12345678901"` |
| `phone` | string | `"+5511999999999"` |
| `date`, `birthdate` | string de data ISO | `"1990-05-15"` |
| `simple_select` | string (deve estar nas opções) | `"BR"` |
| `checkbox` | boolean | `true` |
| `file`, `image`, `multiface_selfie` | — (usar `assetId`) | — |
| `identification_document` | object | `{"docType": "RG", "document": "123456789"}` |
| `simple_location` | object | `{"region": "SP", "country": "BR", "placeId": "..."}` |
| `commerce_product` | object | `{"productId": "uuid", "quantity": "1"}` |
| `separator` | — (ignorado) | — |

### GET /users/documents/:userId — Obter Documentos do Usuário

**Caminho:** `GET /{tenantId}/users/documents/{userId}`
**Auth:** Bearer (Admin, User)

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `page` | integer | Número da página |
| `limit` | integer | Itens por página |
| `type[]` | DataTypesEnum[] | Filtrar por tipos de entrada |
| `contextId` | UUID | Filtrar por context |
| `inputId` | UUID | Filtrar por input específico |

### GET /users/documents/:userId/context/:contextId — Obter Documentos por Context

**Caminho:** `GET /{tenantId}/users/documents/{userId}/context/{contextId}`
**Auth:** Bearer (Admin, User)

Retorna todos os documentos para um context específico (array não paginado).

---

## Endpoints: Ações de Aprovação KYC

### PATCH /users/contexts/:userId/:contextId/approve

**Auth:** Bearer (Admin, KycApprover)

**Requisição:**
```json
{
  "reason": "Documents verified successfully"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da aprovação (registrado na trilha de auditoria) |
| `userContextId` | UUID | Não | Context de usuário específico a aprovar |

**Resposta:** 204 No Content

**Efeitos colaterais:**
- Todos os documentos definidos como `APPROVED`
- Status do context do usuário → `APPROVED`
- Entrada de log adicionada com ID do moderador, timestamp, motivo
- Email de aprovação KYC enviado ao usuário
- Serviço Commerce notificado (pode desbloquear pedidos)
- Webhook `USER_KYC_PROPS_UPDATED` despachado

### PATCH /users/contexts/:userId/:contextId/reject

**Auth:** Bearer (Admin, KycApprover)

**Requisição:**
```json
{
  "reason": "Document image is blurry, please resubmit"
}
```

**Resposta:** 204 No Content

**Efeitos colaterais:**
- Todos os documentos definidos como `DENIED`
- Status do context do usuário → `DENIED`
- Email de rejeição KYC enviado
- Serviço Commerce notificado

### PATCH /users/contexts/:userId/:contextId/require-review

**Auth:** Bearer (Admin, KycApprover)

**Requisição:**
```json
{
  "inputIds": ["input-uuid-1", "input-uuid-2"],
  "reason": "ID document is expired, selfie is blurry"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `inputIds` | UUID[] | Sim (mín: 1) | IDs de input que precisam de reenvio |
| `reason` | string | Não | Explicação para o usuário |

**Resposta:** 204 No Content

**Efeitos colaterais:**
- Documentos especificados definidos como `REQUIRED_REVIEW`
- Status do context do usuário → `REQUIRED_REVIEW`
- Email de solicitação de revisão KYC enviado
- Usuário pode reenviar apenas os campos sinalizados

---

## Endpoints: Customer Infos (Lista KYC)

### GET /customer-infos/:tenantId/search — Listar Submissões KYC

**Auth:** Bearer (Admin)

**Parâmetros de Query:** `page`, `limit`, `sortBy`, `orderBy`

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "ci-uuid",
      "customerId": "user-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "status": "approved",
      "tenantId": "tenant-uuid",
      "createdAt": "2026-01-15T10:00:00Z",
      "updatedAt": "2026-01-15T10:00:00Z"
    }
  ]
}
```

---

## Endpoints: Elegibilidade a Royalty

Caminho base: `/{companyId}/contracts/royalty-eligible` (backend Registry)

### GET / — Listar Contatos Elegíveis a Royalty

**Auth:** Bearer (Admin)

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `userIds` | string | — | IDs de usuário separados por vírgula |
| `externalContactIds` | string | — | IDs de contato externo separados por vírgula |
| `wallets` | boolean | `true` | Incluir informações de carteira |
| `search` | string | — | Busca |
| `limit` | integer | `50` | Itens por página |

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "re-uuid",
      "companyId": "company-uuid",
      "active": true,
      "displayName": "John Doe",
      "userId": "user-uuid",
      "externalContactId": "ext-uuid",
      "walletAddress": "0x1234...abcd",
      "createdAt": "2026-01-01T00:00:00Z",
      "updatedAt": "2026-01-01T00:00:00Z"
    }
  ]
}
```

### POST /create — Criar Registro de Elegibilidade a Royalty

**Auth:** Bearer (Admin)

**Requisição:**
```json
{
  "active": true,
  "displayName": "John Doe",
  "userId": "user-uuid"
}
```

### PATCH / — Atualizar Elegibilidade a Royalty

**Auth:** Bearer (Admin)

**Requisição:**
```json
{
  "active": false,
  "userId": "user-uuid",
  "externalContactId": "ext-uuid"
}
```

---

## Endpoints: Contatos Externos

Caminho base: `/{companyId}/external-contacts` (backend Registry)

### GET / — Listar Contatos Externos

**Auth:** Bearer (Admin)

### POST /import — Criar/Importar Contato Externo

**Auth:** Bearer (Admin)

**Requisição:**
```json
{
  "name": "External Partner",
  "walletAddress": "0xabcd...1234",
  "email": "partner@example.com",
  "phone": "+5511999999999",
  "royaltyEligible": true,
  "description": "Royalty partner"
}
```

---

## Endpoints: Provedores de Pagamento (por Usuário)

Caminho base: `/companies/{companyId}/users/{userId}/providers` (backend Commerce)

### GET / — Listar Provedores de Pagamento do Usuário

**Auth:** Bearer (Admin)

Retorna provedores de pagamento configurados (Stripe, ASAAS, etc.) para um usuário.

### DELETE /:provider — Excluir Provedor de Pagamento

**Auth:** Bearer (Admin)

Remove uma configuração específica de provedor de pagamento de um usuário.

---

## Sistema de Indicação

### PATCH /users/:companyId/:userId/referrer — Editar Indicação

**Caminho:** `PATCH /users/{companyId}/{userId}/referrer`
**Auth:** Bearer (Admin)

**Requisição:**
```json
{
  "referrerCode": "ABC123",
  "referrerUserId": "referrer-uuid"
}
```

Vincula um usuário a um referenciador via código de indicação ou ID de usuário.
