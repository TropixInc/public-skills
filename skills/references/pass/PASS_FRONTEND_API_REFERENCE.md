---
id: PASS_API_REFERENCE
title: "Pass - Referência da API"
module: pass
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - api
  - pass
---

# Referência da API de Pass

Referência consolidada de todos os endpoints, schemas, enums e erros do módulo Pass W3block.

> **Documentação interativa (Swagger):** Para testar endpoints diretamente e ver os schemas sempre atualizados, acesse a documentação Swagger disponível em cada serviço (ex: `https://pass.w3block.io/api-docs`). A documentação abaixo é um guia complementar.

---

## Base URLs

| Serviço | Base URL | Descrição |
|---------|----------|-----------|
| Pass API | `https://pass.w3block.io` | Passes, benefícios, operadores, share codes |
| Commerce API | `https://commerce.w3block.io` | Integração com checkout (passShareCodeData) |
| Identity API | `https://id.w3block.io` | Autenticação, tokens |

> **Nota:** Em ambientes de staging/dev, os domínios podem variar conforme o ambiente configurado.

---

## Autenticação

Todos os endpoints (exceto os marcados como públicos) exigem:

```
Authorization: Bearer <access_token>
```

O `access_token` é obtido via login na Identity API (`https://id.w3block.io`). Consulte a documentação da Identity API para detalhes sobre:
- Endpoint de login e obtenção do token
- Refresh token e renovação de sessão
- Tempo de expiração do token

#### Exemplo de requisição autenticada

```bash
curl -X GET "https://pass.w3block.io/token-passes/tenants/{tenantId}" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json"
```

```typescript
const response = await fetch(
  `https://pass.w3block.io/token-passes/tenants/${tenantId}`,
  {
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    },
  }
);
const data = await response.json();
```

### Roles Disponíveis

| Role | Descrição |
|------|-----------|
| `superAdmin` | Administrador da plataforma |
| `admin` | Administrador do tenant |
| `operator` | Operador de benefícios (verificar/registrar uso) |
| `user` | Usuário final (dono de token) |
| `integration` | Integração via API key |
| `application` | Aplicação do sistema |

### Permissões por Endpoint

| Ação | Roles permitidos |
|------|-----------------|
| CRUD de passes | superAdmin, integration, application, admin |
| CRUD de benefícios | superAdmin, integration, application, admin |
| Verificar/registrar uso | superAdmin, integration, application, admin, operator |
| Self-use (usuário) | superAdmin, integration, application, user |
| Ver benefícios por edition | todos (incl. user) |
| Obter secret do QR code | superAdmin, integration, application, user |
| Share codes (criar) | superAdmin, integration, application, admin |
| Share codes (consultar) | **Público** (sem auth) |

---

## Endpoints

### 1. Criar Token Pass

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-passes/tenants/{tenantId}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "collectionId": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
  "tokenName": "VIP Pass 2024",
  "name": "Pass VIP Evento",
  "description": "Acesso exclusivo aos benefícios VIP do evento",
  "rules": "Válido para maiores de 18 anos",
  "imageUrl": "https://cdn.example.com/pass-vip.png",
  "tokenPassBenefits": [
    {
      "name": "Acesso Backstage",
      "description": "Entrada na área backstage",
      "type": "physical",
      "useLimit": 1,
      "eventStartsAt": "2024-06-15T18:00:00-03:00",
      "eventEndsAt": "2024-06-15T23:59:00-03:00",
      "dynamicQrCode": true,
      "allowSelfUse": false
    }
  ]
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `collectionId` | `uuid` | Sim | ID da collection de tokens (NFT contract) |
| `tokenName` | `string` | Sim | Nome do token associado |
| `name` | `string` | Sim | Nome do pass |
| `description` | `string` | Não | Descrição do pass |
| `rules` | `string` | Não | Regras gerais do pass |
| `imageUrl` | `string` | Não | URL da imagem do pass |
| `tokenPassBenefits` | `CreateTokenPassBenefitDto[]` | Não | Benefícios criados inline com o pass |

#### Response Body (201 Created)

```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "tokenName": "VIP Pass 2024",
  "contractAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "chainId": "137",
  "name": "Pass VIP Evento",
  "description": "Acesso exclusivo aos benefícios VIP do evento",
  "rules": "Válido para maiores de 18 anos",
  "imageUrl": "https://cdn.example.com/pass-vip.png",
  "pdfTemplate": null,
  "tenantId": "c9d1e2f3-4567-89ab-cdef-0123456789ab",
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

---

### 2. Listar Token Passes

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-passes/tenants/{tenantId}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin) |

#### Query Params

| Parâmetro | Tipo | Obrigatório | Default | Descrição |
|-----------|------|-------------|---------|-----------|
| `page` | `number` | Não | `1` | Página |
| `limit` | `number` | Não | `10` | Itens por página |
| `search` | `string` | Não | — | Busca por nome |
| `sortBy` | `string` | Não | — | Campo para ordenação |
| `orderBy` | `OrderByEnum` | Não | — | `ASC` ou `DESC` |
| `chainId` | `ChainId` | Não | — | Filtrar por blockchain |
| `contractAddress` | `string` | Não | — | Filtrar por contrato |

#### Response Body (200 OK)

```json
{
  "items": [
    {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "tokenName": "VIP Pass 2024",
      "contractAddress": "0x1234...",
      "chainId": "137",
      "name": "Pass VIP Evento",
      "description": "...",
      "imageUrl": "https://cdn.example.com/pass-vip.png",
      "tenantId": "c9d1e2f3-...",
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ],
  "meta": {
    "itemCount": 1,
    "totalItems": 1,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

---

### 3. Detalhes de um Token Pass

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-passes/tenants/{tenantId}/{id}` |
| **Auth** | Bearer token (JWT) |

#### Response Body (200 OK)

Mesmo schema de `TokenPassEntityDto` (ver endpoint 1).

---

### 4. Atualizar Token Pass

| Campo | Valor |
|-------|-------|
| **Método** | `PATCH` |
| **Path** | `/token-passes/tenants/{tenantId}/{id}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "name": "Pass VIP Atualizado",
  "description": "Nova descrição",
  "rules": "Novas regras",
  "imageUrl": "https://cdn.example.com/pass-vip-v2.png"
}
```

Todos os campos são opcionais. Apenas os campos enviados serão atualizados.

---

### 5. Remover Token Pass

| Campo | Valor |
|-------|-------|
| **Método** | `DELETE` |
| **Path** | `/token-passes/tenants/{tenantId}/{id}` |
| **Auth** | Bearer token (JWT) |

#### Response: 204 No Content

---

### 6. Listar Passes por Operador

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-passes/tenants/{tenantId}/users/{userId}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |

Retorna apenas os passes que possuem benefícios atribuídos ao operador `userId`.

#### Query Params

Mesmos parâmetros de paginação do endpoint 2 (`page`, `limit`, `search`, `sortBy`, `orderBy`).

#### Response Body (200 OK)

Mesmo schema paginado de `TokenPassEntityDto`.

---

### 7. Benefícios por Edição do Token

Retorna os benefícios de um token pass para um `editionNumber` específico, incluindo status de uso e disponibilidade.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-passes/tenants/{tenantId}/{id}/token-editions/{editionNumber}/benefits` |
| **Auth** | Bearer token (todos os roles, incluindo user) |

#### Path Params

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `tenantId` | `string` | ID do tenant |
| `id` | `string` | ID do token pass |
| `editionNumber` | `number` | Número da edição do token (NFT) |

#### Query Params

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | `number` | Não | Página (default: 1) |
| `limit` | `number` | Não | Itens por página (default: 10) |
| `search` | `string` | Não | Busca por nome |
| `sortBy` | `string` | Não | Campo para ordenação |
| `orderBy` | `OrderByEnum` | Não | `ASC` ou `DESC` |
| `benefitId` | `uuid` | Não | Filtrar por benefício específico |
| `status` | `BenefitUseStatusEnum` | Não | Filtrar por status (`active`, `inactive`, `unavailable`) |

#### Response Body (200 OK)

```json
[
  {
    "id": "b1c2d3e4-5678-90ab-cdef-1234567890ab",
    "tokenPassId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "tokenPass": {
      "id": "f47ac10b-...",
      "tokenName": "VIP Pass 2024",
      "contractAddress": "0x1234...",
      "chainId": "137",
      "name": "Pass VIP Evento"
    },
    "tokenPassBenefitUsage": {
      "id": "usage-uuid",
      "editionNumber": 42,
      "uses": 1,
      "tokenPassBenefitId": "b1c2d3e4-..."
    },
    "status": "active",
    "name": "Acesso Backstage",
    "description": "Entrada na área backstage",
    "rules": "Apresentar documento com foto",
    "type": "physical",
    "useLimit": 3,
    "useAvailable": 2,
    "eventStartsAt": "2024-06-15T18:00:00-03:00",
    "eventEndsAt": "2024-06-15T23:59:00-03:00",
    "checkInStartsAt": "17:30:00-03:00",
    "checkInEndsAt": "22:00:00-03:00",
    "checkIn": null,
    "linkUrl": null,
    "linkRules": null,
    "dynamicQrCode": true,
    "statusMessage": null,
    "tokenPassBenefitAddresses": [
      {
        "id": "addr-uuid",
        "name": "Palco Principal",
        "street": "Av. Exemplo",
        "number": 1000,
        "city": "São Paulo",
        "state": "SP",
        "postalCode": "01234-567",
        "rules": "Entrada pelo portão B"
      }
    ],
    "tokenPassBenefitOperators": [],
    "createdAt": "2024-01-10T10:00:00Z",
    "updatedAt": "2024-01-10T10:00:00Z",
    "nextCheckInDates": [
      {
        "start": "2024-06-15T17:30:00-03:00",
        "end": "2024-06-15T22:00:00-03:00"
      }
    ]
  }
]
```

> **Nota:** Este endpoint retorna `TokenPassBenefitsPublicDto` que inclui `status`, `useAvailable`, `statusMessage` — campos calculados com base no uso do token específico.

---

### 8. Criar Benefício

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "name": "Acesso Backstage",
  "description": "Entrada na área backstage durante o evento",
  "rules": "Apresentar documento com foto",
  "type": "physical",
  "useLimit": 3,
  "eventStartsAt": "2024-06-15T18:00:00-03:00",
  "eventEndsAt": "2024-06-15T23:59:00-03:00",
  "checkIn": {
    "all": [
      { "start": "17:30", "end": "22:00" }
    ],
    "special": null
  },
  "linkUrl": null,
  "linkRules": null,
  "dynamicQrCode": true,
  "tokenPassId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "tokenPassBenefitAddresses": [
    {
      "name": "Palco Principal",
      "street": "Av. Exemplo",
      "number": 1000,
      "city": "São Paulo",
      "state": "SP",
      "postalCode": "01234-567",
      "rules": "Entrada pelo portão B"
    }
  ],
  "allowSelfUse": false,
  "usageRule": null,
  "timezoneOrUtcOffset": "America/Sao_Paulo"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | `string` | Sim | Nome do benefício |
| `description` | `string` | Não | Descrição |
| `rules` | `string` | Não | Regras de uso |
| `type` | `TokenPassBenefitTypeEnum` | Não | `digital` (default) ou `physical` |
| `useLimit` | `number` | Não | Limite de usos por token (null = ilimitado) |
| `eventStartsAt` | `datetime` | Não | Início do evento (ex: `2024-06-15T18:00:00-03:00`) |
| `eventEndsAt` | `datetime` | Não | Fim do evento |
| `checkIn` | `TokenPassBenefitCheckInConfigDto` | Não | Configuração de horários de check-in por dia |
| `linkUrl` | `string` | Não | URL do benefício digital |
| `linkRules` | `string` | Não | Regras do link |
| `dynamicQrCode` | `boolean` | Não | Se `true`, QR code usa secret dinâmico |
| `tokenPassId` | `uuid` | Sim | ID do token pass associado |
| `tokenPassBenefitAddresses` | `CreateAddressDto[]` | Não | Endereços para benefícios físicos |
| `allowSelfUse` | `boolean` | Não | Se `true`, usuário pode registrar uso próprio (default: `false`) |
| `requirements` | `object[]` | Não | Requisitos para usar o benefício |
| `usageRule` | `TokenPassBenefitUsageRuleDto` | Não | Regra de uso temporário |
| `timezoneOrUtcOffset` | `string` | Não | Timezone (default: `America/Sao_Paulo`) |

#### Response Body (201 Created)

```json
{
  "id": "b1c2d3e4-5678-90ab-cdef-1234567890ab",
  "name": "Acesso Backstage",
  "description": "Entrada na área backstage durante o evento",
  "rules": "Apresentar documento com foto",
  "type": "physical",
  "useLimit": 3,
  "eventStartsAt": "2024-06-15T18:00:00-03:00",
  "eventEndsAt": "2024-06-15T23:59:00-03:00",
  "checkIn": {
    "all": [{ "start": "17:30", "end": "22:00" }]
  },
  "linkUrl": null,
  "linkRules": null,
  "dynamicQrCode": true,
  "tokenPass": { "id": "f47ac10b-...", "name": "Pass VIP Evento" },
  "tokenPassId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "allowSelfUse": false,
  "nextCheckInDates": [
    { "start": "2024-06-15T17:30:00-03:00", "end": "2024-06-15T22:00:00-03:00" }
  ],
  "usageRule": null,
  "timezoneOrUtcOffset": "America/Sao_Paulo",
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

---

### 9. Listar Benefícios

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |

#### Query Params

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | `number` | Não | Página (default: 1) |
| `limit` | `number` | Não | Itens por página (default: 10) |
| `search` | `string` | Não | Busca por nome |
| `sortBy` | `string` | Não | Campo para ordenação |
| `orderBy` | `OrderByEnum` | Não | `ASC` ou `DESC` |
| `tokenPassId` | `string` | Não | Filtrar por token pass |
| `chainId` | `ChainId` | Não | Filtrar por blockchain |
| `contractAddress` | `string` | Não | Filtrar por contrato |

#### Response Body (200 OK)

Schema paginado de `TokenPassBenefitEntityDto`.

---

### 10. Detalhes de Benefício

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}` |
| **Auth** | Bearer token (JWT) |

#### Response Body (200 OK)

Schema `TokenPassBenefitEntityDto` (ver endpoint 8).

---

### 11. Atualizar Benefício

| Campo | Valor |
|-------|-------|
| **Método** | `PATCH` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin) |
| **Content-Type** | `application/json` |

#### Request Body

Todos os campos de `CreateTokenPassBenefitDto` (exceto `tokenPassId`), todos opcionais.

---

### 12. Remover Benefício

| Campo | Valor |
|-------|-------|
| **Método** | `DELETE` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin) |

#### Response: 204 No Content

---

### 13. Obter Secret do QR Code

Retorna o secret para geração do QR code que permite usar o benefício.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}/{editionNumber}/secret` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, user) |

#### Path Params

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `tenantId` | `string` | ID do tenant |
| `id` | `string` | ID do benefício |
| `editionNumber` | `number` | Número da edição do token |

#### Response Body (200 OK)

```json
{
  "secret": "a1b2c3d4e5f6g7h8i9j0"
}
```

> **Formato do QR Code:** Após obter o secret, montar a string: `{editionNumber},{userId},{secret},{benefitId}` — esta string deve ser codificada no QR code.

---

### 14. Verificar Benefício

Verifica se um benefício está disponível para uso pelo usuário. Retorna dados do benefício, usuário e usos.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}/verify` |
| **Auth** | Bearer token (todos os roles) |

#### Query Params (obrigatórios)

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `userId` | `uuid` | Sim | ID do usuário dono do token |
| `editionNumber` | `number` | Sim | Número da edição do token |
| `secret` | `string` | Sim | Secret obtido do QR code |

#### Response Body (200 OK)

```json
{
  "id": "usage-uuid",
  "editionNumber": 42,
  "tokenPassBenefit": {
    "id": "b1c2d3e4-...",
    "name": "Acesso Backstage",
    "description": "Entrada na área backstage",
    "type": "physical",
    "useLimit": 3,
    "dynamicQrCode": true,
    "allowSelfUse": false,
    "tokenPass": { "id": "f47ac10b-...", "name": "Pass VIP Evento" }
  },
  "tokenPassBenefitId": "b1c2d3e4-...",
  "uses": 1,
  "user": {
    "name": "João Silva",
    "email": "joao@example.com",
    "cpf": "12345678901",
    "phone": "11987654321"
  },
  "secret": "a1b2c3d4e5f6g7h8i9j0",
  "shareCodes": null,
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

#### Error States

| HTTP | Código | Descrição | Ação de Recuperação |
|------|--------|-----------|---------------------|
| 400 | `invalid-secret-hash` | Secret do QR code inválido | Solicitar novo QR code ao usuário |
| 400 | `secret-expired` | Secret expirado (QR code dinâmico) | Usuário deve gerar novo QR code |
| 400 | `undefined-benefit-start-date` | Benefício sem data de início definida | Admin deve configurar `eventStartsAt` |
| 400 | `benefit-not-started` | Benefício ainda não começou | Aguardar data de início do evento |
| 400 | `benefit-expired` | Benefício expirado | Não é possível usar — período encerrado |
| 400 | `temporary-usage-limit-reached` | Limite temporário atingido (usageRule) | Aguardar próximo período permitido |
| 400 | `usage-limit-reached` | Limite total de uso atingido | Não é possível usar — todos os usos consumidos |
| 400 | `wrong-checkin-time` | Fora do horário de check-in | Aguardar horário de check-in configurado |
| 400 | `undefined-checkin-times` | Horários de check-in não configurados | Admin deve configurar check-in |
| 403 | `invalid-token-owner` | Usuário não é dono do token | Verificar se o usuário possui o NFT |
| 403 | `self-use-not-allowed` | Self-use não habilitado | Benefício requer verificação por operador |
| 403 | `unsatisfied-benefit-requirements` | Requisitos não atendidos | Verificar requisitos do benefício |
| 403 | `unauthorized-operator` | Operador não autorizado | Operador não está atribuído a este benefício |
| 404 | `benefit-not-found` | Benefício não encontrado | Verificar `benefitId` |
| 404 | `token-not-found` | Token não encontrado | Verificar `editionNumber` e contrato |

---

### 15. Registrar Uso (Operador — por secret)

Registra o uso de um benefício como operador, usando os dados extraídos do QR code.

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}/register-use` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "userId": "d4e5f6a7-8901-2345-bcde-f67890123456",
  "editionNumber": 42,
  "secret": "a1b2c3d4e5f6g7h8i9j0"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `userId` | `uuid` | Sim | ID do usuário dono do token |
| `editionNumber` | `number` | Sim | Número da edição do token |
| `secret` | `string` | Sim | Secret do QR code |

#### Response Body (201 Created)

```json
{
  "id": "use-uuid",
  "editionNumber": 42,
  "tokenPassBenefit": { "id": "b1c2d3e4-...", "name": "Acesso Backstage" },
  "tokenPassBenefitId": "b1c2d3e4-...",
  "uses": 2,
  "createdAt": "2024-06-15T19:30:00Z",
  "updatedAt": "2024-06-15T19:30:00Z"
}
```

---

### 16. Registrar Uso (Operador — por userId ou CPF)

Registra o uso identificando o usuário por `userId` ou `cpf`, sem necessidade de QR code.

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}/register-use-by-user` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "userId": "d4e5f6a7-8901-2345-bcde-f67890123456"
}
```

Ou:

```json
{
  "cpf": "12345678901"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `userId` | `uuid` | Não* | ID do usuário |
| `cpf` | `string` | Não* | CPF do usuário (formato: `xxxxxxxxxxx`, 11 dígitos sem pontuação) |

> **Nota:** Enviar `userId` OU `cpf` — pelo menos um é obrigatório.

#### Response Body (201 Created)

Schema `TokenPassBenefitUsageEntityDto`.

---

### 17. Registrar Uso (Operador — por QR Code string)

Registra o uso enviando a string completa do QR code.

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/register-use-by-qrcode` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "qrcode": "42;d4e5f6a7-8901-2345-bcde-f67890123456;a1b2c3d4e5f6g7h8i9j0;b1c2d3e4-5678-90ab-cdef-1234567890ab"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `qrcode` | `string` | Sim | String do QR code no formato: `{editionNumber};{userId};{secret};{benefitId}` |

> **Nota:** Este endpoint usa ponto-e-vírgula (`;`) como separador na string do qrcode, diferente do formato usado pelo frontend SDK que usa vírgula (`,`). Verifique qual formato o seu scanner está gerando.

#### Response Body (201 Created)

Schema `TokenPassBenefitUsageEntityDto`.

---

### 18. Self-Use (Usuário)

Permite que o próprio dono do token registre o uso de um benefício.

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/{id}/use` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, user) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "userId": "d4e5f6a7-8901-2345-bcde-f67890123456",
  "editionNumber": 42
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `userId` | `uuid` | Sim | ID do usuário (deve ser o dono do token) |
| `editionNumber` | `number` | Sim | Número da edição do token |

> **Pré-requisito:** O benefício deve ter `allowSelfUse: true`. Caso contrário, retorna erro `self-use-not-allowed`.

#### Response Body (201 Created)

Schema `TokenPassBenefitUsageEntityDto`.

---

### 19. Listar Usos de Benefícios

Lista o histórico de usos de benefícios com dados do usuário.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/usages` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |

#### Query Params

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | `number` | Não | Página (default: 1) |
| `limit` | `number` | Não | Itens por página (default: 10) |
| `search` | `string` | Não | Busca por nome/email do usuário |
| `sortBy` | `string[]` | Não | Campo(s) para ordenação |
| `orderBy` | `OrderByEnum` | Não | `ASC` ou `DESC` |
| `tokenPassId` | `uuid` | Não | Filtrar por token pass |
| `benefitId` | `uuid` | Não | Filtrar por benefício |
| `editionNumber` | `number` | Não | Filtrar por edição |
| `createdAt` | `datetime` | Não | Filtrar por data (ex: `2024-01-30T10:30:40-03:00`) |

#### Response Body (200 OK)

```json
{
  "items": [
    {
      "id": "use-uuid",
      "editionNumber": 42,
      "tokenPassBenefit": {
        "id": "b1c2d3e4-...",
        "name": "Acesso Backstage",
        "type": "physical"
      },
      "tokenPassBenefitId": "b1c2d3e4-...",
      "uses": 2,
      "user": {
        "name": "João Silva",
        "email": "joao@example.com",
        "cpf": "12345678901",
        "phone": "11987654321"
      },
      "createdAt": "2024-06-15T19:30:00Z",
      "updatedAt": "2024-06-15T19:30:00Z"
    }
  ],
  "meta": {
    "itemCount": 1,
    "totalItems": 50,
    "itemsPerPage": 10,
    "totalPages": 5,
    "currentPage": 1
  }
}
```

---

### 20. Exportar Relatório XLS de Usos

Solicita a geração de um relatório XLS de usos de benefícios.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-benefits/tenants/{tenantId}/usages/xls` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin, operator) |

#### Query Params

Mesmos parâmetros de filtro do endpoint 19 (`tokenPassId`, `benefitId`, `editionNumber`, `search`, `createdAt`, etc.).

#### Response Body (201 Created)

```json
{
  "id": "export-uuid",
  "tenantId": "tenant-uuid",
  "type": "benefit_usages_report",
  "status": "pending",
  "readyForDownloadDate": null,
  "expiresIn": null,
  "assetId": "asset-uuid",
  "asset": {
    "id": "asset-uuid",
    "tenantId": "tenant-uuid",
    "type": "document",
    "status": "waiting_upload",
    "directLink": null
  },
  "params": { "benefitId": "b1c2d3e4-..." },
  "createdAt": "2024-06-15T19:30:00Z"
}
```

> **Fluxo de download:** Após solicitar o export, fazer polling via `GET /exports/tenants/{tenantId}/{exportId}` até o status mudar para `ready_for_download`. O link de download estará em `asset.directLink`.

---

### 21-25. CRUD de Endereços de Benefícios

Gerencia endereços físicos associados a benefícios do tipo `physical`.

#### Criar Endereço

```
POST /token-pass-benefit-addresses/tenants/{tenantId}
```

```json
{
  "name": "Palco Principal",
  "street": "Av. Exemplo",
  "number": 1000,
  "city": "São Paulo",
  "state": "SP",
  "postalCode": "01234-567",
  "rules": "Entrada pelo portão B",
  "tokenPassBenefitId": "b1c2d3e4-5678-90ab-cdef-1234567890ab"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | `string` | Sim | Nome do local |
| `street` | `string` | Não | Rua |
| `number` | `number` | Não | Número |
| `city` | `string` | Sim | Cidade |
| `state` | `string` | Sim | Estado |
| `postalCode` | `string` | Não | CEP |
| `rules` | `string` | Não | Regras do local |
| `tokenPassBenefitId` | `uuid` | Sim | ID do benefício |

#### Outros endpoints

| Método | Path | Descrição |
|--------|------|-----------|
| `GET` | `.../tenants/{tenantId}` | Listar endereços (paginado) |
| `GET` | `.../tenants/{tenantId}/{id}` | Detalhes de um endereço |
| `PATCH` | `.../tenants/{tenantId}/{id}` | Atualizar endereço |
| `DELETE` | `.../tenants/{tenantId}/{id}` | Remover endereço |

---

### 26. Criar Share Code

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/token-pass-share-codes/tenants/{tenantId}` |
| **Auth** | Bearer token (roles: superAdmin, integration, application, admin) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "contractAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "chainId": "137",
  "editionNumber": 42,
  "data": {
    "destinationUserName": "Maria",
    "message": "Feliz aniversário!"
  },
  "expiresIn": "2024-12-31T23:59:59Z"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `contractAddress` | `string` | Sim | Endereço do contrato do token |
| `chainId` | `string` | Sim | ID da blockchain (default: `"137"` — Polygon) |
| `editionNumber` | `number` | Sim | Número da edição do token |
| `data` | `object` | Não | Dados customizáveis (mensagem, destinatário, etc.) |
| `expiresIn` | `datetime` | Não | Data de expiração do share code |

#### Response Body (201 Created)

```json
{
  "id": "share-uuid",
  "tenantId": "tenant-uuid",
  "code": "ABC123XYZ",
  "data": { "destinationUserName": "Maria", "message": "Feliz aniversário!" },
  "editionNumber": 42,
  "expiresIn": "2024-12-31T23:59:59Z",
  "tokenPassId": "f47ac10b-...",
  "createdAt": "2024-06-15T10:00:00Z",
  "updatedAt": "2024-06-15T10:00:00Z"
}
```

---

### 27. Consultar Share Code (Público)

Busca detalhes de um share code com benefícios e metadados do token pass. **Não requer autenticação.**

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/token-pass-share-codes/tenants/{tenantId}/{code}` |
| **Auth** | Nenhuma (público) |

#### Path Params

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `tenantId` | `string` | ID do tenant |
| `code` | `string` | Código do share (ex: `ABC123XYZ`) |

#### Response Body (200 OK)

Retorna `TokenPassShareCodeEntityDto` com benefícios e metadados do token pass associado.

---

### 28-32. CRUD de Operadores de Benefícios

Gerencia quais usuários são operadores de quais benefícios.

#### Criar Operador

```
POST /token-pass-benefit-operators/tenants/{tenantId}
```

```json
{
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "b1c2d3e4-5678-90ab-cdef-1234567890ab"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `userId` | `uuid` | Sim | ID do usuário que será operador |
| `tokenPassBenefitId` | `uuid` | Sim | ID do benefício que o operador gerencia |

#### Listar Operadores de um Benefício

```
GET /token-pass-benefit-operators/tenants/{tenantId}/benefits/{benefitId}
```

Retorna lista paginada de operadores com informações do usuário (`name`, `email`, `walletAddress`, `associatedTokens`).

#### Outros endpoints

| Método | Path | Descrição |
|--------|------|-----------|
| `GET` | `.../tenants/{tenantId}` | Listar todos os operadores (paginado) |
| `GET` | `.../tenants/{tenantId}/{id}` | Detalhes de um operador |
| `DELETE` | `.../tenants/{tenantId}/{id}` | Remover operador |

---

### 33-34. Exports

#### Listar Exportações

```
GET /exports/tenants/{tenantId}
```

Retorna lista paginada de `ExportEntityDto` com status de cada exportação.

#### Consultar Exportação

```
GET /exports/tenants/{tenantId}/{exportId}
```

Retorna `ExportEntityDto` com status atual e link de download quando pronto.

```json
{
  "id": "export-uuid",
  "tenantId": "tenant-uuid",
  "type": "benefit_usages_report",
  "status": "ready_for_download",
  "readyForDownloadDate": "2024-06-15T19:31:00Z",
  "expiresIn": "2024-06-22T19:31:00Z",
  "assetId": "asset-uuid",
  "asset": {
    "id": "asset-uuid",
    "type": "document",
    "status": "associated",
    "directLink": "https://<asset-provider-url>/benefit-usages-report.xlsx"
  }
}
```

---

## Schemas

### TokenPassEntityDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do token pass |
| `tokenName` | `string` | Nome do token na blockchain |
| `contractAddress` | `string` | Endereço do contrato (NFT) |
| `chainId` | `ChainId` | ID da blockchain (default: `137` — Polygon) |
| `name` | `string` | Nome do pass |
| `description` | `string` | Descrição |
| `rules` | `string` | Regras gerais |
| `imageUrl` | `string` | URL da imagem |
| `pdfTemplate` | `string` | Template PDF (para geração de tickets) |
| `tenantId` | `string` | ID do tenant |
| `createdAt` | `datetime` | Data de criação |
| `updatedAt` | `datetime` | Data de atualização |

### TokenPassBenefitEntityDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do benefício |
| `name` | `string` | Nome do benefício |
| `description` | `string` | Descrição |
| `rules` | `string` | Regras de uso |
| `type` | `TokenPassBenefitTypeEnum` | `digital` ou `physical` |
| `useLimit` | `number` | Limite de usos por token (null = ilimitado) |
| `eventStartsAt` | `datetime` | Início do evento |
| `eventEndsAt` | `datetime` | Fim do evento |
| `checkInStartsAt` | `time` | **[DEPRECADO]** Horário de início do check-in. Use o campo `checkIn` (objeto) no lugar. |
| `checkInEndsAt` | `time` | **[DEPRECADO]** Horário de fim do check-in. Use o campo `checkIn` (objeto) no lugar. |
| `checkIn` | `TokenPassBenefitCheckInConfigDto` | Configuração de check-in por dia da semana |
| `linkUrl` | `string` | URL do benefício digital |
| `linkRules` | `string` | Regras do link |
| `dynamicQrCode` | `boolean` | Se usa QR code dinâmico com secret |
| `tokenPass` | `TokenPassEntityDto` | Token pass associado |
| `tokenPassId` | `uuid` | ID do token pass |
| `allowSelfUse` | `boolean` | Se permite self-use pelo usuário (default: `false`) |
| `nextCheckInDates` | `NextCheckInDateDto[]` | Próximos horários de check-in calculados |
| `usageRule` | `TokenPassBenefitUsageRuleDto` | Regra de uso temporário |
| `timezoneOrUtcOffset` | `string` | Timezone (default: `America/Sao_Paulo`) |
| `createdAt` | `datetime` | Data de criação |
| `updatedAt` | `datetime` | Data de atualização |

### TokenPassBenefitsPublicDto

Versão pública do benefício com informações de uso por edição — retornada pelo endpoint 7.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do benefício |
| `tokenPassId` | `uuid` | ID do token pass |
| `tokenPass` | `TokenPassEntityDto` | Token pass |
| `tokenPassBenefitUsage` | `UsageEntityDto` | Dados de uso para esta edição |
| `status` | `BenefitUseStatusEnum` | Status calculado: `active`, `inactive`, `unavailable` |
| `name` | `string` | Nome |
| `description` | `string` | Descrição |
| `rules` | `string` | Regras |
| `type` | `TokenPassBenefitTypeEnum` | Tipo |
| `useLimit` | `number` | Limite de usos |
| `useAvailable` | `number` | Usos disponíveis restantes |
| `eventStartsAt` | `datetime` | Início do evento |
| `eventEndsAt` | `datetime` | Fim do evento |
| `checkIn` | `CheckInConfigDto` | Config de check-in |
| `linkUrl` | `string` | URL (digital) |
| `dynamicQrCode` | `boolean` | QR code dinâmico |
| `statusMessage` | `string` | Mensagem de status (motivo de indisponibilidade) |
| `tokenPassBenefitAddresses` | `AddressDto[]` | Endereços (físico) |
| `tokenPassBenefitOperators` | `OperatorDto[]` | Operadores |
| `nextCheckInDates` | `NextCheckInDateDto[]` | Próximos check-ins |
| `createdAt` | `datetime` | Criação |
| `updatedAt` | `datetime` | Atualização |

### TokenPassBenefitCheckInConfigDto

Configuração de horários de check-in por dia da semana.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `mon` | `CheckInRangeEntryDto[]` | Horários de segunda-feira |
| `tue` | `CheckInRangeEntryDto[]` | Terça-feira |
| `wed` | `CheckInRangeEntryDto[]` | Quarta-feira |
| `thu` | `CheckInRangeEntryDto[]` | Quinta-feira |
| `fri` | `CheckInRangeEntryDto[]` | Sexta-feira |
| `sat` | `CheckInRangeEntryDto[]` | Sábado |
| `sun` | `CheckInRangeEntryDto[]` | Domingo |
| `all` | `CheckInRangeEntryDto[]` | Todos os dias (se definido, substitui configuração individual) |
| `special` | `CheckInSpecialRangeEntryDto[]` | Horários especiais (padrões recorrentes) |

### CheckInRangeEntryDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `start` | `string` | Horário de início (ex: `"10:30"`) |
| `end` | `string` | Horário de fim (ex: `"22:30"`) |

### CheckInSpecialRangeEntryDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `start` | `string` | Horário de início |
| `end` | `string` | Horário de fim |
| `type` | `TokenPassCheckInSpecialTypeEnum` | Tipo de recorrência |
| `data` | `object` | Dados adicionais (ex: `{ startDate, nthWeek, weekday }`) |

### TokenPassBenefitUsageRuleDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `type` | `BenefitUsageRuleTypeEnum` | `timestamp_relative` ou `day_relative` |
| `typeValue` | `object` | Configuração do limite (depende do tipo) |

### TokenPassBenefitUserUsageDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do registro de uso |
| `editionNumber` | `number` | Edição do token |
| `tokenPassBenefit` | `BenefitEntityDto` | Benefício usado |
| `tokenPassBenefitId` | `uuid` | ID do benefício |
| `uses` | `number` | Número total de usos |
| `user` | `TokenPassBenefitUserDto` | Dados do usuário |
| `createdAt` | `datetime` | Data do uso |
| `updatedAt` | `datetime` | Atualização |

### TokenPassBenefitUserDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `name` | `string` | Nome do usuário |
| `email` | `string` | Email |
| `cpf` | `string` | CPF |
| `phone` | `string` | Telefone |

### TokenPassShareCodeEntityDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do share code |
| `tenantId` | `uuid` | ID do tenant |
| `code` | `string` | Código para compartilhamento |
| `data` | `object` | Dados customizáveis (mensagem, destinatário) |
| `editionNumber` | `number` | Edição do token |
| `expiresIn` | `datetime` | Data de expiração |
| `tokenPassId` | `uuid` | ID do token pass |
| `createdAt` | `datetime` | Criação |
| `updatedAt` | `datetime` | Atualização |

### TokenPassBenefitAddressEntityDto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do endereço |
| `name` | `string` | Nome do local |
| `street` | `string` | Rua |
| `number` | `number` | Número |
| `city` | `string` | Cidade |
| `state` | `string` | Estado |
| `postalCode` | `string` | CEP |
| `rules` | `string` | Regras do local |
| `tokenPassBenefitId` | `uuid` | ID do benefício |
| `createdAt` | `datetime` | Criação |
| `updatedAt` | `datetime` | Atualização |

---

## Enums

### TokenPassBenefitTypeEnum

| Valor | Descrição |
|-------|-----------|
| `digital` | Benefício digital (link, conteúdo online) |
| `physical` | Benefício físico (local, evento presencial) |

### BenefitUseStatusEnum

| Valor | Descrição |
|-------|-----------|
| `active` | Benefício ativo e disponível para uso |
| `inactive` | Benefício inativo (fora do período, check-in indisponível) |
| `unavailable` | Benefício indisponível (limite de usos atingido) |

### ChainId

| Valor | Blockchain |
|-------|------------|
| `1` | Ethereum Mainnet |
| `137` | Polygon Mainnet (default) |
| `80002` | Polygon Amoy (testnet) |
| `11155111` | Ethereum Sepolia (testnet) |
| `1337` | Local development |

> **Nota:** Consulte o Swagger da API para a lista atualizada de chains suportadas. Chains legadas (Ropsten, Rinkeby, Kovan, Mumbai) foram descontinuadas e não devem ser usadas.

### OrderByEnum

| Valor | Descrição |
|-------|-----------|
| `ASC` | Ordem crescente |
| `DESC` | Ordem decrescente |

### BenefitUsageRuleTypeEnum

| Valor | Descrição |
|-------|-----------|
| `timestamp_relative` | Limite baseado em período relativo (ex: 1 uso por hora) |
| `day_relative` | Limite baseado em dias relativos (ex: 1 uso por dia) |

### TokenPassCheckInSpecialTypeEnum

| Valor | Descrição |
|-------|-----------|
| `all` | Todos os dias |
| `repeat_all_week` | Repete toda semana |
| `repeat_all_month` | Repete todo mês |
| `nth_week_day_of_month` | N-ésima semana + dia do mês (ex: 2ª terça-feira) |

### ExportStatusEnum

| Valor | Descrição |
|-------|-----------|
| `pending` | Exportação solicitada, aguardando processamento |
| `generating` | Relatório sendo gerado |
| `ready_for_download` | Pronto para download (link em `asset.directLink`) |
| `failed` | Geração falhou |
| `expired` | Link de download expirou |

---

## Formato do QR Code

O QR code usado para verificação e registro de uso de benefícios contém a seguinte string:

```
{editionNumber},{userId},{secret},{benefitId}
```

**Exemplo:**

```
42,d4e5f6a7-8901-2345-bcde-f67890123456,a1b2c3d4e5f6g7h8i9j0,b1c2d3e4-5678-90ab-cdef-1234567890ab
```

| Componente | Tipo | Descrição |
|------------|------|-----------|
| `editionNumber` | `number` | Número da edição do token (NFT) |
| `userId` | `uuid` | ID do usuário dono do token |
| `secret` | `string` | Secret obtido via endpoint 13 |
| `benefitId` | `uuid` | ID do benefício |

> **QR Code Dinâmico vs Estático:**
> - Se `dynamicQrCode: true`: o secret é renovado periodicamente. O usuário deve gerar um novo QR code via endpoint 13 antes de cada uso.
> - Se `dynamicQrCode: false`: o secret é fixo e o QR code pode ser reutilizado.

---

## Paginação

Todos os endpoints de listagem seguem o mesmo padrão de paginação:

```json
{
  "items": [...],
  "meta": {
    "itemCount": 10,
    "totalItems": 50,
    "itemsPerPage": 10,
    "totalPages": 5,
    "currentPage": 1
  },
  "links": {
    "first": "/resource?page=1&limit=10",
    "prev": null,
    "next": "/resource?page=2&limit=10",
    "last": "/resource?page=5&limit=10"
  }
}
```

---

## Formato Padrão de Erro

Todos os endpoints retornam erros no seguinte formato JSON:

```json
{
  "statusCode": 400,
  "message": "benefit-expired",
  "error": "Bad Request"
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `statusCode` | `number` | Código HTTP do erro |
| `message` | `string` | Código do erro (use para lógica no frontend, ex: `"usage-limit-reached"`) |
| `error` | `string` | Descrição genérica do tipo de erro HTTP |

> **Dica:** No frontend, use o campo `message` para mapear erros específicos e exibir mensagens amigáveis ao usuário. Não dependa do campo `error` para lógica de negócio.

---

## Erros Comuns

| HTTP | Situação | `message` | Ação de Recuperação |
|------|----------|-----------|---------------------|
| `401` | Token ausente ou inválido | — | Re-autenticar o usuário via Identity API |
| `403` | Role insuficiente | — | Verificar se o usuário tem o role necessário |
| `404` | Recurso não encontrado | — | Verificar IDs (tenantId, benefitId, etc.) |
| `400` | Validação de campo | — | Verificar campos obrigatórios e formatos |

Para erros específicos do endpoint **verify**, consultar a tabela completa na seção do endpoint 14.
