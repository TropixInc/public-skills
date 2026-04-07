---
id: PASS_API_REFERENCE
title: "Pass & Benefits - Referência da API"
module: offpix/pass
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - pass
  - benefits
  - token-pass
  - operators
  - check-in
  - qr-code
  - api-reference
---

# Referência da API de Pass & Benefits

Referência completa de endpoints para o módulo Pass & Benefits da W3Block. Cobre token passes (passes de acesso baseados em NFT), benefícios (digitais e físicos), operadores, endereços, códigos de compartilhamento, rastreamento de uso e resgate via QR code. Servido pelo backend dedicado do Pass.

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://pass.w3block.io` |
| Staging | *Disponível — use a variante de subdomínio de staging* |
| Swagger | https://pass.w3block.io/docs |

## Autenticação

Todos os endpoints requerem Bearer token, a menos que marcados como públicos:

```
Authorization: Bearer {accessToken}
```

Todos os endpoints são escopados a um tenant via parâmetro de caminho `{tenantId}`.

---

## Enums

### TokenPassBenefitTypeEnum

| Valor | Descrição |
|-------|-----------|
| `digital` | Benefício online/baseado em link |
| `physical` | Benefício baseado em localização que requer check-in |

### BenefitUseStatusEnum

| Valor | Cor | Descrição |
|-------|-----|-----------|
| `active` | Verde | Disponível para uso agora |
| `inactive` | Cinza | Ainda não disponível (antes do início do evento) |
| `unavailable` | Vermelho | Expirado ou limite de uso atingido |

### BenefitUsageRuleTypeEnum

| Valor | Descrição |
|-------|-----------|
| `timestamp_relative` | Redefinir contagem de uso após N milissegundos |
| `day_relative` | Redefinir diariamente, semanalmente, mensalmente ou anualmente |

### TokenPassCheckInKeys

| Valor | Descrição |
|-------|-----------|
| `sun`, `mon`, `tue`, `wed`, `thu`, `fri`, `sat` | Dia específico da semana |
| `all` | Todos os dias |
| `special` | Padrão personalizado (repetição semanal, repetição mensal, enésimo dia da semana) |

### TokenPassCheckInSpecialType

| Valor | Descrição |
|-------|-----------|
| `all` | Padrão de todos os dias |
| `repeat_all_week` | Padrão de repetição semanal |
| `repeat_all_month` | Padrão de repetição mensal |
| `nth_week_day_of_month` | Dia da semana específico do mês (ex.: 2ª terça-feira) |

---

## Entidades e Relacionamentos

```
TokenPass (1) ──→ (N) TokenPassBenefit ──→ (N) TokenPassBenefitAddress
    │                     │                        (localizações físicas)
    │                     │
    │                     ├──→ (N) TokenPassBenefitOperator
    │                     │        (usuários que podem registrar check-ins)
    │                     │
    │                     ├──→ (N) TokenPassBenefitUsage
    │                     │        (agregado: usos por edição)
    │                     │
    │                     └──→ (N) TokenPassBenefitUserUsage
    │                              (detalhado: cada uso com usuário + timestamp)
    │
    └──→ (N) TokenPassShareCode
               (códigos públicos para acessar benefícios do pass)
```

**Conceitos principais:**
- Um **TokenPass** é criado a partir de uma coleção de tokens publicada — ele representa um passe de acesso baseado em NFT
- **Benefits** são as vantagens vinculadas a um pass — digitais (links) ou físicas (localizações com check-in)
- **Operators** são usuários Admin/Operator autorizados a registrar usos de benefícios em nome dos usuários finais
- **Share codes** fornecem acesso público para visualizar benefícios do pass sem autenticação
- **Rastreamento de uso** suporta limites por edição, regras temporais (redefinições diárias/semanais/mensais) e janelas de check-in

---

## Endpoints: Token Passes

Caminho base: `/token-passes/tenants/{tenantId}`

| # | Método | Caminho | Auth | Funções | Descrição |
|---|--------|---------|------|---------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar token pass a partir de coleção |
| 2 | GET | `/` | Bearer | Admin | Listar todos os passes (paginado) |
| 3 | GET | `/{id}` | **Público** | — | Obter detalhes do pass |
| 4 | GET | `/{id}/token-editions/{editionNumber}/benefits` | Bearer | Admin, Operator, User | Obter benefícios para uma edição específica de token |
| 5 | GET | `/users/{userId}` | Bearer | Admin, Operator | Listar passes do usuário do operador |
| 6 | PATCH | `/{id}` | Bearer | Admin, Operator | Atualizar pass |
| 7 | DELETE | `/{id}` | Bearer | Admin | Excluir pass |

### POST /token-passes/tenants/{tenantId}

Criar um token pass a partir de uma coleção de tokens publicada.

**Requisição Mínima:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Access Pass",
  "name": "VIP Pass"
}
```

**Requisição Completa:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Access Pass",
  "name": "VIP Pass",
  "description": "Exclusive access to all premium events",
  "rules": "Must present QR code at entrance. Non-transferable.",
  "imageUrl": "https://cdn.example.com/pass-image.png",
  "tokenPassBenefits": [
    {
      "name": "Main Stage Access",
      "type": "physical",
      "useLimit": 1,
      "eventStartsAt": "2026-06-01T00:00:00Z",
      "eventEndsAt": "2026-06-30T23:59:59Z",
      "dynamicQrCode": true,
      "tokenPassBenefitAddresses": [
        {
          "name": "Main Venue",
          "street": "Av Paulista",
          "number": 1000,
          "city": "São Paulo",
          "state": "SP",
          "postalCode": "01310-100"
        }
      ]
    }
  ]
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `collectionId` | UUID | Sim | ID da coleção de tokens publicada (do serviço Key) |
| `tokenName` | string | Sim | Nome do token |
| `name` | string | Sim | Nome de exibição do pass |
| `description` | string | Não | Descrição do pass (máx. 1000 caracteres) |
| `rules` | string | Não | Regras/termos gerais (máx. 1000 caracteres) |
| `imageUrl` | URL | Não | Imagem de capa do pass |
| `tokenPassBenefits` | array | Não | Benefícios a serem criados com o pass |

**Observações:**
- A coleção deve estar publicada e possuir um endereço de contrato
- O `id` do pass é gerado a partir do ID da coleção
- Criar um pass atualiza o status da coleção no serviço Key

### GET /token-passes/tenants/{tenantId}/{id}/token-editions/{editionNumber}/benefits

Retorna todos os benefícios para uma edição específica de token, com status computado (`active`, `inactive`, `unavailable`) e usos restantes.

**Resposta (200):**
```json
{
  "benefits": [
    {
      "id": "benefit-uuid",
      "name": "Main Stage Access",
      "type": "physical",
      "status": "active",
      "useLimit": 3,
      "usesCount": 1,
      "eventStartsAt": "2026-06-01T00:00:00Z",
      "eventEndsAt": "2026-06-30T23:59:59Z",
      "nextCheckInDates": [
        { "start": "2026-06-15T10:00:00Z", "end": "2026-06-15T22:00:00Z" }
      ],
      "tokenPassBenefitAddresses": [
        { "name": "Main Venue", "city": "São Paulo", "state": "SP" }
      ]
    }
  ]
}
```

---

## Endpoints: Benefícios

Caminho base: `/token-pass-benefits/tenants/{tenantId}`

| # | Método | Caminho | Auth | Funções | Descrição |
|---|--------|---------|------|---------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar benefício |
| 2 | GET | `/` | Bearer | Admin, Operator | Listar benefícios (paginado) |
| 3 | GET | `/usages` | Bearer | Admin, Operator | Listar todos os registros de uso |
| 4 | GET | `/usages/xls` | Bearer | Admin, Operator | Exportar usos como XLS |
| 5 | GET | `/{id}` | **Público** | — | Obter detalhes do benefício |
| 6 | GET | `/{id}/{editionNumber}/secret` | Bearer | User | Gerar segredo QR para check-in |
| 7 | GET | `/{id}/verify` | Bearer | Todos | Verificar disponibilidade do benefício |
| 8 | PATCH | `/{id}` | Bearer | Admin | Atualizar benefício |
| 9 | DELETE | `/{id}` | Bearer | Admin | Excluir benefício |
| 10 | POST | `/{id}/register-use` | Bearer | Admin, Operator | Registrar uso (pelo operador) |
| 11 | POST | `/{id}/register-use-by-user` | Bearer | Admin, Operator | Registrar uso por userId/CPF |
| 12 | POST | `/register-use-by-qrcode` | Bearer | Admin, Operator | Registrar uso por leitura de QR code |
| 13 | POST | `/{id}/use` | Bearer | User | Usuário auto-registra uso |

### POST /token-pass-benefits/tenants/{tenantId}

**Requisição Mínima (digital):**
```json
{
  "name": "Exclusive Content",
  "tokenPassId": "pass-uuid",
  "type": "digital",
  "eventStartsAt": "2026-06-01T00:00:00Z"
}
```

**Requisição Completa (físico com check-in):**
```json
{
  "name": "VIP Lounge Access",
  "description": "Access to the VIP lounge during events",
  "rules": "Maximum 2 guests per visit",
  "tokenPassId": "pass-uuid",
  "type": "physical",
  "useLimit": 10,
  "eventStartsAt": "2026-06-01T00:00:00Z",
  "eventEndsAt": "2026-12-31T23:59:59Z",
  "dynamicQrCode": true,
  "allowSelfUse": false,
  "timezoneOrUtcOffset": "America/Sao_Paulo",
  "checkIn": {
    "mon": [{ "start": "10:00", "end": "22:00" }],
    "tue": [{ "start": "10:00", "end": "22:00" }],
    "wed": [{ "start": "10:00", "end": "22:00" }],
    "thu": [{ "start": "10:00", "end": "22:00" }],
    "fri": [{ "start": "10:00", "end": "23:00" }],
    "sat": [{ "start": "12:00", "end": "23:00" }]
  },
  "usageRule": {
    "type": "day_relative",
    "typeValue": "week"
  },
  "requirements": {
    "tier": "vip",
    "minLevel": { "validator": "gt", "value": "5" }
  },
  "requiredWhitelistIds": ["whitelist-uuid"],
  "tokenPassBenefitAddresses": [
    {
      "name": "VIP Lounge Downtown",
      "street": "Av Paulista",
      "number": 1000,
      "city": "São Paulo",
      "state": "SP",
      "postalCode": "01310-100",
      "rules": "Enter through gate B"
    }
  ]
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `name` | string | Sim | — | Nome do benefício (máx. 70 caracteres) |
| `description` | string | Não | — | Descrição (máx. 1000 caracteres) |
| `rules` | string | Não | — | Regras/termos (máx. 1000 caracteres) |
| `tokenPassId` | UUID | Sim | — | Token pass pai |
| `type` | `digital` \| `physical` | Não | `digital` | Tipo do benefício |
| `useLimit` | integer | Não | — | Máximo de usos por edição (null = ilimitado) |
| `eventStartsAt` | ISO datetime | Não | — | Quando o benefício fica disponível |
| `eventEndsAt` | ISO datetime | Não | — | Quando o benefício expira |
| `dynamicQrCode` | boolean | Não | `false` | Se verdadeiro, QR codes expiram após 45 segundos |
| `allowSelfUse` | boolean | Não | `false` | Se o usuário pode resgatar o próprio benefício |
| `checkIn` | object | Não | — | Janelas de check-in por dia (veja esquema abaixo) |
| `usageRule` | object | Não | — | Limites temporais de uso |
| `requirements` | object | Não | — | Requisitos de metadados do token para elegibilidade |
| `requiredWhitelistIds` | UUID[] | Não | — | Requisitos de whitelist |
| `timezoneOrUtcOffset` | string | Não | `America/Sao_Paulo` | Fuso horário para horários de check-in |
| `linkUrl` | string | Não | — | URL para benefícios digitais |
| `linkRules` | string | Não | — | Regras para acesso ao link |
| `tokenPassBenefitAddresses` | array | Não | — | Localizações físicas |

### Esquema de Configuração de Check-In

```json
{
  "mon": [{ "start": "10:00", "end": "22:00" }],
  "fri": [
    { "start": "10:00", "end": "14:00" },
    { "start": "18:00", "end": "23:00" }
  ],
  "special": [
    {
      "type": "nth_week_day_of_month",
      "start": "19:00",
      "end": "23:00",
      "data": { "nthWeek": 2, "weekday": 5 }
    }
  ]
}
```

Cada dia suporta múltiplas janelas de horário. A chave `special` suporta padrões recorrentes.

### Esquema de Regra de Uso

```json
{
  "type": "day_relative",
  "typeValue": "day"
}
```

| type | typeValue | Descrição |
|------|-----------|-----------|
| `day_relative` | `day` | Redefinir diariamente |
| `day_relative` | `week` | Redefinir semanalmente |
| `day_relative` | `month` | Redefinir mensalmente |
| `day_relative` | `year` | Redefinir anualmente |
| `timestamp_relative` | number (ms) | Redefinir após N milissegundos |

### Esquema de Requisitos

Igualdade simples:
```json
{ "tier": "vip", "color": "gold" }
```

Validadores avançados:
```json
{
  "level": { "validator": "gt", "value": "5" },
  "expiryTimestamp": { "validator": "timestamp_now_relative_secs_gt", "value": 0 }
}
```

Validadores: `gt`, `lt`, `timestamp_now_relative_secs_gt`, `timestamp_now_relative_secs_lt`

### POST /{id}/register-use — Operador Registra Uso

**Requisição:**
```json
{
  "userId": "user-uuid",
  "editionNumber": 42,
  "secret": "sha256-hash-string"
}
```

### POST /{id}/register-use-by-user — Registrar por ID de Usuário ou CPF

```json
{
  "userId": "user-uuid"
}
```
ou
```json
{
  "cpf": "12345678901"
}
```

### POST /register-use-by-qrcode — Registrar por Leitura de QR Code

```json
{
  "qrcode": "42,user-uuid,secret-hash,benefit-uuid"
}
```

Formato do QR code: `{editionNumber},{userId},{secret},{benefitId}`

### GET /{id}/{editionNumber}/secret — Gerar Segredo QR

Retorna um hash para geração de QR code. Para `dynamicQrCode: true`, o hash expira após 45 segundos.

**Resposta:**
```json
{
  "secret": "sha256-hash-string"
}
```

**Cálculo do segredo:**
- Estático: `SHA256(userId + benefitId + editionNumber + secretKey)`
- Dinâmico: `SHA256(userId + benefitId + editionNumber + secretKey + expiration)`

---

## Endpoints: Endereços de Benefícios

Caminho base: `/token-pass-benefit-addresses/tenants/{tenantId}`

| # | Método | Caminho | Auth | Funções | Descrição |
|---|--------|---------|------|---------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar endereço |
| 2 | GET | `/` | Bearer | Admin | Listar endereços (paginado) |
| 3 | GET | `/{id}` | Bearer | Admin | Obter endereço |
| 4 | PATCH | `/{id}` | Bearer | Admin | Atualizar endereço |
| 5 | DELETE | `/{id}` | Bearer | Admin | Excluir endereço |

### Campos de Endereço

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome da localização (máx. 80 caracteres) |
| `street` | string | Não | Endereço da rua |
| `number` | integer | Não | Número do prédio |
| `city` | string | Sim | Cidade |
| `state` | string | Sim | Código do estado/província |
| `postalCode` | string | Não | CEP |
| `rules` | string | Não | Regras específicas da localização |
| `tokenPassBenefitId` | UUID | Sim | Benefício pai |

---

## Endpoints: Operadores de Benefícios

Caminho base: `/token-pass-benefit-operators/tenants/{tenantId}`

| # | Método | Caminho | Auth | Funções | Descrição |
|---|--------|---------|------|---------|-----------|
| 1 | POST | `/` | Bearer | Admin | Atribuir operador ao benefício |
| 2 | GET | `/` | Bearer | Admin | Listar todos os operadores (paginado) |
| 3 | GET | `/{id}` | Bearer | Admin, Operator | Obter detalhes do operador |
| 4 | GET | `/benefits/{benefitId}` | Bearer | Admin, Operator | Listar operadores do benefício |
| 5 | DELETE | `/{id}` | Bearer | Admin | Remover operador |

### POST — Atribuir Operador

```json
{
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "benefit-uuid"
}
```

O usuário deve ter a função Admin ou Operator. Cada usuário só pode ser atribuído uma vez por benefício.

**A resposta inclui dados enriquecidos:** nome do operador, e-mail, endereço da carteira e token passes associados.

---

## Endpoints: Códigos de Compartilhamento

Caminho base: `/token-pass-share-codes/tenants/{tenantId}`

| # | Método | Caminho | Auth | Funções | Descrição |
|---|--------|---------|------|---------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar código de compartilhamento |
| 2 | GET | `/{code}` | **Público** | — | Obter benefícios pelo código de compartilhamento |

### POST — Criar Código de Compartilhamento

```json
{
  "contractAddress": "0xContractAddress...",
  "chainId": 137,
  "editionNumber": 42,
  "expiresIn": "2026-12-31T23:59:59Z",
  "data": { "campaign": "summer-promo" }
}
```

### GET /{code} — Recuperar Benefícios

Retorna todos os benefícios do pass associado ao código de compartilhamento, incluindo metadados do token e status computado do benefício.

---

## Códigos de Erro

| Código | HTTP | Causa |
|--------|------|-------|
| `invalid-secret-hash` | 400 | O segredo QR não corresponde ao hash calculado |
| `secret-expired` | 400 | QR code dinâmico expirado (> 45 segundos) |
| `undefined-benefit-start-date` | 400 | Benefício sem `eventStartsAt` definido |
| `benefit-not-started` | 400 | Horário atual anterior a `eventStartsAt` |
| `benefit-expired` | 400 | Horário atual posterior a `eventEndsAt` |
| `temporary-usage-limit-reached` | 400 | Limite de uso do período atingido |
| `usage-limit-reached` | 400 | Limite permanente `useLimit` atingido |
| `wrong-checkin-time` | 400 | Fora da janela de check-in |
| `undefined-checkin-times` | 400 | Horários de check-in não configurados |
| `invalid-token-owner` | 403 | Usuário não possui a edição do token |
| `self-use-not-allowed` | 403 | `allowSelfUse: false` e o usuário tentou resgatar o próprio benefício |
| `unsatisfied-benefit-requirements` | 403 | Token não atende aos requisitos de metadados/whitelist |
| `unauthorized-operator` | 403 | Operador não atribuído a este benefício |
| `benefit-not-found` | 404 | ID de benefício inválido |
| `token-not-found` | 404 | Edição do token não encontrada |
