---
id: FLOW_PASS_LIFECYCLE
title: "Pass e Benefícios - Ciclo de Vida do Pass"
module: offpix/pass
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - pass
  - token-pass
  - benefits
  - create
  - operators
depends_on:
  - PASS_API_REFERENCE
  - FLOW_CONTRACTS_COLLECTION_MANAGEMENT
---

# Ciclo de Vida do Pass (Criar, Configurar, Gerenciar)

## Visão Geral

Um Token Pass é um passe de acesso baseado em NFT criado a partir de uma coleção de tokens publicada. Este fluxo cobre a jornada do admin: criar um pass, adicionar benefícios (digitais e físicos), configurar horários de check-in, atribuir operadores e gerenciar códigos de compartilhamento. No frontend, isso é acessado em **Tokens > Pass** (`/dash/tokens/pass`).

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Coleção publicada | Coleção de tokens com `status: published` | [Gerenciamento de Coleção](../contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |

---

## Fluxo: Criar um Token Pass

### Passo 1: Criar Pass a partir de uma Coleção

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/token-passes/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Pass",
  "name": "VIP Access"
}
```

**Requisição Completa:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Pass",
  "name": "VIP Access",
  "description": "Exclusive access to premium events and lounges",
  "rules": "Must present QR code at entrance. Valid for token holder only.",
  "imageUrl": "https://cdn.example.com/vip-pass.png"
}
```

**Resposta (201):**
```json
{
  "id": "pass-uuid",
  "tokenName": "VIP Pass",
  "name": "VIP Access",
  "contractAddress": "0xContract...",
  "chainId": 137,
  "description": "Exclusive access to premium events and lounges",
  "rules": "Must present QR code at entrance.",
  "imageUrl": "https://cdn.example.com/vip-pass.png",
  "tenantId": "tenant-uuid",
  "createdAt": "2026-04-01T10:00:00Z"
}
```

**Observações:**
- A coleção deve estar publicada com um endereço de contrato válido
- O ID do pass é derivado do ID da coleção
- Após a criação, o frontend solicita a adição de benefícios (Passo 2)

### Passo 2: Adicionar um Benefício Digital

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/token-pass-benefits/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "name": "Exclusive Video Content",
  "description": "Access to behind-the-scenes footage",
  "tokenPassId": "pass-uuid",
  "type": "digital",
  "eventStartsAt": "2026-06-01T00:00:00Z",
  "linkUrl": "https://example.com/exclusive-content",
  "linkRules": "Link valid for 24 hours after first access"
}
```

### Passo 3: Adicionar um Benefício Físico (com Check-In)

**Requisição:**
```json
{
  "name": "VIP Lounge Access",
  "description": "Access to the VIP lounge",
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
  "tokenPassBenefitAddresses": [
    {
      "name": "Downtown Lounge",
      "street": "Av Paulista",
      "number": 1000,
      "city": "São Paulo",
      "state": "SP",
      "postalCode": "01310-100",
      "rules": "Enter through VIP entrance on the 2nd floor"
    }
  ]
}
```

### Passo 4: Atribuir Operadores

Operadores são usuários (com role Admin ou Operator) que podem registrar usos de benefícios em nome dos detentores de tokens.

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/token-pass-benefit-operators/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "benefit-uuid"
}
```

**Resposta (201):**
```json
{
  "id": "operator-uuid",
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "benefit-uuid",
  "name": "John Operator",
  "email": "john@example.com",
  "walletAddress": "0xOperator...",
  "associatedTokens": [
    { "id": "pass-uuid", "imageUrl": "...", "tokenName": "VIP Pass" }
  ]
}
```

### Passo 5 (Opcional): Criar Código de Compartilhamento

Códigos de compartilhamento permitem acesso público para visualizar benefícios do pass sem autenticação.

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/token-pass-share-codes/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "contractAddress": "0xContract...",
  "chainId": 137,
  "editionNumber": 1,
  "expiresIn": "2026-12-31T23:59:59Z",
  "data": { "campaign": "launch-promo" }
}
```

**Resposta:** Código de compartilhamento com string `code` gerada automaticamente (maiúsculas).

---

## Padrões de Configuração de Check-In

### Agenda Diária (Mais Comum)

```json
{
  "checkIn": {
    "mon": [{ "start": "09:00", "end": "18:00" }],
    "tue": [{ "start": "09:00", "end": "18:00" }],
    "wed": [{ "start": "09:00", "end": "18:00" }],
    "thu": [{ "start": "09:00", "end": "18:00" }],
    "fri": [{ "start": "09:00", "end": "20:00" }]
  }
}
```

### Todos os Dias com a Mesma Agenda

```json
{
  "checkIn": {
    "all": [{ "start": "10:00", "end": "22:00" }]
  }
}
```

### Múltiplas Janelas por Dia

```json
{
  "checkIn": {
    "sat": [
      { "start": "10:00", "end": "14:00" },
      { "start": "18:00", "end": "23:00" }
    ]
  }
}
```

### Padrão Especial: 2a Sexta-feira de Cada Mês

```json
{
  "checkIn": {
    "special": [
      {
        "type": "nth_week_day_of_month",
        "start": "19:00",
        "end": "23:00",
        "data": { "nthWeek": 2, "weekday": 5 }
      }
    ]
  }
}
```

---

## Regras de Uso

### Sem Regra (Padrão)
Total de usos contados por edição contra `useLimit`. Sem reinício.

### Reinício Diário
```json
{ "type": "day_relative", "typeValue": "day" }
```
O usuário pode usar o benefício `useLimit` vezes por dia.

### Reinício Semanal
```json
{ "type": "day_relative", "typeValue": "week" }
```

### Reinício Baseado em Timestamp
```json
{ "type": "timestamp_relative", "typeValue": 3600000 }
```
Reinicia após 1 hora (3600000 ms).

---

## Requisitos de Metadados do Token

Benefícios podem exigir metadados específicos do token para serem elegíveis:

### Igualdade Simples
```json
{ "tier": "vip" }
```
O token deve ter `tokenData.tier === "vip"` (case-insensitive).

### Comparação Numérica
```json
{ "level": { "validator": "gt", "value": "5" } }
```
O token deve ter `tokenData.level > 5`.

### Baseado em Tempo
```json
{ "expiryTimestamp": { "validator": "timestamp_now_relative_secs_gt", "value": 0 } }
```
O `expiryTimestamp` do token deve estar no futuro.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Coleção não publicada | Coleção não possui endereço de contrato | Publique a coleção primeiro |
| 400 | Operador deve ser Admin ou Operator | Usuário não possui a role correta | Atribua a role apropriada primeiro |
| 409 | Operador já atribuído | Atribuição duplicada de operador | Ignore -- o operador já tem acesso |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Não consegue criar pass | A coleção deve estar publicada com um endereço de contrato válido |
| 2 | Horários de check-in não funcionam | Certifique-se de que `timezoneOrUtcOffset` corresponde ao fuso horário da localização |
| 3 | Uso não reinicia | Configure `usageRule` -- sem ela, os usos contam permanentemente contra `useLimit` |
| 4 | Benefícios não aparecem para o usuário | Verifique se o usuário possui a edição do token e atende aos requisitos de metadados |
| 5 | QR dinâmico expirou | QR codes dinâmicos são válidos por 45 segundos. Regenere antes de escanear |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Resgate de Benefícios | Lado do usuário/operador dos benefícios | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Gerenciamento de Coleção | Crie a coleção primeiro | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](../contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Whitelist | Requisitos de whitelist para benefícios | [FLOW_WHITELIST_MANAGEMENT](../whitelist/FLOW_WHITELIST_MANAGEMENT.md) |
