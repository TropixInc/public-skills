---
id: FLOW_KYC_APPROVAL
title: "KYC - Aprovação e Revisão"
module: offpix/kyc
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - approval
  - review
  - admin
  - audit
depends_on:
  - KYC_API_REFERENCE
  - FLOW_KYC_SUBMISSION
---

# Aprovação e Revisão de KYC

> **Referência cruzada:** Este documento é o **canônico** para aprovação KYC via serviço de KYC (`KYC_API_REFERENCE`).
> Para aprovação via serviço de Contacts, consulte [FLOW_CONTACTS_KYC_APPROVAL.md](../contacts/FLOW_CONTACTS_KYC_APPROVAL.md) — ambos cobrem o mesmo fluxo de aprovação com endpoints ligeiramente diferentes.

## Visão Geral

Este fluxo cobre o lado do admin/aprovador do KYC: listar submissões pendentes, revisar documentos enviados e tomar ações (aprovar, rejeitar ou solicitar revisão). Os admins acessam isso através do painel em `/dash/contacts/KYC` para a visão em lista e `/dash/contacts/clients/{id}` para revisão individual.

O W3Block suporta três tipos de fluxo de aprovação:

| Fluxo de trabalho | Configuração | Comportamento |
|-------------------|--------------|---------------|
| Aprovação automática | `autoApprove: true` | Documentos aprovados instantaneamente no envio, nenhuma ação do admin necessária |
| Padrão | `autoApprove: false`, `requireSpecificApprover: false` | Qualquer Admin ou KycApprover pode revisar e agir |
| Aprovador específico | `requireSpecificApprover: true` | Apenas o aprovador designado durante o envio pode revisar |

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin ou KycApprover | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Submissões pendentes | Usuários devem ter enviado documentos KYC com status `CREATED` | [Fluxo de Submissão KYC](./FLOW_KYC_SUBMISSION.md) |

---

## Fluxo: Listar Submissões Pendentes

### Opção A: Via User Contexts (Filtragem Completa)

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/{tenantId}/users/contexts/find` | Bearer (Admin, KycApprover) |

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | integer | `1` | Número da página |
| `limit` | integer | `10` | Itens por página |
| `search` | string | -- | Busca no email/nome do usuário |
| `sortBy` | array | `[{column: 'createdAt', order: 'ASC'}]` | Configuração de ordenação |
| `status[]` | UserContextStatus[] | -- | Filtrar por status(es) |
| `contextId[]` | UUID[] | -- | Filtrar por IDs de contexto específicos |
| `contextType[]` | ContextType[] | -- | Filtrar por `user_properties` ou `form` |
| `userId[]` | UUID[] | -- | Filtrar por usuários específicos (somente Admin) |
| `excludeSelfContexts` | boolean | -- | KycApprover: excluir suas próprias submissões |
| `utmCampaign` | string | -- | Filtrar por campanha UTM |
| `utmSource` | string | -- | Filtrar por fonte UTM |

**Exemplo -- submissões pendentes ordenadas da mais antiga:**
```
GET /{tenantId}/users/contexts/find?status[]=created&sortBy[0][column]=createdAt&sortBy[0][order]=ASC&limit=20
```

**Exemplo -- submissões pendentes e com revisão solicitada:**
```
GET /{tenantId}/users/contexts/find?status[]=created&status[]=requiredReview&limit=20
```

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "uc-uuid-1",
      "tenantId": "tenant-uuid",
      "userId": "user-uuid-1",
      "contextId": "ctx-uuid",
      "status": "created",
      "approverUserId": null,
      "utmParams": null,
      "createdAt": "2026-01-15T10:00:00Z",
      "updatedAt": "2026-01-15T10:00:00Z",
      "user": {
        "name": "John Doe",
        "email": "john@example.com"
      },
      "context": {
        "slug": "signup",
        "description": "SignUp Form",
        "type": "user_properties"
      }
    },
    {
      "id": "uc-uuid-2",
      "tenantId": "tenant-uuid",
      "userId": "user-uuid-2",
      "contextId": "ctx-uuid",
      "status": "created",
      "approverUserId": "approver-uuid",
      "utmParams": { "utm_campaign": "onboarding" },
      "createdAt": "2026-01-16T08:30:00Z",
      "updatedAt": "2026-01-16T08:30:00Z",
      "user": {
        "name": "Jane Smith",
        "email": "jane@example.com"
      },
      "context": {
        "slug": "signup",
        "description": "SignUp Form",
        "type": "user_properties"
      }
    }
  ],
  "meta": {
    "totalItems": 42,
    "itemCount": 2,
    "itemsPerPage": 20,
    "totalPages": 3,
    "currentPage": 1
  }
}
```

### Opção B: Via Customer Infos (Lista Simplificada)

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/customer-infos/{tenantId}/search` | Bearer (Admin) |

**Parâmetros de Query:** `page`, `limit`, `sortBy=createdAt`, `orderBy`

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "ci-uuid",
      "customerId": "user-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "status": "created",
      "tenantId": "tenant-uuid",
      "createdAt": "2026-01-15T10:00:00Z"
    }
  ]
}
```

Este endpoint fornece uma visão simplificada com nome, email e status. Para detalhes completos dos documentos, use o endpoint de user contexts.

---

## Fluxo: Revisar Detalhes da Submissão

Uma vez que você tenha uma submissão da lista, busque os detalhes completos incluindo todos os documentos, definições de input e URLs de assets.

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/{tenantId}/users/contexts/{userId}/{userContextId}` | Bearer (User, Admin, KycApprover) |

**Resposta (200):**
```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "created",
  "approverUserId": null,
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "utmParams": null,
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-name",
      "status": "created",
      "simpleValue": "John Doe",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid-name",
        "label": "Full Name",
        "type": "user_name",
        "mandatory": true
      },
      "asset": null
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-cpf",
      "status": "created",
      "simpleValue": "12345678901",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid-cpf",
        "label": "CPF",
        "type": "cpf",
        "mandatory": true
      },
      "asset": null
    },
    {
      "id": "doc-uuid-3",
      "inputId": "input-uuid-birthdate",
      "status": "created",
      "simpleValue": "1990-05-15",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid-birthdate",
        "label": "Date of Birth",
        "type": "birthdate",
        "mandatory": true
      },
      "asset": null
    },
    {
      "id": "doc-uuid-4",
      "inputId": "input-uuid-id-doc",
      "status": "created",
      "simpleValue": null,
      "complexValue": {
        "docType": "rg",
        "document": "123456789"
      },
      "assetId": "asset-uuid-front",
      "input": {
        "id": "input-uuid-id-doc",
        "label": "Government ID",
        "type": "identification_document",
        "mandatory": true
      },
      "asset": {
        "id": "asset-uuid-front",
        "url": "https://res.cloudinary.com/w3block/image/upload/v1/kyc-documents/front.jpg",
        "mimeType": "image/jpeg"
      }
    },
    {
      "id": "doc-uuid-5",
      "inputId": "input-uuid-selfie",
      "status": "created",
      "simpleValue": null,
      "complexValue": null,
      "assetId": "asset-uuid-selfie",
      "input": {
        "id": "input-uuid-selfie",
        "label": "Selfie Verification",
        "type": "multiface_selfie",
        "mandatory": true
      },
      "asset": {
        "id": "asset-uuid-selfie",
        "url": "https://res.cloudinary.com/w3block/image/upload/v1/kyc-documents/selfie.jpg",
        "mimeType": "image/jpeg"
      }
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

**Guia de renderização no frontend:**
- Exiba cada documento usando seu `input.label` como nome do campo
- Para tipos de texto (`simpleValue`): renderize o valor diretamente
- Para tipos complexos (`complexValue`): renderize dados estruturados (ex.: tipo de documento + número)
- Para tipos de arquivo (`asset`): renderize preview de imagem ou visualizador de documento usando `asset.url`
- Mostre o badge de `status` atual para cada documento
- Exiba os `logs` de auditoria como uma timeline

---

## Fluxo: Aprovar

Aprova uma submissão, marcando todos os documentos e o contexto como aprovados.

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/approve` | Bearer (Admin, KycApprover) | application/json |

**Requisição Mínima:**
```json
{}
```

**Requisição Completa:**
```json
{
  "reason": "All documents verified successfully. Identity confirmed.",
  "userContextId": "uc-uuid"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da aprovação (salvo no log de auditoria) |
| `userContextId` | UUID | Não | User context específico a aprovar |

**Resposta:** 204 No Content

**O que acontece internamente:**

1. Todos os documentos no contexto são definidos com status `APPROVED`
2. O status do user context é definido como `APPROVED`
3. Entrada no log de auditoria adicionada: `{ moderatorId, status: "approved", reason, registerAt }`
4. Email de aprovação KYC enviado ao usuário (se a notificação `KYC_USER_APPROVED` estiver configurada)
5. Notificação de aprovação KYC enviada aos admins (se a notificação `KYC_ADMIN_APPROVED` estiver configurada)
6. Serviço de commerce notificado -- pode desbloquear pedidos pendentes se `blockCommerceDeliver` for true no contexto
7. Webhook `USER_KYC_PROPS_UPDATED` disparado

---

## Fluxo: Rejeitar

Rejeita uma submissão. Este é um estado terminal -- o usuário não pode reenviar para um contexto negado.

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/reject` | Bearer (Admin, KycApprover) | application/json |

**Requisição Mínima:**
```json
{}
```

**Requisição Completa:**
```json
{
  "reason": "Submitted documents are fraudulent. Account flagged for review.",
  "userContextId": "uc-uuid"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da rejeição (enviado ao usuário por email) |
| `userContextId` | UUID | Não | User context específico a rejeitar |

**Resposta:** 204 No Content

**O que acontece internamente:**

1. Todos os documentos no contexto são definidos com status `DENIED`
2. O status do user context é definido como `DENIED`
3. Entrada no log de auditoria adicionada com ID do moderador, status, motivo e timestamp
4. Email de rejeição KYC enviado ao usuário com motivo (se a notificação `KYC_USER_REJECTED` estiver configurada)
5. Notificação de rejeição KYC enviada aos admins (se a notificação `KYC_ADMIN_REJECTED` estiver configurada)
6. Serviço de commerce notificado da falha do KYC

**IMPORTANTE:** `DENIED` é um estado terminal. O usuário não pode reenviar documentos para um contexto negado. Se você deseja que o usuário corrija e reenvie campos específicos, use "solicitar revisão" em vez disso. Uma nova submissão de contexto precisaria ser criada se a rejeição foi feita por engano.

---

## Fluxo: Solicitar Revisão

Marca campos específicos para reenvio. Este é um estado não-terminal que permite ao usuário corrigir e reenviar apenas os campos marcados.

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/require-review` | Bearer (Admin, KycApprover) | application/json |

**Requisição:**
```json
{
  "inputIds": ["input-uuid-id-doc", "input-uuid-selfie"],
  "reason": "Government ID is expired (expiration date 2025-12-31). Selfie image is too dark to verify identity."
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `inputIds` | UUID[] | Sim (mín: 1) | IDs dos inputs dos campos que precisam de reenvio |
| `reason` | string | Não | Explicação enviada ao usuário |

**Resposta:** 204 No Content

**O que acontece internamente:**

1. Apenas os documentos correspondentes aos `inputIds` especificados são definidos com status `REQUIRED_REVIEW`
2. Outros documentos mantêm seu status atual (permanecem `CREATED`)
3. O status do user context é definido como `REQUIRED_REVIEW`
4. Entrada no log de auditoria adicionada com `inputIds`, motivo e timestamp
5. Email de solicitação de revisão enviado ao usuário especificando quais campos precisam de atenção (se a notificação `KYC_REQUIRE_USER_REVIEW` estiver configurada)

**Após o usuário reenviar:**
- Os documentos marcados retornam ao status `CREATED` com novos valores
- O user context retorna ao status `CREATED`
- A submissão re-entra na fila de revisão

---

## Modelo de Permissões

### Acesso Baseado em Roles

| Role | Pode listar submissões | Pode aprovar | Pode rejeitar | Pode solicitar revisão |
|------|:----------------------:|:------------:|:-------------:|:----------------------:|
| SuperAdmin | Sim (todas) | Sim (todas) | Sim (todas) | Sim (todas) |
| Admin | Sim (todas) | Sim (todas) | Sim (todas) | Sim (todas) |
| KycApprover | Sim (restrito) | Sim (restrito) | Sim (restrito) | Sim (restrito) |
| User | Não | Não | Não | Não |

### Restrições do KycApprover

KycApprovers possuem restrições adicionais além dos usuários Admin:

**1. Requisito de Aprovador Específico**
Se `requireSpecificApprover: true` no TenantContext, KycApprovers só podem aprovar submissões onde `approverUserId` corresponde ao seu próprio ID de usuário. Submissões atribuídas a outros aprovadores não são acessíveis.

**2. Restrição de Whitelist**
Se `approverWhitelistIds` estiver definido no TenantContext, apenas KycApprovers que pertencem a uma dessas whitelists podem ver e agir nas submissões desse contexto. Aprovadores que não estão em nenhuma das whitelists especificadas recebem um erro 403 Forbidden.

**3. Auto-Exclusão**
KycApprovers não podem aprovar suas próprias submissões. Use o filtro `excludeSelfContexts: true` ao listar para excluir automaticamente suas próprias submissões dos resultados.

**4. Exclusão de Aprovação Automática**
Se `autoApprove: true` no TenantContext, KycApprovers não podem acessar esses contextos, pois as submissões são aprovadas automaticamente e não requerem revisão manual.

### Fluxo de Decisão de Permissão

```
O usuário é SuperAdmin ou Admin?
  SIM → Pode aprovar qualquer submissão
  NÃO → O usuário é KycApprover?
    NÃO → Acesso negado
    SIM → autoApprove está habilitado?
      SIM → Acesso negado (revisão manual não necessária)
      NÃO → requireSpecificApprover está habilitado?
        SIM → approverUserId == usuário atual?
          SIM → Pode aprovar
          NÃO → Acesso negado
        NÃO → approverWhitelistIds estão definidos?
          SIM → O usuário está em uma das whitelists?
            SIM → Pode aprovar
            NÃO → Acesso negado
          NÃO → É a própria submissão do usuário?
            SIM → Acesso negado
            NÃO → Pode aprovar
```

---

## Trilha de Auditoria

Cada mudança de status em um user context adiciona uma entrada `LogUserContext` ao array `logs`. Isso fornece um histórico completo e imutável do ciclo de vida da submissão.

**Exemplo -- ciclo de vida completo:**
```json
{
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "requiredReview",
      "inputIds": ["input-uuid-id-doc"],
      "reason": "Government ID is expired",
      "registerAt": "2026-01-16T14:00:00Z"
    },
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-17T09:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "approved",
      "inputIds": [],
      "reason": "All documents verified",
      "registerAt": "2026-01-17T15:00:00Z"
    }
  ]
}
```

**Campos do LogUserContext:**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `moderatorId` | UUID | Usuário que realizou a ação (usuário para submissões, admin para revisões) |
| `status` | UserContextStatus | Status definido por esta ação |
| `inputIds` | UUID[] | IDs dos inputs afetados (preenchido para require-review, vazio para outras ações) |
| `reason` | string (nullable) | Motivo fornecido para a ação |
| `registerAt` | DateTime | Quando a ação foi realizada |

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | `BadRequestException` | Contexto não está em estado aprovável | O contexto deve estar em `CREATED` ou `REQUIRED_REVIEW` para aprovar/rejeitar/solicitar revisão |
| 400 | `BadRequestException` | Nenhum `inputIds` fornecido para require-review | Pelo menos um `inputId` é necessário ao solicitar revisão |
| 403 | `ForbiddenException` | Aprovador não autorizado para este contexto | Verifique a associação à whitelist e a atribuição de aprovador específico |
| 403 | `ForbiddenException` | KycApprover tentando aprovar sua própria submissão | KycApprovers não podem agir em suas próprias submissões |
| 404 | `UserContextNotFoundException` | User context não encontrado | Verifique `userId`, `contextId`, `userContextId` e `tenantId` |

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | KycApprover não consegue ver submissões | Verifique se o contexto tem `approverWhitelistIds` definido. O aprovador deve ser membro de uma das whitelists especificadas |
| 2 | Rejeição é permanente | Diferente de "solicitar revisão", a rejeição define o status como `DENIED`, que é terminal. O usuário não pode reenviar. Use "solicitar revisão" para problemas corrigíveis |
| 3 | Endpoint de aprovação retorna 400 | O contexto deve estar com status `CREATED` ou `REQUIRED_REVIEW`. Um contexto `DRAFT` (submissão incompleta) não pode ser aprovado |
| 4 | Pedidos de commerce ainda bloqueados após aprovação | Se `blockCommerceDeliver` foi definido no tenant context, verifique se o serviço de commerce recebeu o evento de aprovação KYC |
| 5 | Confusão com aprovação parcial | A ação "solicitar revisão" marca apenas os `inputIds` especificados. Outros documentos mantêm seu status atual inalterado |
| 6 | Trilha de auditoria ausente | Cada ação é registrada no array `logs`. Verifique os logs para o histórico completo de mudanças de status, incluindo quem agiu e quando |
| 7 | Aprovador atribuído mas outro admin aprovando | Quando `requireSpecificApprover: true`, apenas o `approverUserId` definido durante a submissão pode aprovar. Outros admins (não-SuperAdmin) recebem um 403 |

---

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Submissão KYC | Envio de documentos pelo usuário | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Configuração KYC | Configuração do fluxo de aprovação | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Configuração de Contexto | Como contextos KYC são criados | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Gerenciamento de Inputs | Como campos de formulário são definidos | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Referência da API | Detalhes completos de endpoints e DTOs | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
