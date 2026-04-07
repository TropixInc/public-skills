---
id: FLOW_SETTINGS_BILLING_MANAGEMENT
title: "Configurações - Gerenciamento de Faturamento"
module: offpix/settings
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - settings
  - billing
  - plans
  - credit-card
  - subscription
  - usage
depends_on:
  - SETTINGS_SKILL_INDEX
  - SETTINGS_API_REFERENCE
---

# Gerenciamento de Faturamento

## Visão Geral

Este fluxo cobre o ciclo completo de faturamento de um tenant W3Block: navegar pelos planos disponíveis, configurar um cartão de crédito, assinar um plano, monitorar uso e custos, simular cobranças futuras, revisar o histórico de ciclos de cobrança e cancelar assinaturas. Também cobre operações administrativas como registrar eventos de uso e acionar faturas manuais.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `tenantId` | UUID do tenant | Fluxo de auth / configuração de ambiente |
| Bearer token | Token de acesso JWT (papel Admin) | Autenticação no serviço ID |
| Cartão de crédito | Método de pagamento registrado no provedor de pagamento | Configuração do provedor de pagamento externo |

## Entidades e Relacionamentos

```
BillingPlanEntity (1) -----> (N) TenantPlanEntity -----> (1) TenantEntity
                                      |
                                      +-----> (N) TenantBillingLogEntity
                                      |
                                      +-----> (N) TenantPlanLogEntity (audit)

BillingUsageEntity (N) -----> (1) TenantEntity
BillingCurrencyExchangeRateEntity (standalone, for multi-currency)
```

**Conceitos-chave:**
- Cada tenant possui exatamente um TenantPlanEntity ativo
- O uso é rastreado em BillingUsageEntity com contadores de ciclo e de vida inteira
- Cada ciclo de cobrança produz um TenantBillingLogEntity com um `finalReport` detalhado
- Mudanças de plano são auditadas em TenantPlanLogEntity

---

## Fluxo: Seleção e Assinatura de Plano

### Passo 1: Listar Planos Disponíveis

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/billing/plans` | Nenhuma (Público) | - |

**Requisição:** Nenhum corpo necessário. Opcionalmente filtre com parâmetros de query.

```
GET /billing/plans?isPublic=true&limit=10
```

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "free-plan-uuid",
      "name": "Free",
      "basePrice": 0,
      "isDefault": true,
      "isPublic": true,
      "limits": {
        "nftContracts": { "monthly": 1, "hardPerMonth": 1 },
        "nftMints": { "monthly": 10, "hardPerMonth": 15 }
      },
      "features": {
        "enabledFeatures": ["MINT_ERC_721", "BASIC_TOKEN_PASS"]
      },
      "trial": { "testDays": 0 }
    },
    {
      "id": "pro-plan-uuid",
      "name": "Professional",
      "basePrice": 99.00,
      "isDefault": false,
      "isPublic": true,
      "limits": {
        "nftContracts": { "monthly": 10, "hardPerMonth": 15 },
        "nftMints": { "monthly": 500, "hardPerMonth": null }
      },
      "pricing": {
        "extraNftMintPrice": 0.50,
        "nftTransactionPrice": 0.10
      },
      "features": {
        "enabledFeatures": [
          "MINT_ERC_721", "MINT_ERC_20", "COMMERCE",
          "LOYALTY", "BASIC_TOKEN_PASS", "FULL_TOKEN_PASS"
        ]
      },
      "trial": { "testDays": 14 }
    }
  ],
  "meta": { "totalItems": 3, "currentPage": 1 }
}
```

**Observações:**
- Planos com `isDefault: true` são atribuídos automaticamente a novos tenants
- Compare os arrays `enabledFeatures` para entender as diferenças de funcionalidades entre planos
- `trial.testDays` indica quantos dias de teste gratuito estão incluídos
- Limites com `hardPerMonth: null` significam que não há limite rígido (excedente é cobrado por unidade)

### Passo 2: Configurar Cartão de Crédito

Um cartão de crédito válido deve ser configurado antes de assinar um plano pago.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/billing/{tenantId}/credit-card` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "creditCardId": "card_abc123"   // required
}
```

**Resposta (200):**
```json
{
  "tenantId": "tenant-uuid",
  "planId": "free-plan-uuid",
  "primaryCreditCardId": "card_abc123",
  "startCycleDate": "2026-03-01T00:00:00.000Z",
  "endCycleDate": "2026-03-31T23:59:59.000Z"
}
```

**Observações:**
- O `creditCardId` vem da integração com o provedor de pagamento (ex.: token de cartão do Stripe)
- Você pode atualizar o cartão de crédito a qualquer momento; o novo cartão será usado na próxima cobrança
- Apenas um cartão principal por tenant

### Passo 3: Assinar um Plano

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/billing/{tenantId}/plan` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "planId": "pro-plan-uuid"       // required
}
```

**Requisição Completa:**
```json
{
  "planId": "pro-plan-uuid",      // required
  "coupon": "LAUNCH2026"          // optional -- cupom de desconto
}
```

**Resposta (200):**
```json
{
  "tenantId": "tenant-uuid",
  "planId": "pro-plan-uuid",
  "customLimits": null,
  "startCycleDate": "2026-03-31T00:00:00.000Z",
  "endCycleDate": "2026-04-30T23:59:59.000Z",
  "primaryCreditCardId": "card_abc123",
  "coupon": "LAUNCH2026"
}
```

**Observações:**
- As datas do ciclo de cobrança são redefinidas para começar a partir de hoje
- Um TenantPlanLogEntity é criado com `oldPlanId`, `newPlanId` e `userId`
- Se estiver migrando de um plano pago, o ciclo anterior pode ser faturado
- Um cartão de crédito é obrigatório para planos não gratuitos

---

## Fluxo: Monitoramento de Uso e Custos

### Passo 4: Verificar Estado Atual do Faturamento

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/billing/{tenantId}/state` | Bearer token (Admin) | - |

**Resposta (200):**
```json
{
  "plan": {
    "id": "pro-plan-uuid",
    "name": "Professional",
    "basePrice": 99.00
  },
  "tenantPlan": {
    "tenantId": "tenant-uuid",
    "planId": "pro-plan-uuid",
    "customLimits": null,
    "startCycleDate": "2026-03-01T00:00:00.000Z",
    "endCycleDate": "2026-03-31T23:59:59.000Z",
    "primaryCreditCardId": "card_abc123",
    "coupon": null
  },
  "features": {
    "enabledFeatures": ["MINT_ERC_721", "MINT_ERC_20", "COMMERCE", "LOYALTY"]
  }
}
```

### Passo 5: Visualizar Resumo de Uso

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/billing/{tenantId}/billing-summary` | Bearer token (Admin) | - |

**Resposta (200):**
```json
{
  "currentCycle": {
    "startDate": "2026-03-01T00:00:00.000Z",
    "endDate": "2026-03-31T23:59:59.000Z"
  },
  "basePlanCost": 99.00,
  "usageCosts": [
    {
      "type": "KEY_NFT_MINTED",
      "count": 520,
      "includedInPlan": 500,
      "overage": 20,
      "overageCost": 10.00
    },
    {
      "type": "NFT_TRANSACTION",
      "count": 45,
      "cost": 4.50
    }
  ],
  "totalEstimatedCost": 113.50
}
```

**Observações:**
- `includedInPlan` mostra quantas unidades são cobertas pelo preço base do plano
- `overage` é a contagem que excede o limite do plano
- `overageCost` = excedente * preço por unidade da precificação do plano

### Passo 6: Listar Registros Detalhados de Uso

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/billing/{tenantId}/billing-usages` | Bearer token (Admin) | - |

**Resposta (200):**
```json
{
  "items": [
    {
      "tenantId": "tenant-uuid",
      "type": "KEY_NFT_MINTED",
      "isLifeTime": false,
      "value": 520,
      "metadata": {}
    },
    {
      "tenantId": "tenant-uuid",
      "type": "KEY_NFT_MINTED",
      "isLifeTime": true,
      "value": 8400,
      "metadata": {}
    }
  ],
  "meta": { "totalItems": 22, "currentPage": 1 }
}
```

**Observações:**
- Registros com `isLifeTime: false` são contadores de ciclo (resetados a cada período de cobrança)
- Registros com `isLifeTime: true` são contadores acumulativos (nunca resetados)
- Restrição de unicidade: `(tenantId, type, isLifeTime)` -- um registro por combinação

### Passo 7: Simular um Evento de Faturamento

Antes de realizar uma operação custosa, simule seu custo.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/billing/{tenantId}/simulate-billing-event-usage` | Bearer token (Admin) | - |

**Requisição:**
```
GET /billing/{tenantId}/simulate-billing-event-usage?type=KEY_NFT_MINTED&value=50
```

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `type` | BillingUsageKnownTypes | Sim | Tipo de evento de uso |
| `value` | number | Não | Unidades a simular (padrão: 1) |

**Resposta (200):**
```json
{
  "type": "KEY_NFT_MINTED",
  "currentUsage": 480,
  "planLimit": 500,
  "simulatedAdditional": 50,
  "wouldExceedBy": 30,
  "estimatedOverageCost": 15.00
}
```

**Observações:**
- Não modifica nenhum dado -- é puramente uma simulação de leitura
- Use isso para mostrar aos usuários os custos projetados em um diálogo de confirmação antes de mintar, criar contratos, etc.

---

## Fluxo: Histórico de Ciclos de Cobrança

### Passo 8: Visualizar Ciclos de Cobrança Anteriores

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/billing/{tenantId}/cycles` | Bearer token (Admin) | - |

**Resposta (200):**
```json
{
  "items": [
    {
      "tenantId": "tenant-uuid",
      "startCycleDate": "2026-02-01T00:00:00.000Z",
      "endCycleDate": "2026-02-28T23:59:59.000Z",
      "paymentDate": "2026-03-01T00:05:00.000Z",
      "creditCardId": "card_abc123",
      "cyclePrice": 124.50,
      "paidValue": 124.50,
      "paymentTries": 1,
      "planId": "pro-plan-uuid",
      "customLimits": null,
      "finalReport": [
        {
          "type": "KEY_NFT_MINTED",
          "count": 550,
          "includedInPlan": 500,
          "overage": 50,
          "overageCost": 25.00
        },
        {
          "type": "NFT_TRANSACTION",
          "count": 5,
          "cost": 0.50
        }
      ]
    }
  ],
  "meta": { "totalItems": 6, "currentPage": 1 }
}
```

**Observações:**
- `finalReport` contém o detalhamento por tipo de uso daquele ciclo
- `paymentTries` > 1 indica tentativas de pagamento malsucedidas antes do sucesso
- `cyclePrice` é o total calculado; `paidValue` é o que foi efetivamente cobrado (pode diferir com cupons)

---

## Fluxo: Cancelamento de Assinatura

### Passo 9: Cancelar Assinatura

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/billing/{tenantId}/cancel` | Bearer token (Admin) | - |

**Requisição:** Nenhum corpo necessário.

**Resposta (200):**
```json
{
  "tenantId": "tenant-uuid",
  "planId": "free-plan-uuid",
  "customLimits": null,
  "startCycleDate": "2026-03-31T00:00:00.000Z",
  "endCycleDate": "2026-04-30T23:59:59.000Z",
  "primaryCreditCardId": "card_abc123",
  "coupon": null
}
```

**Observações:**
- Reverte o tenant para o plano padrão (gratuito) imediatamente
- Funcionalidades do plano pago são desabilitadas imediatamente
- O cartão de crédito permanece cadastrado mas não será cobrado
- Uma entrada de log de auditoria (TenantPlanLogEntity) é criada

---

## Fluxo: Operações Administrativas

Estes endpoints são para operadores da plataforma, não para administradores de tenants.

### Passo 10: Registrar Evento de Uso (Integração/SuperAdmin)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/billing/{tenantId}/register-billing-usage` | Bearer token (SuperAdmin/Integration) | application/json |

**Requisição Mínima:**
```json
{
  "type": "KEY_NFT_MINTED"       // required
}
```

**Requisição Completa:**
```json
{
  "type": "KEY_NFT_MINTED",      // required
  "value": 5,                     // optional -- unidades consumidas (padrão: 1)
  "metadata": {                   // optional -- contexto do evento
    "contractId": "contract-uuid",
    "collectionId": "collection-uuid"
  }
}
```

**Observações:**
- Tipicamente chamado por serviços internos (chave de API de Integration) quando eventos faturáveis ocorrem
- Incrementa tanto os contadores de ciclo quanto os de vida inteira
- Se um limite rígido for excedido, a operação pode ser rejeitada a montante

### Passo 11: Acionar Fatura Manual (SuperAdmin)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/billing/{tenantId}/invoice-and-charge` | Bearer token (SuperAdmin) | - |

**Requisição:** Nenhum corpo necessário.

**Resposta (200):** TenantBillingLogEntity com os detalhes da fatura e `finalReport`.

**Observações:**
- Força o fechamento imediato do ciclo, geração de fatura e cobrança no cartão de crédito
- Cria um registro TenantBillingLogEntity
- Incrementa `paymentTries` se a cobrança falhar
- Apenas SuperAdmin -- não acessível para administradores de tenants

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | VALIDATION_ERROR | `planId` inválido, `creditCardId` ausente ou tipo de uso inválido | Verifique os valores dos campos contra enums e registros existentes |
| 401 | UNAUTHORIZED | Token expirado ou ausente | Re-autentique via serviço ID |
| 403 | FORBIDDEN | Não-admin tentando mudar plano ou cancelar | Certifique-se de que o usuário tem papel Admin |
| 404 | PLAN_NOT_FOUND | `planId` não existe | Liste os planos primeiro para obter IDs válidos |
| 404 | TENANT_NOT_FOUND | `tenantId` não existe | Verifique o ID do tenant |
| 422 | NO_CREDIT_CARD | Tentando plano pago sem cartão de crédito | Configure o cartão de crédito primeiro (Passo 2) |
| 422 | HARD_LIMIT_EXCEEDED | Uso excede o limite rígido | Faça upgrade do plano ou aguarde o reset do ciclo |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Mudança de plano falha sem cartão de crédito | Sempre configure um cartão de crédito (Passo 2) antes de mudar para um plano pago (Passo 3) |
| 2 | Uso aparece zerado após mudança de plano | Contadores de ciclo são resetados quando o ciclo de cobrança é redefinido; contadores de vida inteira persistem |
| 3 | Limite flexível excedido mas operação funciona | Limites flexíveis são para faturamento, não para aplicação. Apenas limites `hard*` bloqueiam operações |
| 4 | Resumo de faturamento mostra custos inesperados | Verifique a precificação de excedente na definição do plano -- custos se acumulam por unidade além da quantidade incluída |
| 5 | Simulação mostra custo zero mas fatura é alta | A simulação é pontual no tempo; uso entre a simulação e o final da fatura pode se acumular |
| 6 | Cancelamento ainda mostra cartão de crédito | O cartão permanece cadastrado após o cancelamento; simplesmente não será cobrado no plano gratuito |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|----------------|-----------|
| Autenticação | Necessário para todos os endpoints autenticados | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Configuração do Tenant | Configure funcionalidades verificadas via faturamento | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Contratos e Tokens | Operações que geram eventos de uso de faturamento | [CONTRACTS_SKILL_INDEX](../contracts/CONTRACTS_SKILL_INDEX.md) |
| Comércio | Compras de produtos geram eventos de faturamento | [COMMERCE_SKILL_INDEX](../commerce/COMMERCE_SKILL_INDEX.md) |
