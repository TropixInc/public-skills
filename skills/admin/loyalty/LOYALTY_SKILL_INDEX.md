---
id: LOYALTY_SKILL_INDEX
title: "Índice de Skills de Fidelidade"
module: offpix/loyalty
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Índice de Skills de Fidelidade

Documentação para o módulo de Fidelidade do W3Block. Cobre contratos de token ERC-20, programas de fidelidade, regras de recompensa (add, multiply, split, cashback com comissões multinível), pagamentos baseados em pontos, staking e distribuição de recompensas via webhook.

**Serviço:** KEY (pixway-registry) | **Swagger:** https://api.w3block.io/docs/

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|----------|---------|-----------|--------|-------------|
| 1 | [LOYALTY_API_REFERENCE](./LOYALTY_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos de entidades | Implementado | Referência da API a qualquer momento |
| 2 | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) | 1.0.0 | Criar contrato ERC-20, deploy on-chain, criar programa de fidelidade, configurar regras | Implementado | Configuração inicial do sistema de fidelidade |
| 3 | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) | 1.0.0 | Mint, transferir, queimar tokens; verificar saldos e histórico de transações | Implementado | Gerenciamento manual de pontos |
| 4 | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) | 1.0.0 | Depósitos, cashback, comissões multinível, pagamentos, staking, rollbacks | Implementado | Recompensas automatizadas e gasto de pontos |

---

## Guia Rápido

### Para uma implementação básica de fidelidade:

```
1. Leia: FLOW_LOYALTY_CONTRACT_SETUP.md       -> Criar contrato ERC-20 e programa de fidelidade
2. Leia: FLOW_LOYALTY_BALANCE_OPERATIONS.md   -> Mintar tokens para usuários, verificar saldos
3. Leia: FLOW_LOYALTY_REWARDS_PAYMENTS.md     -> Configurar recompensas automatizadas e pagamentos
4. Consulte: LOYALTY_API_REFERENCE.md          -> Para enums, DTOs, casos especiais
```

### Implementação mínima API-first (5 chamadas):

```
1. POST /{companyId}/erc20-contracts             -> Criar contrato de token (rascunho)
2. PATCH /{companyId}/erc20-contracts/{id}/publish -> Deploy on-chain
3. POST /{companyId}/loyalties/admin              -> Criar programa de fidelidade
4. POST /{companyId}/loyalties/admin/{id}/rules   -> Adicionar regra de recompensa
5. PATCH /{companyId}/erc20-tokens/{id}/mint      -> Mintar pontos para um usuário
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Contrato ainda em `draft` ao tentar usar | Deve publicar o contrato primeiro. Faça polling do status até `published`. |
| 2 | Programa de fidelidade sem `contractId` | Operações on-chain falham sem um contrato ERC-20 vinculado. Sempre atribua um. |
| 3 | Colisão de prioridade de regra na criação | Cada regra precisa de uma prioridade única dentro da fidelidade (exceto prioridade 0). |
| 4 | Webhook de depósito retorna 404 | O campo `action` deve corresponder exatamente ao `name` de uma regra (slugificado, minúsculas). |
| 5 | Pontos não aparecem após mint/depósito | Saldos são cacheados por 5-10 segundos. Pontos também podem ter uma data `available` futura. |
| 6 | Pagamento falha com InsufficientBalanceException | Sempre faça preview do pagamento primeiro e verifique o saldo do usuário. |
| 7 | `userCode` não corresponde no pagamento | O `userId` e `userCode` devem pertencer ao mesmo usuário. |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Criar um novo token e programa de fidelidade | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Configurar regras de cashback com comissões multinível | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) (Passo 5) |
| Mintar pontos para um usuário | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) (Passo 2) |
| Transferir pontos entre carteiras | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) (Passos 3-4) |
| Verificar o saldo de pontos de um usuário | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) (Passo 6) |
| Acionar recompensas automatizadas via webhook | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Passos 1-2) |
| Visualizar e executar pagamentos baseados em pontos | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Passos 4-5) |
| Dividir recompensas entre múltiplos beneficiários | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Passo 6) |
| Reverter transações de recompensa | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Passos 7-8) |
| Visualizar detalhes de staking e resgatar | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Passos 9-10) |
| Consultar todos os enums e códigos de erro | [LOYALTY_API_REFERENCE](./LOYALTY_API_REFERENCE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Config. Contrato | Op. Saldo | Recompensas e Pagamentos |
|----------|:-------:|:--------------:|:-----------:|:------------------:|
| POST /{companyId}/erc20-contracts | X | X | | |
| GET /{companyId}/erc20-contracts | X | | | |
| GET /{companyId}/erc20-contracts/:id | X | X | | |
| PATCH /{companyId}/erc20-contracts/:id | X | X | | |
| PATCH /{companyId}/erc20-contracts/:id/publish | X | X | | |
| GET /{companyId}/erc20-contracts/:id/estimate-gas | X | X | | |
| PATCH /{companyId}/erc20-contracts/:id/transfer-config | X | | | |
| PATCH /{companyId}/erc20-contracts/has-role | X | | | |
| PATCH /{companyId}/erc20-contracts/grant-role | X | | | |
| POST /{companyId}/loyalties/admin | X | X | | |
| GET /{companyId}/loyalties/admin | X | X | | |
| GET /{companyId}/loyalties/admin/:loyaltyId | X | X | | |
| PATCH /{companyId}/loyalties/admin/:loyaltyId | X | X | | |
| POST /{companyId}/loyalties/admin/:loyaltyId/rules | X | X | | |
| GET /{companyId}/loyalties/admin/:loyaltyId/rules/:ruleId | X | X | | |
| PATCH /{companyId}/loyalties/admin/:loyaltyId/rules/:ruleId | X | X | | |
| DELETE /{companyId}/loyalties/admin/:loyaltyId/rules/:ruleId | X | X | | |
| PATCH /{companyId}/erc20-tokens/:id/mint | X | | X | |
| PATCH /{companyId}/erc20-tokens/:id/transfer/admin | X | | X | |
| PATCH /{companyId}/erc20-tokens/:id/transfer/user | X | | X | |
| PATCH /{companyId}/erc20-tokens/:id/burn | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history/action/:actionId | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history/operator/:operatorId | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history/:userId | X | | X | |
| GET /{companyId}/loyalties/users/balance/:userId | X | | X | |
| GET /{companyId}/loyalties/users/balance/:userId/:loyaltyId | X | | X | |
| GET /{companyId}/loyalties/users/deferred | X | | X | X |
| GET /{companyId}/loyalties/users/deferred/xlsx | X | | X | |
| GET /{companyId}/loyalties/users/deferred/:userId | X | | X | |
| PATCH /{companyId}/loyalties/rewards/cashback/preview | X | | | X |
| PATCH /{companyId}/loyalties/rewards/payment/preview | X | | | X |
| PATCH /{companyId}/loyalties/rewards/payment | X | | | X |
| POST /{companyId}/loyalties/webhook/deposit | X | | | X |
| POST /{companyId}/loyalties/webhook/cashback-multilevel | X | | | X |
| POST /{companyId}/loyalties/webhook/rollback | X | | | X |
| GET /{companyId}/loyalties/webhook/rollback/:requestId | X | | | X |
| POST /{companyId}/loyalties/webhook/split-payees | X | | | X |
| GET /{companyId}/loyalties/users/:userId/cashback-multilevel-report/:loyaltyId | X | | | X |
| GET /{companyId}/loyalties/users/:userId/staking/:loyaltyId | X | | | X |
| PATCH /{companyId}/loyalties/users/:userId/staking/:loyaltyId/redeem | X | | | X |
