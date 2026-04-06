---
id: FLOW_CONTACTS_KYC_SUBMISSION
title: "Contatos - Submissão de Documentos KYC"
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
  - documents
  - submission
depends_on:
  - CONTACTS_API_REFERENCE
  - CONFIGURATIONS_API_REFERENCE
  - FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
  - FLOW_CONFIGURATIONS_INPUT_MANAGEMENT
---

# Submissão de Documentos KYC

## Visão Geral

Este fluxo cobre como os usuários submetem documentos KYC em um formulário de contexto. O processo é: buscar a definição do formulário (inputs), coletar dados do usuário, submeter documentos e acompanhar o status da submissão através do pipeline de aprovação. As submissões seguem o ciclo de vida de status: `DRAFT → CREATED → APPROVED / DENIED / REQUIRED_REVIEW`.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT do usuário que está submetendo | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| E-mail verificado | O usuário deve ter um e-mail verificado | Verificação de e-mail no cadastro |
| Contexto ativo | Um contexto KYC deve estar configurado e ativo | [Ciclo de Vida do Contexto](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

## Entidades e Ciclo de Vida de Status

```
Status Lifecycle:

DRAFT ──→ CREATED ──→ APPROVED
              │
              ├──→ DENIED (terminal)
              │
              └──→ REQUIRED_REVIEW ──→ CREATED (user resubmits)
                        │
                        └──→ (cycle continues until APPROVED or DENIED)
```

| Status | Significado | Acionado por |
|--------|-------------|-------------|
| `DRAFT` | Submissão iniciada mas incompleta | Submissão inicial de documentos (nem todos os campos obrigatórios) |
| `CREATED` | Todos os documentos obrigatórios submetidos, aguardando revisão | Preenchimento de todos os campos obrigatórios |
| `REQUIRED_REVIEW` | Aprovador solicitou que o usuário reenvie campos específicos | Ação do administrador |
| `APPROVED` | Todos os documentos aprovados | Ação do administrador (ou aprovação automática) |
| `DENIED` | Documentos rejeitados | Ação do administrador |

---

## Fluxo: Submeter Documentos KYC

### Passo 1: Buscar Campos do Formulário

Obtenha a definição do formulário para saber quais campos coletar.

**Endpoint (Público):**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/tenants/{tenantId}/tenant-input/slug/{slug}` | Nenhuma |

Exemplo para o formulário de cadastro:

```
GET /tenants/{tenantId}/tenant-input/slug/signup
```

**Resposta (200):**
```json
[
  {
    "id": "input-uuid-1",
    "label": "Full Name",
    "description": "Enter your full legal name",
    "type": "user_name",
    "order": 1,
    "mandatory": true,
    "active": true,
    "attributeName": "fullName",
    "step": 1,
    "options": []
  },
  {
    "id": "input-uuid-2",
    "label": "CPF",
    "description": "Enter your CPF number",
    "type": "cpf",
    "order": 2,
    "mandatory": true,
    "active": true,
    "attributeName": "cpf",
    "step": 1,
    "options": []
  },
  {
    "id": "input-uuid-3",
    "label": "Government ID",
    "description": "Upload front and back of your ID",
    "type": "identification_document",
    "order": 3,
    "mandatory": true,
    "active": true,
    "attributeName": "govId",
    "step": 2,
    "options": []
  },
  {
    "id": "input-uuid-4",
    "label": "Selfie",
    "description": "Take a selfie holding your ID",
    "type": "multiface_selfie",
    "order": 4,
    "mandatory": true,
    "active": true,
    "attributeName": "selfie",
    "step": 2,
    "options": []
  }
]
```

### Passo 2: Fazer Upload de Arquivos (se necessário)

Para inputs baseados em arquivo (`file`, `image`, `identification_document`, `multiface_selfie`), faça o upload do arquivo primeiro para obter um `assetId`.

Os arquivos são enviados para o Cloudinary através do serviço de assets do W3Block. O fluxo de upload retorna um `assetId` que você referencia na submissão do documento.

### Passo 3: Submeter Documentos

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{tenantId}/users/documents/{userId}/context/{contextId}` | Bearer (Usuário) | application/json |

**Requisição Mínima (apenas campos de texto):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-1",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-2",
      "value": "12345678901"
    }
  ]
}
```

**Requisição Completa (todos os tipos de campo, exemplo de produção):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-1",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-2",
      "value": "12345678901"
    },
    {
      "inputId": "input-uuid-3",
      "assetId": "asset-uuid-front"
    },
    {
      "inputId": "input-uuid-4",
      "assetId": "asset-uuid-selfie"
    }
  ],
  "currentStep": 2,
  "approverUserId": "approver-uuid",
  "utmParams": {
    "utm_campaign": "onboarding",
    "utm_source": "email"
  }
}
```

**Resposta (201):**
```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "CREATED",
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
      "inputId": "input-uuid-1",
      "status": "CREATED",
      "simpleValue": "John Doe"
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-2",
      "status": "CREATED",
      "simpleValue": "12345678901"
    }
  ]
}
```

**Observações:**
- Se todos os campos obrigatórios estiverem incluídos, o status transiciona para `CREATED` (pronto para revisão)
- Se alguns campos obrigatórios estiverem faltando, o status permanece como `DRAFT`
- Se o contexto tiver `autoApprove: true`, os documentos são imediatamente definidos como `APPROVED`
- `currentStep` é usado para validação de formulários multi-etapas -- só valida campos obrigatórios até aquela etapa
- O tipo de input `user_name` atualiza automaticamente o campo `name` do perfil do usuário

---

## Referência de Formato de Valores

### Valores Simples (armazenados como `simpleValue`)

| Tipo | Formato | Exemplo | Validação |
|------|---------|---------|-----------|
| `text` | string | `"John Doe"` | Nenhuma |
| `email` | string | `"john@example.com"` | Formato de e-mail |
| `url` | string | `"https://example.com"` | URL HTTP(S) válida |
| `phone` | string | `"+5511999999999"` | Formato de telefone |
| `cpf` | string (apenas dígitos) | `"12345678901"` | 11 dígitos, checksum CPF válido, único por tenant |
| `date` | Data ISO | `"2026-01-15"` | Data válida |
| `birthdate` | Data ISO | `"1990-05-15"` | Data válida |
| `user_name` | string | `"John Doe"` | Atualiza user.name |
| `simple_select` | string | `"BR"` | Deve estar nos valores de input.options |
| `checkbox` | boolean | `true` | Booleano |

### Valores Complexos (armazenados como `complexValue`)

| Tipo | Formato | Exemplo |
|------|---------|---------|
| `identification_document` | `{ docType, document }` | `{ "docType": "RG", "document": "123456789" }` |
| `simple_location` | `{ region, country, placeId, city?, ... }` | `{ "region": "SP", "country": "BR", "placeId": "ChIJ...", "city": "Sao Paulo" }` |
| `commerce_product` | `{ productId, quantity, variantIds? }` | `{ "productId": "uuid", "quantity": "1" }` |

### Valores Baseados em Arquivo (use `assetId`)

| Tipo | Observações |
|------|-------------|
| `file` | Upload de arquivo genérico |
| `image` | Arquivo de imagem |
| `identification_document` | Pode combinar `assetId` + `complexValue` |
| `multiface_selfie` | Aciona validação biométrica. Requer que o usuário tenha submetido CPF e data de nascimento |

---

## Submissão Multi-Etapas

Para formulários com múltiplas etapas, submeta documentos incrementalmente:

### Submeter Etapa 1

```json
{
  "documents": [
    { "inputId": "name-input", "value": "John Doe" },
    { "inputId": "email-input", "value": "john@example.com" }
  ],
  "currentStep": 1
}
```

Status: `DRAFT` (campos da etapa 2 ainda faltando)

### Submeter Etapa 2

```json
{
  "documents": [
    { "inputId": "id-doc-input", "assetId": "asset-uuid" },
    { "inputId": "selfie-input", "assetId": "asset-uuid" }
  ],
  "currentStep": 2,
  "userContextId": "uc-uuid-from-step-1"
}
```

Status: `CREATED` (todos os campos obrigatórios submetidos)

**Importante:** Inclua o `userContextId` da resposta da Etapa 1 para vincular os documentos ao mesmo contexto de usuário.

---

## Reenvio (Após Revisão Solicitada)

Quando um aprovador sinaliza campos específicos para revisão, o usuário reenvia apenas esses campos:

### Passo 1: Verificar Quais Campos Precisam de Revisão

```
GET /{tenantId}/users/contexts/{userId}/{userContextId}
```

Procure documentos com `status: "REQUIRED_REVIEW"` -- esses são os campos a serem reenviados.

### Passo 2: Reenviar Campos Sinalizados

```json
{
  "documents": [
    { "inputId": "flagged-input-1", "assetId": "new-asset-uuid" },
    { "inputId": "flagged-input-2", "value": "corrected value" }
  ],
  "userContextId": "uc-uuid"
}
```

Transição de status: `REQUIRED_REVIEW → CREATED` (de volta na fila de aprovação).

---

## Fluxo de Aprovação Automática

Se o contexto tiver `autoApprove: true`:

1. Usuário submete documentos
2. Todos os documentos são imediatamente definidos como `APPROVED`
3. Status do contexto do usuário → `APPROVED`
4. Evento KYC de commerce é disparado
5. E-mail de aprovação enviado automaticamente
6. Nenhuma revisão manual necessária

---

## Cadeia de Validação (Interna)

A API realiza estas validações na submissão:

1. **Verificação do usuário** -- o e-mail deve estar verificado (a menos que passwordless esteja habilitado)
2. **Verificação de permissão** -- o usuário só pode submeter seus próprios documentos (a menos que seja Admin/Integration)
3. **Validação de assets** -- todos os assets de arquivo devem existir e estar no Cloudinary
4. **Validação específica por tipo** -- cada valor é validado conforme o tipo de input
5. **Verificação de campos obrigatórios** -- todos os inputs obrigatórios devem ter documentos antes do status `CREATED`
6. **Atribuição de aprovador** -- deve ser definido se `requireSpecificApprover: true`
7. **Unicidade de CPF** -- não são permitidos valores duplicados de CPF por tenant
8. **Biometria multiface** -- selfie validada contra serviço externo (requer CPF + data de nascimento)

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | BadRequestException | Documento obrigatório ausente/valor inválido | Verifique os requisitos do campo e o formato do valor |
| 400 | CPfDuplicatedException | CPF já utilizado por outro usuário neste tenant | Cada CPF deve ser único por tenant |
| 400 | InvalidCPFException | Falha na validação do checksum do CPF | Verifique o número do CPF (11 dígitos, checksum válido) |
| 400 | MaxSubmissionReachedException | Usuário excedeu o máximo de submissões para este contexto | Limite de `maxSubmissions` do contexto atingido |
| 400 | TenantContextDisabledException | Contexto está inativo | Entre em contato com o administrador para ativar o contexto |
| 403 | NotAuthorizedAttachmentDocumentToContextException | Usuário não pode submeter neste contexto | Verifique permissões e acesso ao contexto |
| 403 | UserNeedVerifyAccount | E-mail não verificado | Complete a verificação de e-mail primeiro |
| 404 | UserContextNotFoundException | Contexto do usuário não encontrado | Verifique o userContextId |
| 500 | ItWasNotPossibleRegisterMutiplaceExpection | Falha na validação biométrica | Tente novamente com uma selfie mais nítida |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | CPF rejeitado no reenvio | O CPF deve ser único por tenant. Se outro usuário já possui este CPF, a submissão falha |
| 2 | Selfie multiface falha | O usuário deve ter submetido documentos de CPF e data de nascimento ANTES da selfie. O serviço biométrico requer esses dados |
| 3 | Status permanece DRAFT | Nem todos os campos obrigatórios foram submetidos. Verifique os campos com `mandatory: true` e submeta os que faltam |
| 4 | Upload de arquivo rejeitado | Faça upload do arquivo no serviço de assets primeiro, obtenha o `assetId`, e depois referencie-o na submissão do documento |
| 5 | Reenvio cria novo contexto | Inclua `userContextId` ao reenviar para vincular ao contexto existente ao invés de criar um novo |
| 6 | Valor de `simple_select` rejeitado | O valor submetido deve corresponder exatamente a um dos `options[].value` definidos no input |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Configuração de Contexto | Formulários devem ser configurados antes da submissão | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Gerenciamento de Inputs | Campos do formulário devem existir | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Aprovação KYC | Administrador revisa submissões | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Gerenciamento de Usuários | O usuário deve existir primeiro | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
