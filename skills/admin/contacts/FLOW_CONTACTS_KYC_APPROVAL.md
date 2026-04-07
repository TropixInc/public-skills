---
id: FLOW_CONTACTS_KYC_APPROVAL
title: "Contatos - Aprovação e Revisão KYC"
module: offpix/contacts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contacts
  - kyc
  - approval
  - review
  - documents
depends_on:
  - CONTACTS_API_REFERENCE
  - FLOW_CONTACTS_KYC_SUBMISSION
---

# Aprovação e Revisão KYC

> **Referência cruzada:** Este documento cobre aprovação KYC via **serviço de Contacts** (`CONTACTS_API_REFERENCE`).
> Para aprovação via serviço de KYC dedicado, consulte [FLOW_KYC_APPROVAL.md](../kyc/FLOW_KYC_APPROVAL.md) — documento canônico de aprovação KYC.

## Visão Geral

Este fluxo cobre o lado do admin/aprovador do KYC: listar submissões pendentes, revisar documentos e tomar ações (aprovar, rejeitar ou solicitar revisão). No frontend, admins acessam isso através de **Contacts > KYC** (`/dash/contacts/KYC`) para a lista, e **Detalhes do Contato** (`/dash/contacts/clients/{id}`) para revisão individual.

O W3Block suporta três fluxos de aprovação:
- **Auto-aprovação** — documentos aprovados instantaneamente na submissão (nenhuma ação do admin necessária)
- **Aprovação padrão** — qualquer Admin ou KycApprover pode revisar
- **Aprovador específico** — apenas o aprovador designado pode revisar

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin ou KycApprover | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de auth / configuração do ambiente |
| Submissões pendentes | Usuários devem ter enviado documentos KYC | [Fluxo de Submissão KYC](./FLOW_CONTACTS_KYC_SUBMISSION.md) |

## Tipos de Fluxo de Aprovação

```
Tipo 1: Auto-Aprovação (autoApprove=true no TenantContext)
  Usuário envia → Documentos auto-aprovados → Concluído

Tipo 2: Padrão (autoApprove=false, requireSpecificApprover=false)
  Usuário envia → Qualquer Admin/KycApprover revisa → Aprovar/Rejeitar/Revisar

Tipo 3: Aprovador Específico (requireSpecificApprover=true)
  Usuário envia (com approverUserId) → Apenas aquele aprovador revisa → Aprovar/Rejeitar/Revisar
```

**Filtragem por whitelist:** Se o TenantContext tiver `approverWhitelistIds` definido, apenas KycApprovers que pertencem a essas whitelists podem ver e aprovar as submissões.

---

## Fluxo: Listar Submissões KYC

### Opção A: Via Customer Infos (página de Lista KYC)

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
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

**Exibição no frontend:** Nome, data de criação e badge de status (verde/vermelho/laranja).

### Opção B: Via User Contexts (filtragem detalhada)

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{tenantId}/users/contexts/find` | Bearer (Admin, KycApprover) |

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `status[]` | UserContextStatus[] | Filtro: `CREATED`, `REQUIRED_REVIEW`, etc. |
| `contextId[]` | UUID[] | Filtrar por context específico |
| `contextType[]` | ContextType[] | `user_properties` ou `form` |
| `userId[]` | UUID[] | Filtrar por usuários específicos |
| `excludeSelfContexts` | boolean | KycApprover: excluir próprias submissões |
| `page`, `limit` | integer | Paginação |

**Exemplo — submissões pendentes:**
```
GET /{tenantId}/users/contexts/find?status[]=CREATED&status[]=REQUIRED_REVIEW&sortBy[0][column]=createdAt&sortBy[0][order]=ASC
```

---

## Fluxo: Revisar uma Submissão

### Etapa 1: Buscar User Context com Documentos

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{tenantId}/users/contexts/{userId}/{userContextId}` | Bearer (Admin, KycApprover) |

**Resposta (200):**
```json
{
  "id": "uc-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "CREATED",
  "approverUserId": null,
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-name",
      "status": "CREATED",
      "simpleValue": "John Doe",
      "complexValue": null,
      "assetId": null,
      "input": {
        "label": "Full Name",
        "type": "user_name",
        "mandatory": true
      }
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-cpf",
      "status": "CREATED",
      "simpleValue": "12345678901",
      "complexValue": null,
      "assetId": null,
      "input": {
        "label": "CPF",
        "type": "cpf",
        "mandatory": true
      }
    },
    {
      "id": "doc-uuid-3",
      "inputId": "input-uuid-doc",
      "status": "CREATED",
      "simpleValue": null,
      "complexValue": null,
      "assetId": "asset-uuid",
      "input": {
        "label": "Government ID",
        "type": "identification_document",
        "mandatory": true
      },
      "asset": {
        "id": "asset-uuid",
        "url": "https://res.cloudinary.com/...",
        "mimeType": "image/jpeg"
      }
    }
  ],
  "context": {
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

**Renderização no frontend:** Cada documento é exibido usando seu `input.label` e o valor (`simpleValue` para texto, `asset.url` para arquivos). O `input.type` determina como o valor é renderizado (texto, pré-visualização de imagem, visualizador de documentos, etc.).

### Etapa 2: Tomar uma Ação

Escolha uma das três ações:

---

## Ação A: Aprovar

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/approve` | Bearer (Admin, KycApprover) | application/json |

**Requisição Mínima:**
```json
{}
```

**Requisição Completa:**
```json
{
  "reason": "All documents verified successfully",
  "userContextId": "uc-uuid"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da aprovação (salvo no log de auditoria) |
| `userContextId` | UUID | Não | User context específico a ser aprovado |

**Resposta:** 204 No Content

**O que acontece internamente:**
1. Todos os documentos → status `APPROVED`
2. User context → status `APPROVED`
3. Entrada no log de auditoria adicionada: `{ moderatorId, status: APPROVED, reason, registerAt }`
4. E-mail: notificação de aprovação KYC enviada ao usuário
5. Commerce: evento de status KYC despachado (pode desbloquear pedidos se `blockCommerceDeliver` estava definido)
6. Webhook: `USER_KYC_PROPS_UPDATED` disparado

---

## Ação B: Rejeitar

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/reject` | Bearer (Admin, KycApprover) | application/json |

**Requisição:**
```json
{
  "reason": "Document is expired. Please submit a valid government ID."
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `reason` | string | Não | Motivo da rejeição (enviado ao usuário por e-mail) |

**Resposta:** 204 No Content

**O que acontece internamente:**
1. Todos os documentos → status `DENIED`
2. User context → status `DENIED`
3. Entrada no log de auditoria adicionada
4. E-mail: notificação de rejeição KYC enviada ao usuário com o motivo
5. Commerce: evento de falha KYC despachado

**Observação:** A rejeição é um estado terminal. O usuário não pode reenviar para um context `DENIED`. Uma nova submissão de context precisaria ser criada.

---

## Ação C: Solicitar Revisão (Rejeição Parcial)

Esta é a ação mais detalhada — sinaliza campos específicos para reenvio enquanto mantém o restante intacto.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/require-review` | Bearer (Admin, KycApprover) | application/json |

**Requisição:**
```json
{
  "inputIds": ["input-uuid-doc", "input-uuid-selfie"],
  "reason": "Government ID is blurry and selfie doesn't match the document photo"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `inputIds` | UUID[] | Sim (mín: 1) | IDs dos inputs dos campos que precisam de reenvio |
| `reason` | string | Não | Explicação enviada ao usuário |

**Resposta:** 204 No Content

**O que acontece internamente:**
1. Apenas os documentos especificados → status `REQUIRED_REVIEW`
2. Outros documentos permanecem em seu status atual
3. User context → status `REQUIRED_REVIEW`
4. Entrada no log de auditoria adicionada com `inputIds` e `reason`
5. E-mail: notificação de solicitação de revisão enviada ao usuário, especificando quais campos precisam de atenção

**Após o usuário reenviar:** Os documentos sinalizados retornam ao status `CREATED`, e o user context retorna a `CREATED` — de volta na fila de revisão.

---

## Regras de Permissão de Aprovação

| Cenário | Quem pode aprovar |
|---------|-------------------|
| Context padrão | Qualquer Admin ou KycApprover |
| Context com `approverWhitelistIds` | Apenas KycApprovers que estão em uma das whitelists especificadas |
| Context com `requireSpecificApprover: true` | Apenas o usuário especificado em `approverUserId` na submissão |
| KycApprover revisando própria submissão | Não permitido (filtro `excludeSelfContexts`) |
| SuperAdmin | Pode aprovar qualquer context independente de regras de whitelist/aprovador |

---

## Trilha de Auditoria

Toda ação é registrada no array `logs` do user context:

```json
{
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "REQUIRED_REVIEW",
      "inputIds": ["input-uuid-doc"],
      "reason": "Document is blurry",
      "registerAt": "2026-01-16T14:00:00Z"
    },
    {
      "moderatorId": "user-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-17T09:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "APPROVED",
      "inputIds": [],
      "reason": "Documents verified",
      "registerAt": "2026-01-17T15:00:00Z"
    }
  ]
}
```

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 404 | UserContextNotFoundException | User context não encontrado | Verifique userId, contextId e tenantId |
| 403 | ForbiddenException | Aprovador não autorizado para este context | Verifique regras de whitelist/aprovador específico |
| 400 | BadRequestException | Context não está em estado aprovável | Context deve estar em `CREATED` ou `REQUIRED_REVIEW` para aprovar/rejeitar |
| 400 | BadRequestException | Nenhum inputIds fornecido para require-review | Pelo menos um inputId é necessário |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | KycApprover não consegue ver submissões | Verifique se o context tem `approverWhitelistIds` — o aprovador pode não estar na whitelist |
| 2 | Rejeição é permanente | Diferente de "solicitar revisão", a rejeição define o status como `DENIED` que é terminal. Use "solicitar revisão" para reenvio |
| 3 | Endpoint de aprovação retorna 400 | O context deve estar com status `CREATED` ou `REQUIRED_REVIEW`. Verifique o status atual primeiro |
| 4 | Pedidos do Commerce ainda bloqueados após aprovação | Se `blockCommerceDeliver` estava definido no tenant context, verifique se o serviço Commerce recebeu o evento KYC |
| 5 | Confusão com aprovação parcial | Ao usar `require-review`, apenas os `inputIds` especificados são sinalizados. Outros documentos mantêm seu status |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Submissão KYC | Submissão de documentos pelo usuário | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Gerenciamento de Usuários | Gerenciamento do registro do usuário | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Configuração de Context | Como os formulários são configurados | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Configuração de Inputs | Como os campos de formulário são definidos | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
