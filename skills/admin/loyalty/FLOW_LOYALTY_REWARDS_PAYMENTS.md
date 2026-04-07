---
id: FLOW_LOYALTY_REWARDS_PAYMENTS
title: "Fidelidade - Recompensas, Cashback e Pagamentos"
module: offpix/loyalty
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - loyalty
  - rewards
  - cashback
  - payments
  - staking
  - webhooks
depends_on:
  - FLOW_LOYALTY_CONTRACT_SETUP
  - FLOW_LOYALTY_BALANCE_OPERATIONS
---

# Fidelidade - Recompensas, Cashback e Pagamentos

## Visão Geral

Este fluxo cobre o sistema automatizado de recompensas e operações de pagamento: acionar depósitos e cashback (incluindo comissões multinível), visualizar e executar pagamentos baseados em pontos, dividir recompensas entre múltiplos beneficiários, gerenciamento de staking e reversão de transações. Estas operações se baseiam no programa de fidelidade e nas regras configuradas no fluxo de setup, e alimentam a economia de pontos para transações em loja e online.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | Token de acesso JWT | Autenticação do serviço ID |
| Programa de fidelidade ativo | Com contrato ERC-20 e pelo menos uma regra | [Fluxo de Configuração de Contrato](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Carteira do usuário | Usuário deve ter um endereço de carteira | Criação de usuário / provisionamento de carteira |
| Nome da regra | O campo `name` da regra a acionar | Configuração de regras de fidelidade |

## Entidades e Relacionamentos

```
LoyaltiesEntity (1) ──→ (N) LoyaltiesRulesEntity
      │                          │
      │                          └── Correspondida pelo nome da ação durante depósito/cashback
      │
      └──→ (N) LoyaltiesDeferredEntity
                    │
                    ├── status: pending → started → success (ou failed)
                    ├── status: deferred (agendada para o futuro)
                    ├── status: pool (agrupada para lote)
                    ├── status: waiting_for_rollback → rollback
                    │
                    ├── parentMinterId (link para o deferred de mint original)
                    ├── parentTransferId (link para o deferred de transferência original)
                    └── stakingRuleId (link para a regra de staking)
```

**Conceitos-chave:**
- **Webhook de depósito** aciona a distribuição de recompensas. O sistema encontra a regra correspondente pelo nome da `action`, calcula o valor da recompensa e cria transação(ões) diferida(s).
- **Cashback multinível** processa um valor de compra através da regra de cashback, distribuindo cashback direto ao comprador e comissões indiretas pela cadeia de indicação.
- **Pagamento** é o inverso: usuários gastam pontos para obter desconto em uma compra. Pontos são queimados/transferidos da carteira do usuário.
- **Transações diferidas** são os registros atômicos de todos os movimentos de pontos. Elas rastreiam status, timing de execução, metadados e podem ser revertidas.
- **Staking** distribui automaticamente recompensas periódicas para usuários que atendem requisitos de retenção.

---

## Fluxo: Depositar Recompensas (Webhook)

Use este fluxo para acionar a distribuição de pontos com base em um evento externo (ex.: uma compra, uma indicação ou qualquer ação personalizada).

### Passo 1: Acionar Depósito

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/loyalties/webhook/deposit` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "user-uuid",
  "action": "welcome-bonus"
}
```

**Requisição Completa:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "user-uuid",
  "action": "cashback_multilevel",
  "amount": "100.00",
  "description": "Cashback from order #1234",
  "metadata": {
    "orderId": "order-uuid",
    "totalAmount": "100.00",
    "buyerName": "John Doe"
  }
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | Programa de fidelidade a usar |
| `userId` | UUID | Sim | Usuário recebendo a recompensa |
| `action` | string | Sim | Deve corresponder ao campo `name` de uma regra no programa de fidelidade |
| `amount` | string | Não | Valor base para cálculo de recompensa. Obrigatório para regras `multiply`/`cashback`. |
| `description` | string | Não | Descrição legível da transação |
| `metadata` | object | Não | Passado para templates Handlebars em `descriptionTemplate`. Inclua quaisquer variáveis que seus templates referenciem (ex.: `{{orderId}}`, `{{buyerName}}`). |

**Resposta (200):**
```json
{
  "requestId": "request-uuid"
}
```

**Observações:**
- O campo `action` é correspondido com o `name` da regra para determinar qual regra aplicar. Se nenhuma regra correspondente for encontrada, uma `ThereIsNoRuleForThisActionException` é lançada.
- O `requestId` na resposta pode ser usado posteriormente para operações de rollback.
- Para regras do tipo `add`, o campo `value` da regra é usado como a quantidade fixa, independente do `amount` na requisição.
- Para regras do tipo `multiply`, o `amount` da requisição é multiplicado pelo `value` da regra.
- Para regras do tipo `cashback`, o `amount` da requisição é multiplicado pelo `value` da regra (porcentagem).
- Transações diferidas são criadas com a expressão de data `available` da regra, controlando quando os pontos realmente se tornam utilizáveis.

---

## Fluxo: Cashback Multinível (Webhook)

Use este fluxo para cashback baseado em compra com comissões multinível.

### Passo 2: Acionar Cashback Multinível

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/loyalties/webhook/cashback-multilevel` | Bearer token (Integration) | application/json |

**Requisição:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "buyer-user-uuid",
  "amount": "500.00",
  "description": "Purchase cashback from order #5678",
  "operatorUserId": "operator-uuid",
  "metadata": {
    "orderId": "order-uuid",
    "productName": "Premium Subscription"
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | Programa de fidelidade |
| `userId` | UUID | Sim | O comprador (destinatário do cashback direto) |
| `amount` | string | Sim | Valor da compra |
| `description` | string | Não | Descrição da transação |
| `operatorUserId` | UUID | Não | O operador/loja que processou a venda |
| `metadata` | object | Não | Metadados customizados |

**Resposta (200):**
```json
{
  "requestId": "request-uuid"
}
```

**Como funciona o cashback multinível:**

Dada uma regra com `value: "0.05"` (5% de cashback) e `cashbackConfigurations.indirectCashback: [0.03, 0.02, 0.01]`:

1. **Cashback direto** para o comprador: `amount * value` = 500 * 0.05 = 25 pontos
2. **Comissão N1** para o indicador do comprador: `amount * indirectCashback[0]` = 500 * 0.03 = 15 pontos
3. **Comissão N2** para o indicador do N1: `amount * indirectCashback[1]` = 500 * 0.02 = 10 pontos
4. **Comissão N3** para o indicador do N2: `amount * indirectCashback[2]` = 500 * 0.01 = 5 pontos
5. **Destinatário final** (se configurado): recebe a porcentagem `finalRecipientRate`
6. **Pontos restantes** (se a cadeia estiver incompleta): vão para `indirectCashbackUserIdToReceiveRemainingPoints`

Os pontos de cada nível se tornam disponíveis de acordo com suas respectivas expressões de data `available`.

**Webhooks de substituição:** Se `cashbackOverride`, `rateOverride` ou `overrideMultilevelCommission` estiverem habilitados na configuração de cashback, o sistema chama o webhook externo configurado para obter valores customizados antes de calcular as recompensas.

---

## Fluxo: Preview de Cashback

Visualize o cashback antes de acioná-lo.

### Passo 3: Preview de Cashback

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/loyalties/rewards/cashback/preview` | Bearer token | application/json |

**Requisição:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "100.00",
  "userId": "user-uuid",
  "action": "cashback_multilevel"
}
```

**Resposta (200):**
```json
{
  "amount": "100.00",
  "pointsCashback": "5",
  "currency": "BRL",
  "contractName": "Loyalty Points",
  "contractSymbol": "LPT",
  "currencyEquivalent": "0.50"
}
```

| Campo da Resposta | Tipo | Descrição |
|-------------------|------|-----------|
| `amount` | string | Valor original da compra |
| `pointsCashback` | string | Pontos que o usuário ganharia |
| `currency` | string | Moeda das configurações de pagamento da fidelidade |
| `contractName` | string | Nome do token ERC-20 |
| `contractSymbol` | string | Símbolo do token ERC-20 |
| `currencyEquivalent` | string | Valor em moeda dos pontos de cashback |

---

## Fluxo: Pagamento Baseado em Pontos

Use este fluxo para pagamentos em loja ou online onde usuários gastam pontos de fidelidade para obter desconto.

### Passo 4: Preview de Pagamento

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/loyalties/rewards/payment/preview` | Bearer token | application/json |

**Requisição:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "50.00",
  "points": "200",
  "userId": "user-uuid",
  "userCode": "ABC123"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | Programa de fidelidade |
| `amount` | string | Sim | Valor total da compra (deve ser uma string numérica positiva) |
| `points` | string | Sim | Pontos que o usuário deseja gastar |
| `userId` | UUID | Sim | Usuário realizando o pagamento |
| `userCode` | string | Sim | Código de verificação (usado para pagamentos em loja para confirmar identidade) |

**Resposta (200):**
```json
{
  "total": "50.00",
  "amount": "30.00",
  "points": "200",
  "pointsCashback": "0",
  "discount": "20.00",
  "currency": "BRL",
  "contractName": "Loyalty Points",
  "contractSymbol": "LPT"
}
```

| Campo da Resposta | Tipo | Descrição |
|-------------------|------|-----------|
| `total` | string | Valor original da compra |
| `amount` | string | Valor restante após o desconto |
| `points` | string | Pontos sendo gastos |
| `pointsCashback` | string | Quaisquer pontos de cashback ganhos neste pagamento |
| `discount` | string | Valor do desconto em moeda |
| `currency` | string | Código da moeda |
| `contractName` | string | Nome do token |
| `contractSymbol` | string | Símbolo do token |

### Passo 5: Executar Pagamento

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/loyalties/rewards/payment` | Bearer token (Admin, LoyaltyOperator) | application/json |

**Requisição:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "50.00",
  "points": "200",
  "userId": "user-uuid",
  "userCode": "ABC123",
  "description": "In-store purchase at Location X"
}
```

**Resposta (200):**
```json
{
  "requestId": "request-uuid"
}
```

**Observações:**
- O `userCode` e `userId` devem corresponder ao mesmo usuário. Uma incompatibilidade lança `UserNotMatchException`.
- O usuário deve ter saldo suficiente. Caso contrário, `InsufficientBalanceException` é lançada.
- O `operatorUserId` é automaticamente definido a partir do usuário autenticado (o caixa/operador).
- Pontos são queimados ou transferidos com base no `tokenTransferabilityMethod` do programa de fidelidade.
- O campo `amount` é validado para ser uma string numérica positiva.

---

## Fluxo: Dividir Beneficiários (Webhook)

Distribua recompensas entre múltiplos endereços de carteira com divisões baseadas em porcentagem.

### Passo 6: Dividir Recompensas

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/loyalties/webhook/split-payees` | Bearer token (Integration) | application/json |

**Requisição:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "total": "1000",
  "payees": [
    {
      "walletAddress": "0xAddress1...",
      "percent": "0.60",
      "metadata": { "role": "owner" },
      "descriptionTemplate": "Owner share: {{amount}} points",
      "available": "1s",
      "sendEmail": true
    },
    {
      "walletAddress": "0xAddress2...",
      "percent": "0.25",
      "metadata": { "role": "partner" },
      "available": "7d",
      "sendEmail": false
    },
    {
      "walletAddress": "0xAddress3...",
      "percent": "0.15",
      "metadata": { "role": "affiliate" },
      "available": "30d",
      "sendEmail": false
    }
  ]
}
```

**Observações:**
- O valor de cada beneficiário é calculado como `total * percent`.
- Cada beneficiário pode ter sua própria expressão de data `available` (diferentes períodos de carência).
- Os valores de `percent` não precisam somar exatamente 1.0 -- excesso ou restante é silenciosamente ignorado.
- Útil para modelos de compartilhamento de receita onde diferentes stakeholders recebem porções diferentes dos pontos de recompensa.

---

## Fluxo: Reverter Transações

Reverta transações diferidas criadas anteriormente (ex.: quando um pedido é cancelado).

### Passo 7: Solicitar Rollback

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/loyalties/webhook/rollback` | Bearer token (Integration) | application/json |

**Requisição:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "requestId": "original-request-uuid",
  "metadata": {
    "reason": "Order #1234 cancelled by customer"
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | Programa de fidelidade |
| `requestId` | string | Não | O `requestId` da resposta original de depósito/cashback |
| `metadata` | object | Não | Motivo e contexto do rollback |

**Resposta (200):** Retorna informações de processamento do rollback.

### Passo 8: Verificar Status do Rollback

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/loyalties/webhook/rollback/{requestId}` | Bearer token (Integration) | -- |

**Resposta (200):**
```json
{
  "completed": true
}
```

**Observações:**
- O rollback é assíncrono. Use o endpoint de status para fazer polling até a conclusão.
- Transações revertidas mudam o status do seu estado atual para `waiting_for_rollback` e depois `rollback`.
- Se o `requestId` corresponder a um rollback já processado, uma `DuplicateRequestException` é lançada.

---

## Fluxo: Recompensas de Staking

Staking distribui automaticamente recompensas periódicas para usuários que atendem requisitos de retenção.

### Passo 9: Visualizar Resumo de Staking

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/loyalties/users/{userId}/staking/{loyaltyId}` | Bearer token | -- |

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "deferred-uuid",
      "amount": "50",
      "status": "deferred",
      "executeAt": "2026-04-08T00:00:00Z",
      "expiresAt": "2026-05-08T00:00:00Z",
      "withdrawableAt": "2026-04-15T00:00:00Z",
      "isExpired": false,
      "isWithdrawal": false
    }
  ],
  "meta": {
    "totalItems": 1,
    "totalPages": 1,
    "currentPage": 1,
    "itemsPerPage": 10
  },
  "summary": {
    "availableToRedeem": "150",
    "delivering": "50",
    "expired": "0",
    "received": "200"
  }
}
```

| Campo do Resumo | Descrição |
|-----------------|-----------|
| `availableToRedeem` | Pontos prontos para resgate agora |
| `delivering` | Pontos em trânsito (execução pendente) |
| `expired` | Pontos que expiraram |
| `received` | Total de pontos recebidos do staking |

### Passo 10: Resgatar Recompensas de Staking

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem` | Bearer token | -- |

**Corpo da requisição:** Nenhum.

**Resposta (204):** Sem conteúdo.

**Observações:**
- Resgata todas as recompensas de staking disponíveis para o usuário.
- Regras de staking podem ter `requireRedeem: true`, significando que recompensas permanecem no status `deferred` até que o usuário resgate explicitamente.
- Se `redeemExpirationDays` estiver definido na regra de staking, recompensas não resgatadas expiram após esse número de dias.

---

## Fluxo: Relatório de Cashback Multinível

Visualize estatísticas agregadas sobre os ganhos de cashback multinível de um usuário.

### Passo 11: Obter Relatório

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/loyalties/users/{userId}/cashback-multilevel-report/{loyaltyId}` | Bearer token (User) | -- |

**Resposta (200):**
```json
{
  "reportPerLevel": [
    { "level": "1", "totalAmount": 500, "transactionsCount": 25 },
    { "level": "2", "totalAmount": 200, "transactionsCount": 15 },
    { "level": "3", "totalAmount": 50, "transactionsCount": 5 },
    { "level": "businessReferral", "totalAmount": 100, "transactionsCount": 5 },
    { "level": "areaReferral", "totalAmount": 30, "transactionsCount": 2 }
  ]
}
```

**Observações:**
- Os dados vêm de uma view materializada para performance. Cache de 5 minutos.
- Níveis numéricos correspondem aos índices do array `indirectCashback` (base 1).
- Níveis nomeados (`businessReferral`, `areaReferral`) correspondem a recompensas de embaixadores de zona e indicadores de negócios.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 404 | ThereIsNoRuleForThisActionException | `action` não corresponde a nenhum nome de regra | Verifique os nomes das regras no programa de fidelidade |
| 404 | UserByCodeNotFoundException | Código de usuário não encontrado | Verifique o código do usuário |
| 400 | UserNotMatchException | userId e userCode não correspondem | Certifique-se de que ambos identificam o mesmo usuário |
| 400 | InsufficientBalanceException | Pontos insuficientes para pagamento | Verifique o saldo antes do pagamento |
| 400 | LoyaltiesNotHaveContractException | Fidelidade não tem contrato ERC-20 vinculado | Vincule um contrato publicado à fidelidade |
| 400 | DuplicateRequestException | Mesma requisição de rollback enviada duas vezes | Use o requestId original para verificar o status |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Webhook de depósito retorna 404 | O campo `action` deve corresponder exatamente ao `name` de uma regra (minúsculas, slugificado). Verifique a grafia e caixa. |
| 2 | Pontos não aparecem no saldo após depósito | Pontos podem ter uma data `available` futura da regra. Verifique a data `executeAt` da transação diferida. |
| 3 | Preview de pagamento funciona mas execução falha | O saldo pode ter mudado entre preview e execução. Sempre re-verifique ou execute imediatamente após o preview. |
| 4 | Cashback multinível não alcança todos os níveis | A cadeia de indicação deve estar estabelecida (relacionamentos de usuários). Indicadores ausentes fazem com que os pontos restantes vão para o usuário de fallback. |
| 5 | Rollback não conclui | O rollback é assíncrono. Faça polling em `/rollback/{requestId}` em vez de assumir conclusão imediata. |
| 6 | Validação de `amount` falha no pagamento | O amount deve ser uma string numérica positiva (ex.: `"50.00"`, não `"0"` ou negativo). |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Configuração de Contrato e Programa | Deve ser feito primeiro | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Operações de Saldo | Mint/transferência direto | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) |
| Pedidos de Commerce | Aciona cashback na compra | Módulo de Commerce |
