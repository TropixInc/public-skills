---
id: FLOW_KYC_SUBMISSION
title: "KYC - Submissão de Documentos"
module: offpix/kyc
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - submission
  - documents
  - verification
  - user
depends_on:
  - KYC_API_REFERENCE
---

# Submissão de Documentos KYC

## Visão Geral

Este fluxo cobre como os usuários submetem documentos KYC para verificação. O processo é: buscar a definição do formulário (inputs), coletar dados do usuário, opcionalmente fazer upload de arquivos, submeter documentos e acompanhar a submissão através do pipeline de aprovação. As submissões seguem o ciclo de vida de status: `DRAFT -> CREATED -> APPROVED / DENIED / REQUIRED_REVIEW`.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT do usuário que está submetendo | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| `userId` | UUID do Usuário (normalmente o usuário autenticado) | Fluxo de autenticação / perfil do usuário |
| E-mail verificado | O usuário deve ter um e-mail verificado (a menos que passwordless esteja habilitado) | Verificação de e-mail no cadastro |
| Slug ou ID do contexto | O contexto KYC para submissão | Administrador fornece o slug; ou obtenha da lista de contextos |

## Ciclo de Vida de Status

```
                     +-----------+
                     |   DRAFT   |  (incomplete submission)
                     +-----+-----+
                           |
                     all mandatory fields submitted
                           |
                     +-----v-----+
              +------|  CREATED  |------+
              |      +-----+-----+      |
              |            |            |
         approve      require-review   reject
              |            |            |
        +-----v-----+ +---v--------+ +-v--------+
        |  APPROVED  | | REQUIRED   | |  DENIED  |
        +------------+ |  _REVIEW   | +----------+
                       +---+--------+  (terminal)
                           |
                      user resubmits
                           |
                     +-----v-----+
                     |  CREATED  |  (back in queue)
                     +-----------+
```

| Status | Significado | Acionado por |
|--------|-------------|-------------|
| `DRAFT` | Submissão iniciada mas nem todos os campos obrigatórios foram preenchidos | Submissão parcial de documentos |
| `CREATED` | Todos os documentos obrigatórios submetidos, aguardando revisão | Preenchimento de todos os campos obrigatórios |
| `REQUIRED_REVIEW` | Aprovador sinalizou campos específicos para reenvio | Ação do administrador/aprovador |
| `APPROVED` | Todos os documentos aprovados | Ação do administrador ou aprovação automática |
| `DENIED` | Submissão rejeitada (terminal, sem possibilidade de reenvio) | Ação do administrador |

---

## Fluxo: Buscar Definição do Formulário

Antes de submeter documentos, o frontend busca a definição do formulário para saber quais campos exibir.

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/tenants/{tenantId}/tenant-input/slug/{slug}` | Nenhuma (público) |

**Exemplo:**
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
    "description": "Enter your CPF number (11 digits)",
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
    "label": "Date of Birth",
    "description": "You must be at least 18 years old",
    "type": "birthdate",
    "order": 3,
    "mandatory": true,
    "active": true,
    "attributeName": "birthdate",
    "step": 1,
    "options": []
  },
  {
    "id": "input-uuid-4",
    "label": "Country",
    "description": "Select your country",
    "type": "simple_select",
    "order": 4,
    "mandatory": true,
    "active": true,
    "attributeName": "country",
    "step": 1,
    "options": [
      { "label": "Brazil", "value": "BR" },
      { "label": "United States", "value": "US" },
      { "label": "Portugal", "value": "PT" }
    ]
  },
  {
    "id": "input-uuid-5",
    "label": "Government ID",
    "description": "Upload front and back of your ID document",
    "type": "identification_document",
    "order": 5,
    "mandatory": true,
    "active": true,
    "attributeName": "govId",
    "step": 2,
    "options": []
  },
  {
    "id": "input-uuid-6",
    "label": "Selfie Verification",
    "description": "Take a selfie for biometric verification",
    "type": "multiface_selfie",
    "order": 6,
    "mandatory": true,
    "active": true,
    "attributeName": "selfie",
    "step": 2,
    "options": []
  }
]
```

**Campos-chave para renderização:**
- `type`: determina o componente de UI (campo de texto, upload de arquivo, dropdown de seleção, etc.)
- `mandatory`: se o campo é obrigatório
- `step`: a qual etapa do formulário este campo pertence (para formulários multi-etapas)
- `order`: ordem de exibição dentro da etapa
- `options`: opções disponíveis para inputs do tipo select
- `active`: renderize apenas inputs ativos

---

## Fluxo: Upload de Arquivos

Para tipos de input baseados em arquivo (`file`, `image`, `identification_document`, `multiface_selfie`), os arquivos devem ser enviados para o Cloudinary primeiro para obter um `assetId`.

### Passo 1: Obter Assinatura de Upload

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| POST | `/cloudinary/get-signature` | Bearer (Usuário) |

**Requisição:**
```json
{
  "folder": "kyc-documents"
}
```

**Resposta (200):**
```json
{
  "signature": "abcdef1234567890",
  "timestamp": 1705312000,
  "cloudName": "w3block",
  "apiKey": "123456789",
  "folder": "kyc-documents"
}
```

### Passo 2: Upload para o Cloudinary

Use a assinatura retornada para fazer upload do arquivo diretamente na API de upload do Cloudinary. O upload retorna um `assetId` que você referencia na submissão do documento.

### Passo 3: Referenciar na Submissão

Use o `assetId` no array `documents` ao submeter:

```json
{
  "inputId": "input-uuid-5",
  "assetId": "asset-uuid-from-upload"
}
```

---

## Fluxo: Submeter Documentos

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{tenantId}/users/documents/{userId}/context/{contextId}` | Bearer (User, Admin, KycApprover) | application/json |

### Requisição Mínima (apenas campos de texto, etapa única)

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
      "value": "1990-05-15"
    },
    {
      "inputId": "input-uuid-4",
      "value": "BR"
    }
  ]
}
```

### Requisição Completa (todos os tipos de campo, multi-etapas, com aprovador)

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
      "value": "1990-05-15"
    },
    {
      "inputId": "input-uuid-4",
      "value": "BR"
    },
    {
      "inputId": "input-uuid-5",
      "value": {
        "docType": "rg",
        "document": "123456789"
      },
      "assetId": "asset-uuid-front"
    },
    {
      "inputId": "input-uuid-6",
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

### Resposta (201)

```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "created",
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-1",
      "status": "created",
      "simpleValue": "John Doe"
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-2",
      "status": "created",
      "simpleValue": "12345678901"
    },
    {
      "id": "doc-uuid-3",
      "inputId": "input-uuid-3",
      "status": "created",
      "simpleValue": "1990-05-15"
    },
    {
      "id": "doc-uuid-4",
      "inputId": "input-uuid-4",
      "status": "created",
      "simpleValue": "BR"
    }
  ]
}
```

### Determinação de Status

- Se todos os campos obrigatórios para a etapa atual (ou todas as etapas) forem submetidos: status = `CREATED` (pronto para revisão)
- Se alguns campos obrigatórios estiverem faltando: status = `DRAFT` (incompleto)
- Se `autoApprove: true` no TenantContext: status = `APPROVED` imediatamente, sem necessidade de revisão manual

---

## Fluxo: Submissão Multi-Etapas

Para formulários com múltiplas etapas, submeta documentos incrementalmente usando o parâmetro `currentStep`. O backend só valida campos obrigatórios até a etapa especificada.

### Passo 1: Submeter Primeira Página

```json
{
  "documents": [
    { "inputId": "input-uuid-1", "value": "John Doe" },
    { "inputId": "input-uuid-2", "value": "12345678901" },
    { "inputId": "input-uuid-3", "value": "1990-05-15" },
    { "inputId": "input-uuid-4", "value": "BR" }
  ],
  "currentStep": 1
}
```

**Resultado:** Status = `DRAFT` (campos obrigatórios da etapa 2 ainda faltando). A resposta inclui `userContextId` para a próxima etapa.

### Passo 2: Submeter Segunda Página

```json
{
  "documents": [
    { "inputId": "input-uuid-5", "value": { "docType": "rg", "document": "123456789" }, "assetId": "asset-uuid-front" },
    { "inputId": "input-uuid-6", "assetId": "asset-uuid-selfie" }
  ],
  "currentStep": 2,
  "userContextId": "uc-uuid-from-step-1"
}
```

**Resultado:** Status = `CREATED` (todos os campos obrigatórios agora submetidos).

**Importante:** Sempre inclua o `userContextId` da resposta do Passo 1 ao submeter etapas subsequentes. Isso garante que os documentos sejam vinculados ao mesmo contexto de usuário ao invés de criar um novo.

---

## Fluxo: Reenvio Após Revisão

Quando um aprovador sinaliza campos específicos via "solicitar revisão", o usuário pode reenviar apenas esses campos.

### Passo 1: Verificar Quais Campos Precisam de Revisão

```
GET /{tenantId}/users/contexts/{userId}/{userContextId}
```

Procure documentos com `status: "requiredReview"` -- esses são os campos que precisam de reenvio.

**Exemplo de trecho da resposta:**
```json
{
  "status": "requiredReview",
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-1",
      "status": "created",
      "simpleValue": "John Doe"
    },
    {
      "id": "doc-uuid-5",
      "inputId": "input-uuid-5",
      "status": "requiredReview",
      "assetId": "asset-uuid-old"
    },
    {
      "id": "doc-uuid-6",
      "inputId": "input-uuid-6",
      "status": "requiredReview",
      "assetId": "asset-uuid-old-selfie"
    }
  ],
  "logs": [
    {
      "moderatorId": "admin-uuid",
      "status": "requiredReview",
      "inputIds": ["input-uuid-5", "input-uuid-6"],
      "reason": "ID document is expired, selfie is blurry",
      "registerAt": "2026-01-16T14:00:00Z"
    }
  ]
}
```

### Passo 2: Reenviar Campos Sinalizados

```json
{
  "documents": [
    { "inputId": "input-uuid-5", "value": { "docType": "rg", "document": "123456789" }, "assetId": "new-asset-uuid-front" },
    { "inputId": "input-uuid-6", "assetId": "new-asset-uuid-selfie" }
  ],
  "userContextId": "uc-uuid"
}
```

**Resultado:** Os documentos retornam ao status `CREATED`. O contexto do usuário retorna para `CREATED` (de volta na fila de revisão).

---

## Processamento Especial por Tipo de Input

### Processamento de CPF

1. Caracteres não numéricos são removidos (ex.: `"123.456.789-01"` se torna `"12345678901"`)
2. Comprimento validado: deve ter exatamente 11 dígitos
3. Checksum validado usando o algoritmo mod-11 do CPF brasileiro
4. Verificação de unicidade no tenant: nenhum outro usuário no mesmo tenant pode ter o mesmo CPF
5. Armazenado apenas como dígitos em `simpleValue`

### Processamento de USER_NAME

1. Valor armazenado como `simpleValue`
2. Atualiza automaticamente o campo `name` do registro de perfil do usuário
3. Isso significa que o nome de exibição do usuário muda em toda a plataforma

### Processamento de MULTIFACE_SELFIE

1. Valida que o usuário já submeteu um documento de CPF (qualquer contexto, mesmo tenant)
2. Valida que o usuário já submeteu um documento de data de nascimento (qualquer contexto, mesmo tenant)
3. Faz upload do asset da selfie
4. Chama o serviço externo de validação biométrica com a selfie, CPF e data de nascimento
5. Se a validação biométrica falhar, lança `ItWasNotPossibleRegisterMutiplaceExpection`
6. Armazenado como `assetId`

### Processamento de COMMERCE_PRODUCT

1. Valida que o `productId` referenciado existe no serviço de commerce
2. Valida quantidade e IDs de variantes opcionais
3. Armazenado como `complexValue`

### Processamento de BIRTHDATE

1. Valida que a data é uma data ISO válida
2. Verifica requisito de idade mínima (padrão: 18 anos a partir da data atual)
3. Armazenado como `simpleValue`

### Processamento de SIMPLE_SELECT

1. Valida que o valor submetido corresponde a um dos `options[].value` da definição do input
2. Se `isMultiple` for true no input, aceita um array de valores (cada um deve corresponder a uma opção)
3. Armazenado como `simpleValue`

---

## Cadeia de Validação

O backend realiza estas validações em ordem em cada submissão:

1. **Verificação do usuário** -- o e-mail deve estar verificado (a menos que passwordless esteja habilitado para o tenant)
2. **Verificação de permissão** -- o usuário só pode submeter seus próprios documentos (a menos que tenha o papel de Admin ou Integration)
3. **Verificação de contexto ativo** -- o TenantContext deve estar ativo
4. **Verificação de máximo de submissões** -- o usuário não excedeu o limite de `maxSubmissions` para este contexto
5. **Validação de assets** -- todos os valores de `assetId` referenciados devem corresponder a assets existentes no Cloudinary
6. **Validação específica por tipo** -- cada valor de documento é validado de acordo com as regras do seu tipo de input
7. **Verificação de campos obrigatórios** -- se `currentStep` cobre todas as etapas, todos os inputs obrigatórios devem ter documentos
8. **Atribuição de aprovador** -- se `requireSpecificApprover: true`, o `approverUserId` deve ser fornecido
9. **Unicidade de CPF** -- não são permitidos valores duplicados de CPF por tenant
10. **Biometria multiface** -- se o tipo for multiface_selfie, o serviço biométrico é chamado

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | `BadRequestException` | Documento obrigatório ausente ou valor inválido | Verifique os requisitos do campo e o formato do valor |
| 400 | `CPfDuplicatedException` | CPF já utilizado por outro usuário neste tenant | Cada CPF deve ser único por tenant |
| 400 | `InvalidCPFException` | Falha na validação do checksum do CPF | Verifique se o CPF tem 11 dígitos com checksum válido |
| 400 | `MaxSubmissionReachedException` | Usuário excedeu o máximo de submissões para este contexto | Limite de `maxSubmissions` do contexto atingido |
| 400 | `TenantContextDisabledException` | Contexto está inativo | Entre em contato com o administrador para ativar o contexto |
| 403 | `NotAuthorizedAttachmentDocumentToContextException` | Usuário não pode submeter neste contexto | Verifique permissões e acesso ao contexto |
| 403 | `UserNeedVerifyAccount` | E-mail não verificado | Complete a verificação de e-mail primeiro |
| 404 | `UserContextNotFoundException` | Contexto do usuário não encontrado (`userContextId` inválido) | Verifique o `userContextId` de uma submissão anterior |
| 500 | `ItWasNotPossibleRegisterMutiplaceExpection` | Falha no serviço de validação biométrica | Tente novamente com uma selfie mais nítida; certifique-se de que CPF e data de nascimento foram submetidos primeiro |

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Erro de e-mail não verificado | Os usuários devem verificar seu e-mail antes de submeter documentos KYC. Verifique a exceção `UserNeedVerifyAccount`. Tenants com passwordless estão isentos |
| 2 | Rejeição por CPF duplicado | Valores de CPF devem ser únicos por tenant. Se outro usuário já submeteu o mesmo CPF, a submissão falha com `CPfDuplicatedException` |
| 3 | Selfie multiface falha | O serviço biométrico requer que documentos de CPF e data de nascimento existam para o usuário ANTES de submeter a selfie. Submeta esses campos primeiro |
| 4 | Status permanece DRAFT | Nem todos os campos obrigatórios foram submetidos. Verifique quais inputs têm `mandatory: true` e certifique-se de que todos estão incluídos |
| 5 | Reenvio cria novo contexto | Ao reenviar após "solicitar revisão", sempre inclua `userContextId` para vincular ao contexto existente |
| 6 | `maxSubmissions` excedido | O contexto tem um limite de submissões. Uma vez atingido, nenhuma nova submissão é permitida para aquele par usuário/contexto |
| 7 | Valor de `simple_select` rejeitado | O valor submetido deve corresponder exatamente a um dos `options[].value` definidos no input. Verifique a sensibilidade a maiúsculas/minúsculas |
| 8 | Upload de arquivo não aceito | Faça upload do arquivo para o Cloudinary primeiro via serviço de assets, obtenha o `assetId`, e depois referencie-o na submissão do documento |

---

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Aprovação KYC | Administrador revisa e atua nas submissões | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Configuração KYC | Configuração de contexto e inputs do tenant | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Configuração de Contexto | Como contextos KYC são criados | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Gerenciamento de Inputs | Como campos de formulário são definidos | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Autenticação Sign-In | Obter JWT para submissão | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| Referência da API | Detalhes completos de endpoints e DTOs | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
