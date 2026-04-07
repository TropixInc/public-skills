---
id: FLOW_PASS_BENEFIT_REDEMPTION
title: "Pass & Benefícios - Resgate de Benefícios"
module: offpix/pass
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - pass
  - benefits
  - redemption
  - qr-code
  - check-in
  - usage
depends_on:
  - PASS_API_REFERENCE
  - FLOW_PASS_LIFECYCLE
---

# Resgate de Benefícios

## Visão Geral

Este fluxo cobre os lados do usuário e do operador no resgate de benefícios: visualizar benefícios disponíveis, gerar QR codes, verificar elegibilidade e registrar usos. Existem quatro métodos de resgate: leitura pelo operador (QR), busca pelo operador (userId/CPF), leitura de QR code e autouso pelo usuário.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT (Usuário para autouso, Admin/Operador para registrar usos) | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Posse do token | O usuário deve possuir uma edição de token da coleção de passes | Compra ou mint de token |
| Benefício configurado | O benefício deve estar ativo com datas válidas | [Ciclo de Vida do Pass](./FLOW_PASS_LIFECYCLE.md) |

## Cadeia de Validação

Antes que um benefício possa ser utilizado, estas verificações são realizadas em ordem:

```
1. Token ownership → Does user own the edition?
2. Benefit dates → Is current time between eventStartsAt and eventEndsAt?
3. Check-in window → Is current time within a configured check-in slot?
4. Requirements → Does token metadata meet all requirements?
5. Whitelist → Is user in required whitelists?
6. Usage limit → Has the edition's useLimit been reached?
7. Usage rule → Has the temporal limit (daily/weekly/monthly) been reached?
8. Secret validation → Does the QR secret match?
9. Self-use check → If allowSelfUse=false, is the registering user different from the owner?
```

---

## Fluxo A: Visualizar Benefícios Disponíveis

### Passo 1: Obter Benefícios para Edição de Token

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/token-passes/tenants/{tenantId}/{passId}/token-editions/{editionNumber}/benefits` | Bearer (User, Admin, Operator) |

**Resposta (200):**
```json
{
  "benefits": [
    {
      "id": "benefit-uuid-1",
      "name": "VIP Lounge Access",
      "type": "physical",
      "status": "active",
      "useLimit": 10,
      "usesCount": 3,
      "eventStartsAt": "2026-06-01T00:00:00Z",
      "eventEndsAt": "2026-12-31T23:59:59Z",
      "dynamicQrCode": true,
      "nextCheckInDates": [
        { "start": "2026-04-01T13:00:00Z", "end": "2026-04-01T22:00:00Z" },
        { "start": "2026-04-02T13:00:00Z", "end": "2026-04-02T22:00:00Z" }
      ],
      "tokenPassBenefitAddresses": [
        { "name": "Downtown Lounge", "city": "São Paulo", "state": "SP" }
      ]
    },
    {
      "id": "benefit-uuid-2",
      "name": "Exclusive Content",
      "type": "digital",
      "status": "active",
      "linkUrl": "https://example.com/exclusive"
    }
  ]
}
```

**Cálculo de status:**
- `active` -- todas as verificações passam, o benefício pode ser usado agora
- `inactive` -- o horário atual é anterior a `eventStartsAt`
- `unavailable` -- expirado ou limite de uso atingido

---

## Fluxo B: Operador Registra Uso (QR Code)

O fluxo padrão de check-in físico.

### Passo 1: Usuário Gera o Segredo do QR

**Endpoint (dispositivo do Usuário):**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/token-pass-benefits/tenants/{tenantId}/{benefitId}/{editionNumber}/secret` | Bearer (User) |

**Resposta:**
```json
{
  "secret": "a1b2c3d4e5f6..."
}
```

Para `dynamicQrCode: true`, o segredo inclui um timestamp e expira após **45 segundos**.

O aplicativo do usuário gera um QR code contendo: `{editionNumber},{userId},{secret},{benefitId}`

### Passo 2: Operador Lê o QR Code

**Endpoint (dispositivo do Operador):**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/token-pass-benefits/tenants/{tenantId}/register-use-by-qrcode` | Bearer (Operator) | application/json |

**Requisição:**
```json
{
  "qrcode": "42,user-uuid,a1b2c3d4e5f6...,benefit-uuid"
}
```

**Resposta:** 201 Created (uso registrado)

**O que acontece internamente:**
1. QR code é parseado em componentes
2. Segredo é validado (comparação de hash SHA256)
3. Para QR dinâmico: expiração verificada (janela de 45 segundos)
4. Todas as verificações da cadeia de validação são aplicadas
5. Registro de uso criado (`TokenPassBenefitUserUsageEntity`)
6. Contagem agregada atualizada (`TokenPassBenefitUsageEntity`)

---

## Fluxo C: Operador Registra Uso (Por Busca de Usuário)

Para situações em que a leitura de QR não é prática.

### Por ID do Usuário

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use-by-user
```

```json
{
  "userId": "user-uuid"
}
```

### Por CPF

```json
{
  "cpf": "12345678901"
}
```

O sistema busca o usuário pelo CPF, depois verifica a posse do token e registra o uso.

---

## Fluxo D: Operador Registra Uso (Manual com Segredo)

O método mais explícito -- o operador fornece todos os dados de verificação.

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use
```

```json
{
  "userId": "user-uuid",
  "editionNumber": 42,
  "secret": "a1b2c3d4e5f6..."
}
```

---

## Fluxo E: Autouso pelo Usuário

Disponível apenas quando `allowSelfUse: true` no benefício.

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/use
```

```json
{
  "userId": "user-uuid",
  "editionNumber": 42
}
```

Normalmente usado para benefícios digitais onde nenhum operador é necessário.

---

## Fluxo F: Verificar Antes de Usar

Pré-verificação se um benefício pode ser usado sem realmente registrar o uso.

```
GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/verify
```

**Query:** `?userId={uuid}&editionNumber={number}&secret={hash}`

Retorna o resultado da verificação sem efeitos colaterais.

---

## Fluxo G: Acesso via Código de Compartilhamento (Público)

```
GET /token-pass-share-codes/tenants/{tenantId}/{code}
```

Nenhuma autenticação necessária. Retorna benefícios do pass com metadados do token para a edição específica.

---

## Rastreamento de Uso

### Visualizar Registros de Uso

```
GET /token-pass-benefits/tenants/{tenantId}/usages
```

Retorna uma lista paginada de todos os registros de uso de benefícios com detalhes do usuário, timestamps e números de edição.

### Exportar Relatório de Uso

```
GET /token-pass-benefits/tenants/{tenantId}/usages/xls
```

Retorna arquivo XLS com dados de uso. Rota do frontend: `/dash/tokens/pass/usages/{collectionId}`.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | `invalid-secret-hash` | Segredo do QR não corresponde | Regenere o QR code no dispositivo do usuário |
| 400 | `secret-expired` | QR dinâmico com mais de 45 segundos | Regenere o QR code |
| 400 | `benefit-not-started` | Antes de `eventStartsAt` | Aguarde o início do evento |
| 400 | `benefit-expired` | Após `eventEndsAt` | O benefício não está mais disponível |
| 400 | `wrong-checkin-time` | Fora da janela de check-in | Verifique a programação |
| 400 | `usage-limit-reached` | Atingiu o `useLimit` para esta edição | Não há mais usos disponíveis |
| 400 | `temporary-usage-limit-reached` | Atingiu o limite da regra de uso para o período | Aguarde a reinicialização (diário/semanal/mensal) |
| 403 | `invalid-token-owner` | Usuário não possui a edição | Verifique a posse do token |
| 403 | `self-use-not-allowed` | `allowSelfUse: false` | Deve ser registrado por um operador |
| 403 | `unsatisfied-benefit-requirements` | Metadados do token não correspondem | Verifique os requisitos do token |
| 403 | `unauthorized-operator` | Operador não atribuído | Atribua o operador a este benefício |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | QR code rejeitado após atualização | QR codes dinâmicos expiram em 45 segundos. Não atualize cedo demais |
| 2 | Contagem de uso não reinicia | Verifique `usageRule` -- sem ela, as contagens são permanentes |
| 3 | Usuário não consegue fazer autouso | `allowSelfUse` tem valor padrão `false`. Habilite-o para benefícios digitais |
| 4 | Operador não consegue registrar uso | O operador deve ser explicitamente atribuído ao benefício |
| 5 | Benefício mostra `unavailable` | Verifique: datas do evento, limites de uso, janelas de check-in e requisitos |
| 6 | Busca por CPF falha | O CPF deve existir nos documentos KYC do usuário para o tenant |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Ciclo de Vida do Pass | Criar e configurar passes | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Referência da API | Detalhes completos dos endpoints | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) |
| KYC (para busca por CPF) | CPF dos documentos | [FLOW_CONTACTS_KYC_SUBMISSION](../contacts/FLOW_CONTACTS_KYC_SUBMISSION.md) |
