# Gerenciamento de Passes e Benefícios (Admin)

## Overview

Fluxo administrativo para criação, atualização e exclusão de token passes, benefícios, endereços de benefícios, configuração de check-in e atribuição de operadores. Apenas usuários com roles administrativos podem executar essas operações. O fluxo completo de setup envolve criar o pass, adicionar benefícios (digitais ou físicos), configurar horários de check-in, definir regras de uso e atribuir operadores que poderão verificar e registrar o uso dos benefícios.

## Prerequisites

- **Auth**: Bearer token com role `superAdmin`, `admin`, `integration` ou `application`
- **tenantId**: UUID do tenant (organização)
- **collectionId**: UUID da collection de NFTs (necessário para criar passes — representa o contrato na blockchain)

---

## Steps

### Step 1: Criar Token Pass

- **Screen**: Formulário admin com campos: collectionId, tokenName, name, description, rules, imageUrl.
- **User Action**: Preenche os campos do pass e clica em "Criar".

- **API Call**:

```
POST /token-passes/tenants/{tenantId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

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
| `tokenName` | `string` | Sim | Nome do token associado na blockchain |
| `name` | `string` | Sim | Nome do pass |
| `description` | `string` | Não | Descrição do pass |
| `rules` | `string` | Não | Regras gerais do pass |
| `imageUrl` | `string` | Não | URL da imagem do pass |
| `tokenPassBenefits` | `CreateTokenPassBenefitDto[]` | Não | Benefícios criados inline com o pass |

- **Response** (201 Created):

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

- **State Changes**: Pass criado com `id`, `contractAddress` e `chainId` retornados. Se `tokenPassBenefits[]` foi enviado inline, os benefícios também são criados junto com o pass.

> **Nota:** Incluir `tokenPassBenefits[]` no body permite criar benefícios junto com o pass em uma única chamada. Caso prefira criar benefícios separadamente, omita esse campo e use o Step 2.

---

### Step 2: Criar Benefício

- **Screen**: Formulário de benefício com campos: name, type, useLimit, eventStartsAt/EndsAt, checkIn, linkUrl, dynamicQrCode, allowSelfUse.
- **User Action**: Preenche os campos e clica em "Criar Benefício".

- **API Call**:

```
POST /token-pass-benefits/tenants/{tenantId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

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
| `eventStartsAt` | `datetime` | Não | Início do evento |
| `eventEndsAt` | `datetime` | Não | Fim do evento |
| `checkIn` | `TokenPassBenefitCheckInConfigDto` | Não | Configuração de horários de check-in por dia |
| `linkUrl` | `string` | Não | URL do benefício digital |
| `linkRules` | `string` | Não | Regras do link |
| `dynamicQrCode` | `boolean` | Não | Se `true`, QR code usa secret dinâmico |
| `tokenPassId` | `uuid` | Sim | ID do token pass associado |
| `tokenPassBenefitAddresses` | `CreateAddressDto[]` | Não | Endereços para benefícios físicos |
| `allowSelfUse` | `boolean` | Não | Se `true`, usuário pode registrar uso próprio (default: `false`) |
| `usageRule` | `TokenPassBenefitUsageRuleDto` | Não | Regra de uso temporário |
| `timezoneOrUtcOffset` | `string` | Não | Timezone (default: `America/Sao_Paulo`) |

- **Response** (201 Created): `TokenPassBenefitEntityDto` com `id`, `name`, `type`, `checkIn`, `nextCheckInDates`, `tokenPass`, etc.

> **Nota:** Para benefícios físicos (`type: "physical"`), é recomendado adicionar endereços — via `tokenPassBenefitAddresses` inline ou separadamente no Step 3.
>
> **Referência:** Para o schema completo de request/response e todos os campos disponíveis, consulte [PASS_API_REFERENCE.md — Endpoint 8](./PASS_API_REFERENCE.md).

---

### Step 3: Adicionar Endereço (benefícios físicos)

- **Screen**: Formulário de endereço vinculado ao benefício.
- **User Action**: Preenche dados do local e clica em "Adicionar Endereço".

> Apenas para benefícios do tipo `physical`. Benefícios digitais não possuem endereços.

- **API Call**:

```
POST /token-pass-benefit-addresses/tenants/{tenantId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

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

- **Response** (201 Created): `TokenPassBenefitAddressEntityDto` com `id`, `name`, `street`, `number`, `city`, `state`, `postalCode`, `rules`, `tokenPassBenefitId`.

---

### Step 4: Configurar Check-in (horários)

A configuração de check-in define os horários em que o benefício pode ser utilizado. O campo `checkIn` no benefício aceita uma estrutura flexível por dia da semana, com suporte a horários especiais recorrentes.

#### Estrutura do CheckIn Config

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `mon` | `CheckInRangeEntryDto[]` | Horários de segunda-feira |
| `tue` | `CheckInRangeEntryDto[]` | Terça-feira |
| `wed` | `CheckInRangeEntryDto[]` | Quarta-feira |
| `thu` | `CheckInRangeEntryDto[]` | Quinta-feira |
| `fri` | `CheckInRangeEntryDto[]` | Sexta-feira |
| `sat` | `CheckInRangeEntryDto[]` | Sábado |
| `sun` | `CheckInRangeEntryDto[]` | Domingo |
| `all` | `CheckInRangeEntryDto[]` | Todos os dias (se definido, **substitui** configuração individual) |
| `special` | `CheckInSpecialRangeEntryDto[]` | Horários especiais com padrões recorrentes |

Cada `CheckInRangeEntryDto` possui:

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `start` | `string` | Horário de início (ex: `"10:30"`) |
| `end` | `string` | Horário de fim (ex: `"22:30"`) |

#### Tipos Especiais (`special`)

| Tipo | Descrição |
|------|-----------|
| `repeat_all_week` | Repete toda semana |
| `repeat_all_month` | Repete todo mês |
| `nth_week_day_of_month` | N-ésima semana + dia do mês (ex: 2a terça-feira de cada mês) |

#### Timezone

O campo `timezoneOrUtcOffset` define o fuso horário para interpretar os horários de check-in. O valor padrão é `America/Sao_Paulo`.

#### Exemplo completo de configuração de check-in

```json
{
  "checkIn": {
    "mon": [
      { "start": "10:00", "end": "14:00" },
      { "start": "18:00", "end": "22:00" }
    ],
    "tue": [
      { "start": "10:00", "end": "22:00" }
    ],
    "wed": [
      { "start": "10:00", "end": "22:00" }
    ],
    "thu": [
      { "start": "10:00", "end": "22:00" }
    ],
    "fri": [
      { "start": "10:00", "end": "23:00" }
    ],
    "sat": [
      { "start": "12:00", "end": "23:00" }
    ],
    "sun": [
      { "start": "12:00", "end": "20:00" }
    ],
    "special": null
  }
}
```

**Exemplo usando `all` (mesmo horário todos os dias):**

```json
{
  "checkIn": {
    "all": [
      { "start": "10:30", "end": "22:30" }
    ],
    "special": null
  }
}
```

> **Nota:** Quando `all` está definido, ele substitui qualquer configuração individual por dia (`mon`, `tue`, etc.).

**Exemplo com horário especial recorrente:**

```json
{
  "checkIn": {
    "special": [
      {
        "start": "19:00",
        "end": "23:00",
        "type": "nth_week_day_of_month",
        "data": {
          "nthWeek": 2,
          "weekday": "tue"
        }
      }
    ]
  }
}
```

- **API Call** (atualizar benefício com checkIn):

```
PATCH /token-pass-benefits/tenants/{tenantId}/{benefitId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "checkIn": {
    "all": [
      { "start": "10:30", "end": "22:30" }
    ],
    "special": null
  },
  "timezoneOrUtcOffset": "America/Sao_Paulo"
}
```

- **Response** (200 OK): `TokenPassBenefitEntityDto` atualizado com `checkIn` e `nextCheckInDates` recalculados.

---

### Step 5: Configurar Regra de Uso (usageRule)

A regra de uso (`usageRule`) define limites temporários de uso do benefício, complementando o `useLimit` global. Existem dois tipos:

#### Tipo `timestamp_relative`

Limita o número de usos dentro de um período relativo a partir do último uso.

```json
{
  "usageRule": {
    "type": "timestamp_relative",
    "typeValue": {
      "uses": 1,
      "period": 3600
    }
  }
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `uses` | `number` | Número máximo de usos no período |
| `period` | `number` | Período em segundos (3600 = 1 hora) |

> **Exemplo:** Com `uses: 1` e `period: 3600`, o usuário pode usar o benefício no máximo 1 vez por hora.

#### Tipo `day_relative`

Limita o número de usos por dia (relativo ao fuso horário configurado).

```json
{
  "usageRule": {
    "type": "day_relative",
    "typeValue": {
      "uses": 2,
      "period": 1
    }
  }
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `uses` | `number` | Número máximo de usos no período |
| `period` | `number` | Período em dias (1 = 1 dia) |

> **Exemplo:** Com `uses: 2` e `period: 1`, o usuário pode usar o benefício no máximo 2 vezes por dia.

- **API Call** (atualizar benefício com usageRule):

```
PATCH /token-pass-benefits/tenants/{tenantId}/{benefitId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "usageRule": {
    "type": "day_relative",
    "typeValue": {
      "uses": 2,
      "period": 1
    }
  }
}
```

- **Response** (200 OK): `TokenPassBenefitEntityDto` atualizado com `usageRule`.

---

### Step 6: Atribuir Operadores

- **Screen**: Formulário de atribuição de operador a um benefício.
- **User Action**: Seleciona o usuário e o benefício, clica em "Atribuir".

- **API Call**:

```
POST /token-pass-benefit-operators/tenants/{tenantId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

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

- **Response** (201 Created): `TokenPassBenefitOperatorEntityDto` com `id`, `userId`, `tokenPassBenefitId`.

> **Nota:** Após a atribuição, o operador poderá visualizar o pass associado ao benefício (via `GET /token-passes/tenants/{tenantId}/users/{userId}`) e verificar/registrar usos nos endpoints de register-use.

---

### Step 7: Atualizar Pass/Benefício

Para atualizar um pass ou benefício existente, use os endpoints PATCH. Apenas os campos enviados serão atualizados — não é necessário enviar o body completo.

#### Atualizar Token Pass

```
PATCH /token-passes/tenants/{tenantId}/{id}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body (exemplo):**

```json
{
  "name": "Pass VIP Atualizado",
  "description": "Nova descrição do pass",
  "rules": "Novas regras atualizadas",
  "imageUrl": "https://cdn.example.com/pass-vip-v2.png"
}
```

#### Atualizar Benefício

```
PATCH /token-pass-benefits/tenants/{tenantId}/{id}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body (exemplo):**

```json
{
  "name": "Acesso VIP Atualizado",
  "useLimit": 5,
  "eventEndsAt": "2024-06-16T23:59:00-03:00"
}
```

- **Response** (200 OK): Entidade atualizada (`TokenPassEntityDto` ou `TokenPassBenefitEntityDto`).

> **Nota:** Campos como `collectionId` e `tokenPassId` não podem ser alterados após a criação.

---

### Step 8: Remover Recursos

Todos os endpoints de exclusão retornam `204 No Content` sem body.

| Recurso | Método | Endpoint |
|---------|--------|----------|
| Token Pass | `DELETE` | `/token-passes/tenants/{tenantId}/{id}` |
| Benefício | `DELETE` | `/token-pass-benefits/tenants/{tenantId}/{id}` |
| Endereço | `DELETE` | `/token-pass-benefit-addresses/tenants/{tenantId}/{id}` |
| Operador | `DELETE` | `/token-pass-benefit-operators/tenants/{tenantId}/{id}` |

- **Auth**: Bearer token (roles: superAdmin, integration, application, admin)
- **Response**: `204 No Content`

> **Nota:** Remover um token pass pode impactar benefícios associados. Remover um benefício remove também suas associações com operadores e endereços.

---

## API Sequence (setup completo)

```
1. POST /token-passes/tenants/{tenantId}                    — Criar pass
2. POST /token-pass-benefits/tenants/{tenantId}              — Criar benefício
3. POST /token-pass-benefit-addresses/tenants/{tenantId}     — Adicionar endereço (se físico)
4. PATCH /token-pass-benefits/tenants/{tenantId}/{id}        — Configurar checkIn e usageRule
5. POST /token-pass-benefit-operators/tenants/{tenantId}     — Atribuir operador
```

> **Nota:** Os steps 2-5 podem ser repetidos para criar múltiplos benefícios, endereços e operadores. O step 2 pode incluir `checkIn`, `usageRule` e `tokenPassBenefitAddresses` inline, eliminando a necessidade dos steps 3 e 4.

---

## Error Recovery

| Situação | HTTP | Código | Ação de Recuperação |
|----------|------|--------|---------------------|
| Token ausente ou inválido | `401` | — | Re-autenticar o usuário via Identity API |
| Role insuficiente para a operação | `403` | — | Verificar se o usuário tem role `superAdmin`, `admin`, `integration` ou `application` |
| Pass/Benefício não encontrado | `404` | — | Verificar se os IDs (`tenantId`, `id`) estão corretos |
| Campo obrigatório ausente | `400` | — | Verificar campos obrigatórios (`name`, `collectionId`, `tokenPassId`, etc.) |
| Collection não encontrada | `400` | — | Verificar se o `collectionId` é válido e pertence ao tenant |
| Formato de data inválido | `400` | — | Usar formato ISO 8601 com timezone (ex: `2024-06-15T18:00:00-03:00`) |
| Horário de checkIn inválido | `400` | — | Verificar formato `HH:mm` (ex: `"10:30"`) e que `start` < `end` |
| Operador já atribuído ao benefício | `400` | — | Verificar se o operador já existe antes de criar duplicata |
| Secret do QR code inválido | `400` | `invalid-secret-hash` | Solicitar novo QR code ao usuário |
| Secret expirado (QR dinâmico) | `400` | `secret-expired` | Usuário deve gerar novo QR code |
| Fora do horário de check-in | `400` | `wrong-checkin-time` | Aguardar horário de check-in configurado |
| Limite de uso atingido | `400` | `usage-limit-reached` | Não é possível usar — todos os usos consumidos |
| Limite temporário atingido (usageRule) | `400` | `temporary-usage-limit-reached` | Aguardar próximo período permitido |

---

## React SDK

O componente `ConfigPanel` no React SDK gerencia a interface de configuração de horários de check-in, permitindo ao admin:

- Selecionar dias da semana individualmente ou usar "todos os dias" (`all`)
- Adicionar múltiplas faixas de horário por dia
- Configurar horários especiais recorrentes (`special`)
- Definir o timezone via `timezoneOrUtcOffset`
- Visualizar os próximos horários calculados (`nextCheckInDates`)

O componente utiliza o endpoint `PATCH /token-pass-benefits/tenants/{tenantId}/{id}` para salvar as alterações de check-in.
