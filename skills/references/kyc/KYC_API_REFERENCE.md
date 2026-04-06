---
id: KYC_API_REFERENCE
title: "KYC - Referência da API"
module: offpix/kyc
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - documents
  - verification
  - approval
  - api-reference
---

# Referência da API de KYC

Referência completa de endpoints para o módulo KYC (Know Your Customer) da W3Block. Este módulo cobre submissão de documentos, validação, fluxos de aprovação e configuração KYC em nível de tenant. Todos os endpoints são servidos pelo backend PixwayID.

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://pixwayid.w3block.io` |
| Swagger | https://pixwayid.w3block.io/docs/ |

> **Nota:** Um ambiente de staging também está disponível para desenvolvimento e testes.

## Autenticação

Todos os endpoints requerem Bearer token a menos que explicitamente marcados como públicos:

```
Authorization: Bearer {accessToken}
```

---

## Enums

### UserDocumentStatus

Status de um documento individual dentro de uma submissão.

| Valor | String | Descrição |
|-------|--------|-----------|
| `CREATED` | `created` | Documento submetido, aguardando revisão |
| `APPROVED` | `approved` | Documento aprovado pelo moderador |
| `DENIED` | `denied` | Documento rejeitado pelo moderador |
| `REQUIRED_REVIEW` | `requiredReview` | Moderador solicitou reenvio deste documento |

### UserContextStatus

Status geral do contexto do usuário (agregação de todos os documentos em uma submissão).

| Valor | String | Descrição |
|-------|--------|-----------|
| `DRAFT` | `draft` | Submissão iniciada mas nem todos os campos obrigatórios foram fornecidos |
| `CREATED` | `created` | Todos os documentos obrigatórios submetidos, aguardando revisão |
| `REQUIRED_REVIEW` | `requiredReview` | Moderador sinalizou campos específicos para reenvio |
| `APPROVED` | `approved` | Todos os documentos aprovados |
| `DENIED` | `denied` | Submissão rejeitada (estado terminal) |

### DataTypesEnum

Os 19 tipos de entrada suportados para campos de formulário KYC.

| Valor | Descrição | Armazenamento | Validação |
|-------|-----------|---------------|-----------|
| `TEXT` | Texto livre | `simpleValue` | Nenhuma |
| `EMAIL` | Endereço de email | `simpleValue` | Formato RFC 5322 |
| `PHONE` | Número de telefone | `simpleValue` | Formato de telefone; suporta array ou único com `extraPhones` |
| `URL` | URL web | `simpleValue` | URL HTTP(S) válida |
| `CPF` | CPF brasileiro | `simpleValue` | 11 dígitos, validação de checksum, único por tenant |
| `BIRTHDATE` | Data de nascimento | `simpleValue` | Data válida, verificação de idade mínima (padrão 18) |
| `DATE` | Data genérica | `simpleValue` | Data ISO válida |
| `USER_NAME` | Nome completo do usuário | `simpleValue` | Atualiza automaticamente o campo `user.name` do perfil |
| `CHECKBOX` | Checkbox booleano | `simpleValue` | Valor booleano |
| `SIMPLE_SELECT` | Seleção dropdown/radio | `simpleValue` | Deve corresponder a `option.value`; respeita flag `isMultiple` |
| `DYNAMIC_SELECT` | Select populado pelo servidor | `simpleValue` | Opções buscadas dinamicamente |
| `FILE` | Upload de arquivo genérico | `assetId` | Requer asset carregado (Cloudinary) |
| `IMAGE` | Upload de imagem | `assetId` | Requer asset carregado (Cloudinary) |
| `IDENTIFICATION_DOCUMENT` | Documento de identidade | `complexValue` + `assetId` | Requer `docType` (passport, cpf, rg) e valor `document` |
| `MULTIFACE_SELFIE` | Selfie biométrica | `assetId` | Requer CPF + data de nascimento submetidos primeiro; aciona serviço biométrico |
| `SIMPLE_LOCATION` | Endereço/localização | `complexValue` | Requer `placeId`, `region`, `country` |
| `COMMERCE_PRODUCT` | Referência de produto | `complexValue` | Valida que o produto existe no serviço commerce |
| `SEPARATOR` | Separador visual | N/A | Não armazenado; ignorado durante processamento |
| `IFRAME` | Iframe incorporado | N/A | Apenas exibição; não armazenado |

### TenantContextNotificationType

Tipos de notificação por email configuráveis por tenant context.

| Valor | Destinatário | Descrição |
|-------|-------------|-----------|
| `KYC_APPROVAL_REQUEST` | Aprovadores | Enviado quando uma nova submissão é criada e pronta para revisão |
| `KYC_REQUIRE_USER_REVIEW` | Usuário | Enviado quando um moderador solicita reenvio de campos específicos |
| `KYC_USER_APPROVED` | Usuário | Enviado quando a submissão do usuário é aprovada |
| `KYC_ADMIN_APPROVED` | Admin | Enviado aos admins quando uma submissão é aprovada |
| `KYC_USER_REJECTED` | Usuário | Enviado quando a submissão do usuário é rejeitada |
| `KYC_ADMIN_REJECTED` | Admin | Enviado aos admins quando uma submissão é rejeitada |

---

## Entidades e Relacionamentos

```
User (1) ──→ (N) UsersContextsEntity ──→ (N) UsersDocumentEntity
                       │                           │
                       │ status, logs[],            │ inputId, simpleValue,
                       │ approverUserId,            │ complexValue, assetId,
                       │ utmParams                  │ status
                       │
TenantContextEntity ───┘ (define configuração de aprovação)
       │
       └──→ (N) TenantInput (define campos do formulário)
```

### UsersContextsEntity

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | UUID | Chave primária |
| `tenantId` | UUID | Tenant ao qual este context pertence |
| `userId` | UUID | Usuário que submeteu |
| `contextId` | UUID | Referência à definição do TenantContext |
| `status` | UserContextStatus | Status atual da submissão |
| `logs` | LogUserContext[] | Trilha de auditoria de todas as mudanças de status |
| `approverUserId` | UUID (nullable) | Aprovador atribuído (quando `requireSpecificApprover` é true) |
| `utmParams` | object (nullable) | Parâmetros de rastreamento UTM da submissão |
| `createdAt` | DateTime | Timestamp de criação |
| `updatedAt` | DateTime | Timestamp da última atualização |

### UsersDocumentEntity

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | UUID | Chave primária |
| `tenantId` | UUID | Tenant ao qual este documento pertence |
| `userId` | UUID | Usuário que submeteu |
| `inputId` | UUID | Referência à definição do TenantInput |
| `contextId` | UUID | Referência ao TenantContext |
| `userContextId` | UUID (nullable) | Referência ao UsersContextsEntity |
| `status` | UserDocumentStatus | Status atual do documento |
| `simpleValue` | string (nullable) | Valor armazenado para tipos simples (text, email, cpf, etc.) |
| `complexValue` | object (nullable) | Valor armazenado para tipos complexos (location, identification, etc.) |
| `assetId` | UUID (nullable) | Referência ao asset de arquivo carregado (Cloudinary) |
| `createdAt` | DateTime | Timestamp de criação |
| `updatedAt` | DateTime | Timestamp da última atualização |

### TenantContextEntity

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | UUID | Chave primária |
| `tenantId` | UUID | Tenant ao qual este context pertence |
| `contextId` | UUID | Referência à definição do context |
| `active` | boolean | Se este context KYC está ativo |
| `data` | object (nullable) | Dados de configuração adicionais |
| `approverWhitelistIds` | UUID[] (nullable) | IDs de whitelist restringindo quais aprovadores podem revisar |
| `requireSpecificApprover` | boolean | Se as submissões devem especificar um `approverUserId` |
| `autoApprove` | boolean | Se as submissões são auto-aprovadas na criação |
| `blockCommerceDeliver` | boolean | Se deve bloquear entrega de pedidos commerce até KYC ser aprovado |
| `notificationsConfig` | object (nullable) | Configuração de notificação por email por tipo de notificação |

### LogUserContext

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `moderatorId` | UUID | Usuário que realizou a ação |
| `status` | UserContextStatus | Status definido por esta ação |
| `inputIds` | UUID[] | IDs de input afetados (para ações de require-review) |
| `reason` | string (nullable) | Motivo fornecido para a ação |
| `registerAt` | DateTime | Quando a ação foi realizada |

---

## Endpoints: Submissão de Documentos

### POST /{tenantId}/users/documents/{userId}/context/{contextId} -- Submeter Documentos

**Auth:** Bearer (User, Admin, KycApprover)

Submeter um ou mais documentos contra um context KYC. Cria ou atualiza uma submissão de contexto de usuário.

**Requisição (Mínima -- apenas campos de texto):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-name",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-cpf",
      "value": "12345678901"
    }
  ]
}
```

**Requisição (Completa -- todas as opções):**
```json
{
  "documents": [
    { "inputId": "input-uuid-name", "value": "John Doe" },
    { "inputId": "input-uuid-cpf", "value": "12345678901" },
    { "inputId": "input-uuid-email", "value": "john@example.com" },
    {
      "inputId": "input-uuid-location",
      "value": {
        "region": "SP", "country": "BR",
        "placeId": "ChIJrTLr-GyuEmsRBfy61i59si0",
        "city": "Sao Paulo", "postal_code": "01000-000",
        "street_address_1": "Av Paulista 1000"
      }
    },
    {
      "inputId": "input-uuid-id-doc",
      "value": { "docType": "rg", "document": "123456789" },
      "assetId": "asset-uuid-front-photo"
    },
    { "inputId": "input-uuid-selfie", "assetId": "asset-uuid-selfie" }
  ],
  "currentStep": 2,
  "approverUserId": "approver-uuid",
  "userContextId": "existing-uc-uuid",
  "utmParams": { "utm_campaign": "onboarding", "utm_source": "email", "utm_medium": "link" }
}
```

**DTO: AttachDocumentsToUser**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `documents` | DocumentDto[] | Sim (mín: 1) | Array de submissões de documentos |
| `currentStep` | number | Não | Etapa atual do formulário para validação multi-etapas |
| `approverUserId` | UUID | Condicional | Obrigatório se o context tiver `requireSpecificApprover: true` |
| `userContextId` | UUID | Não | Anexar a contexto de usuário existente (para reenvio ou multi-etapas) |
| `utmParams` | UTMParamsDto | Não | Parâmetros de rastreamento UTM |

**DTO: DocumentDto**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `inputId` | UUID | Sim | ID do TenantInput ao qual este valor pertence |
| `value` | string ou object | Condicional | Valor do campo (para tipos texto ou tipos complexos) |
| `assetId` | UUID | Condicional | ID do asset de arquivo carregado (para tipos arquivo/imagem) |

**Resposta (201):** `UsersContextsEntity` com documentos.

**Transições de Status:**
- Se nem todos os campos obrigatórios foram submetidos: status = `DRAFT`
- Se todos os campos obrigatórios foram submetidos: status = `CREATED`
- Se `autoApprove` é true no context: status = `APPROVED` imediatamente

---

### GET /{tenantId}/users/documents/{userId} -- Obter Documentos do Usuário (Paginado)

**Auth:** Bearer (Admin, User -- usuário pode buscar apenas os próprios)

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | `1` | Número da página |
| `limit` | integer | `10` | Itens por página |
| `type[]` | DataTypesEnum[] | -- | Filtrar por tipos de entrada |
| `contextId` | UUID | -- | Filtrar por context |
| `inputId` | UUID | -- | Filtrar por input específico |

**Resposta (200):** Lista paginada de `UsersDocumentEntity` com relações de input e asset.

---

### GET /{tenantId}/users/documents/{userId}/context/{contextId} -- Obter Documentos por Context

**Auth:** Bearer (Admin, User -- usuário pode buscar apenas os próprios)

Retorna todos os documentos para um context específico como array não paginado.

**Resposta (200):**
```json
[
  {
    "id": "doc-uuid-1", "inputId": "input-uuid-name", "status": "created",
    "simpleValue": "John Doe", "complexValue": null, "assetId": null,
    "input": { "id": "input-uuid-name", "label": "Full Name", "type": "user_name", "mandatory": true },
    "asset": null
  },
  {
    "id": "doc-uuid-2", "inputId": "input-uuid-file", "status": "created",
    "simpleValue": null, "complexValue": null, "assetId": "asset-uuid",
    "input": { "id": "input-uuid-file", "label": "Government ID", "type": "identification_document", "mandatory": true },
    "asset": { "id": "asset-uuid", "url": "https://res.cloudinary.com/w3block/image/upload/v1/...", "mimeType": "image/jpeg" }
  }
]
```

---

### GET /documents/find-user-by-any -- Buscar Usuário por CPF ou ID

**Auth:** Bearer (Admin, Integration)

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `cpf` | string | Número do CPF (11 dígitos) |
| `userId` | UUID | ID do usuário |

Pelo menos um entre `cpf` ou `userId` é obrigatório. A busca por CPF pesquisa nos documentos KYC submetidos dentro do tenant.

**Resposta (200):**
```json
{ "id": "user-uuid", "name": "John Doe", "email": "john@example.com", "tenantId": "tenant-uuid" }
```

---

## Endpoints: Contextos de Usuário

### GET /{tenantId}/users/contexts/find -- Listar Contextos com Filtros

**Auth:** Bearer (User, Admin, KycApprover)

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | `1` | Número da página |
| `limit` | integer | `10` | Itens por página |
| `search` | string | -- | Buscar em email/nome do usuário |
| `sortBy` | array | `[{column: 'createdAt', order: 'ASC'}]` | Configuração de ordenação |
| `status[]` | UserContextStatus[] | -- | Filtrar por status(es) |
| `contextId[]` | UUID[] | -- | Filtrar por IDs de context |
| `contextType[]` | ContextType[] | -- | Filtrar por `user_properties` ou `form` |
| `userId[]` | UUID[] | -- | Filtrar por IDs de usuário (apenas Admin) |
| `excludeSelfContexts` | boolean | -- | KycApprover: excluir próprias submissões |
| `preOrder` | boolean | -- | Filtrar contexts de pré-pedido |
| `utmCampaign` | string | -- | Filtrar por campanha UTM |
| `utmSource` | string | -- | Filtrar por fonte UTM |

**Resposta (200):** Lista paginada de UsersContextsEntity com relações de usuário e context.

---

### GET /{tenantId}/users/contexts/{userId} -- Obter Contextos do Usuário

**Auth:** Bearer (Admin, User -- usuário pode ver apenas os próprios)

**Parâmetros de Query:** `page`, `limit`, `search`, `sortBy`, `status`, `contextId`

Retorna lista paginada de contextos de usuário para um usuário específico.

---

### GET /{tenantId}/users/contexts/{userId}/{userContextId} -- Obter Contexto com Documentos

**Auth:** Bearer (User, Admin, KycApprover)

**Resposta (200):** Retorna o UsersContextsEntity completo com logs, documentos, context e dados do usuário.

---

## Endpoints: Ações de Aprovação

### PATCH /{tenantId}/users/contexts/{userId}/{contextId}/approve -- Aprovar Submissão

**Auth:** Bearer (Admin, KycApprover)

**DTO: UserContextStatusDto**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da aprovação (salvo no log de auditoria) |
| `userContextId` | UUID | Não | Contexto de usuário específico a aprovar |

**Resposta:** 204 No Content

**Efeitos Colaterais:**
1. Todos os documentos no context definidos como `APPROVED`
2. Status do contexto do usuário definido como `APPROVED`
3. Entrada no log de auditoria adicionada com ID do moderador, timestamp, motivo
4. Email de aprovação KYC enviado ao usuário (se configurado)
5. Serviço Commerce notificado (pode desbloquear pedidos se `blockCommerceDeliver` for true)
6. Webhook `USER_KYC_PROPS_UPDATED` despachado

---

### PATCH /{tenantId}/users/contexts/{userId}/{contextId}/reject -- Rejeitar Submissão

**Auth:** Bearer (Admin, KycApprover)

**DTO: UserContextStatusDto**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da rejeição (enviado ao usuário por email) |
| `userContextId` | UUID | Não | Contexto de usuário específico a rejeitar |

**Resposta:** 204 No Content

**Efeitos Colaterais:**
1. Todos os documentos no context definidos como `DENIED`
2. Status do contexto do usuário definido como `DENIED`
3. Entrada no log de auditoria adicionada
4. Email de rejeição KYC enviado ao usuário com motivo
5. Serviço Commerce notificado

**IMPORTANTE:** `DENIED` é um estado terminal. O usuário não pode reenviar para um context negado. Use "solicitar revisão" em vez disso se o reenvio for desejado.

---

### PATCH /{tenantId}/users/contexts/{userId}/{contextId}/require-review -- Solicitar Revisão

**Auth:** Bearer (Admin, KycApprover)

**DTO: RequiredReviewContextStatusDto**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `inputIds` | UUID[] | Sim (mín: 1) | IDs de input dos campos que precisam de reenvio |
| `reason` | string | Não | Explicação enviada ao usuário |

**Resposta:** 204 No Content

**Efeitos Colaterais:**
1. Apenas os documentos especificados definidos como `REQUIRED_REVIEW`
2. Outros documentos mantêm seu status atual
3. Status do contexto do usuário definido como `REQUIRED_REVIEW`
4. Entrada no log de auditoria adicionada com `inputIds` e `reason`
5. Email de solicitação de revisão enviado ao usuário especificando quais campos precisam de atenção

---

## Regras de Validação por Tipo de Entrada

### CPF

| Regra | Detalhe |
|-------|---------|
| Formato | Exatamente 11 dígitos numéricos |
| Checksum | Validado usando o algoritmo de CPF brasileiro (dígitos verificadores mod-11) |
| Sanitização | Caracteres não numéricos são removidos antes da validação |
| Unicidade | Cada valor de CPF deve ser único por tenant |
| Armazenamento | `simpleValue` (apenas dígitos) |

### EMAIL

| Regra | Detalhe |
|-------|---------|
| Formato | Endereço de email compatível com RFC 5322 |
| Armazenamento | `simpleValue` |

### PHONE

| Regra | Detalhe |
|-------|---------|
| Formato | String de número de telefone (ex: `"+5511999999999"`) |
| Telefones extras | Suporta array `extraPhones` para números adicionais |
| Armazenamento | `simpleValue` (único) ou `complexValue` (com extras) |

### BIRTHDATE

| Regra | Detalhe |
|-------|---------|
| Formato | String de data ISO (ex: `"1990-05-15"`) |
| Idade mínima | Idade mínima padrão é 18 anos; configurável por input |
| Armazenamento | `simpleValue` |

### SIMPLE_SELECT

| Regra | Detalhe |
|-------|---------|
| Formato | String correspondendo a um dos `options[].value` da definição do input |
| Múltiplo | Se `isMultiple` for true, o valor pode ser um array de valores de opção |
| Armazenamento | `simpleValue` |

### SIMPLE_LOCATION

| Regra | Detalhe |
|-------|---------|
| Campos obrigatórios | `placeId`, `region`, `country` |
| Campos opcionais | `city`, `postal_code`, `street_address_1`, `street_address_2`, `latitude`, `longitude` |
| Armazenamento | `complexValue` |

Exemplo:
```json
{
  "placeId": "ChIJrTLr-GyuEmsRBfy61i59si0",
  "region": "SP", "country": "BR",
  "city": "Sao Paulo", "postal_code": "01000-000",
  "street_address_1": "Av Paulista 1000"
}
```

### IDENTIFICATION_DOCUMENT

| Regra | Detalhe |
|-------|---------|
| Campos obrigatórios | `docType` (um de: `passport`, `cpf`, `rg`), `document` (número do documento) |
| Anexo de arquivo | Opcionalmente incluir `assetId` para foto do documento |
| Armazenamento | `complexValue` + `assetId` opcional |

### FILE / IMAGE

| Regra | Detalhe |
|-------|---------|
| Requisito | Deve fornecer `assetId` de um upload prévio ao Cloudinary |
| Validação de arquivo | O asset deve existir e estar acessível |
| Armazenamento | `assetId` |

### MULTIFACE_SELFIE

| Regra | Detalhe |
|-------|---------|
| Requisito | Deve fornecer `assetId` de um upload prévio ao Cloudinary |
| Pré-requisitos | Usuário deve ter submetido documentos de CPF e data de nascimento ANTES da selfie |
| Processamento | Aciona chamada ao serviço de validação biométrica |
| Armazenamento | `assetId` |

### COMMERCE_PRODUCT

| Regra | Detalhe |
|-------|---------|
| Campos obrigatórios | `productId`, `quantity` |
| Validação | Produto deve existir no serviço commerce |
| Armazenamento | `complexValue` |

---

## Referência de Erros

| Status | Exceção | Causa | Resolução |
|--------|---------|-------|-----------|
| 400 | `BadRequestException` | Documento obrigatório ausente ou formato de valor inválido | Verificar requisitos do campo e formato do valor por tipo de entrada |
| 400 | `CPfDuplicatedException` | CPF já usado por outro usuário neste tenant | Cada CPF deve ser único por tenant; verificar submissões existentes |
| 400 | `InvalidCPFException` | Validação de checksum do CPF falhou | Verificar se o CPF tem exatamente 11 dígitos com dígitos verificadores mod-11 válidos |
| 400 | `MaxSubmissionReachedException` | Usuário excedeu máximo de submissões para este context | Limite de `maxSubmissions` do context atingido; não é possível criar novas submissões |
| 400 | `TenantContextDisabledException` | Context está inativo | Contatar admin para ativar o context |
| 400 | `BadRequestException` (aprovar) | Context não está em estado aprovável | Context deve estar em `CREATED` ou `REQUIRED_REVIEW` para aprovar/rejeitar |
| 400 | `BadRequestException` (revisão) | Nenhum `inputIds` fornecido para require-review | Pelo menos um inputId é obrigatório |
| 403 | `NotAuthorizedAttachmentDocumentToContextException` | Usuário não pode submeter neste context | Verificar permissões do usuário e regras de acesso ao context |
| 403 | `UserNeedVerifyAccount` | Email não verificado | Completar verificação de email primeiro (a menos que passwordless) |
| 403 | `ForbiddenException` | Aprovador não autorizado para este context | Verificar associação à whitelist e atribuição de aprovador específico |
| 404 | `UserContextNotFoundException` | Contexto de usuário não encontrado | Verificar userId, contextId, userContextId e tenantId |
| 500 | `ItWasNotPossibleRegisterMutiplaceExpection` | Serviço de validação biométrica falhou | Tentar novamente com foto de selfie mais clara; garantir que CPF e data de nascimento foram submetidos |
