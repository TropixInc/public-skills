---
id: LOYALTY_API_REFERENCE
title: "Fidelidade - Referência da API"
module: offpix/loyalty
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - loyalty
  - erc20
  - cashback
  - staking
  - rewards
  - api-reference
---

# Referência da API de Fidelidade

Referência completa de endpoints para o módulo de Fidelidade da W3Block. Este módulo abrange quatro grupos de recursos -- Programas de Fidelidade (CRUD admin), Contratos ERC20 (criação e deploy de tokens), Tokens ERC20 (operações de mint/transfer/burn) e Usuários de Fidelidade (saldos, transações, staking, recompensas) -- todos servidos pelo backend KEY (pixway-registry).

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://api.w3block.io` |
| Staging | *(ambiente de staging disponível — use a URL base de staging)* |
| Swagger | https://api.w3block.io/docs/ |

## Autenticação

Endpoints marcados como **Auth: Bearer** requerem:

```
Authorization: Bearer {accessToken}
```

Os endpoints de fidelidade usam estes níveis de papel:
- **SuperAdmin / Admin** -- acesso completo a todas as operações de fidelidade e ERC20
- **LoyaltyOperator** -- pode listar fidelidades, ver saldos, pré-visualizar/executar pagamentos
- **User** -- pode ver próprio saldo e histórico de transações
- **Integration** -- endpoints de webhook serviço-a-serviço

---

## Enums

### LoyaltiesRuleType

| Valor | Descrição |
|-------|-----------|
| `add` | Adiciona uma quantidade fixa de pontos |
| `multiply` | Multiplica a quantidade base por um fator |
| `split` | Divide recompensa entre múltiplos pagadores |
| `cashback` | Cashback com suporte a comissão multinível |

### LoyaltiesDeferredStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Aguardando execução |
| `started` | Execução em andamento |
| `success` | Executado com sucesso on-chain |
| `failed` | Execução falhou |
| `deferred` | Agendado para execução futura |
| `pool` | Agrupado em pool para processamento em lote |
| `waiting_for_rollback` | Rollback solicitado, aguardando processamento |
| `rollback` | Rollback realizado com sucesso |

### TokenIssuanceMethod

| Valor | Descrição |
|-------|-----------|
| `mint` | Novos tokens são mintados para o destinatário |
| `transfer` | Tokens são transferidos de um endereço fonte (`tokenIssuanceAddress`) |

### TokenTransferabilityMethod

| Valor | Descrição |
|-------|-----------|
| `transfer` | Tokens são transferidos entre carteiras |
| `burn` | Tokens são queimados da carteira fonte |

### PointPrecisionOptions

| Valor | Descrição |
|-------|-----------|
| `integer` | Pontos são apenas números inteiros |
| `decimal` | Pontos suportam precisão decimal |

### LoyaltiesAction

| Valor | Descrição |
|-------|-----------|
| `multiply` | Multiplicar quantidade base |
| `sum` | Somar à quantidade base |
| `percentage` | Porcentagem da quantidade base |

### LoyaltiesProfile

| Valor | Descrição |
|-------|-----------|
| `cashback` | Recompensas tipo cashback |
| `commission` | Recompensas tipo comissão |

### ActionSubtype

| Valor | Descrição |
|-------|-----------|
| `cashback` | Cashback direto |
| `commission_l1` | Comissão nível 1 |
| `commission_l2` | Comissão nível 2 |
| `commission_l3` | Comissão nível 3 |
| `commission_l4` | Comissão nível 4 |
| `commission_lx` | Comissão de nível genérico (qualquer nível além de L4) |
| `finalRecipient` | Destinatário final dos pontos restantes |
| `businessReferral` | Recompensa de indicação comercial |
| `areaReferral` | Recompensa de indicação de área/zona |

### LoyaltyStakingAmountType

| Valor | Descrição |
|-------|-----------|
| `fixed` | Quantidade fixa por ciclo de staking |
| `holder_factor` | Quantidade calculada com base no saldo do titular multiplicado por um fator |

### ERC20ContractType

| Valor | Descrição |
|-------|-----------|
| `classic` | Token ERC20 padrão |
| `permissioned` | ERC20 permissionado com acesso baseado em papéis |
| `in_custody` | ERC20 custodial gerenciado pela plataforma |

### ContractStatus

| Valor | Descrição |
|-------|-----------|
| `draft` | Contrato criado mas não implantado |
| `publishing` | Deploy em andamento |
| `published` | Implantado com sucesso on-chain |
| `failed` | Deploy falhou |

### Erc20TransferModel

| Valor | Descrição |
|-------|-----------|
| `max_limit` | Limite máximo de transferência |
| `percentage` | Limite de transferência baseado em porcentagem |
| `fixed` | Quantidade fixa de transferência |
| `period` | Restrição de transferência baseada em período |
| `free` | Sem restrições de transferência |

---

## Entidades e Relacionamentos

```
ERC20ContractEntity (1) ←── (N) LoyaltiesEntity (1) ──→ (N) LoyaltiesRulesEntity
                                     │
                                     ├──→ (N) LoyaltiesDeferredEntity (transações)
                                     │           │
                                     │           └──→ Erc20ActionEntity (ação on-chain)
                                     │
                                     └──→ (N) LoyaltiesStakingRulesEntity
```

**Conceitos-chave:**
- Um **Contrato ERC20** é o contrato de token on-chain (name, symbol, chainId). Deve ser implantado (published) antes que operações de fidelidade possam executar on-chain.
- Uma **Fidelidade** (programa) vincula um tenant (`companyId`) a um contrato ERC20 e define como tokens são emitidos e transferidos, além de configurações de visualização de pagamento (equivalência pontos-para-moeda).
- **Regras de Fidelidade** definem a lógica de recompensa para um programa de fidelidade. Cada regra tem um tipo (add, multiply, split, cashback), uma prioridade e um escopo de whitelist opcional.
- **Transações Diferidas** (`LoyaltiesDeferredEntity`) rastreiam todos os movimentos de pontos (pendentes, executados, revertidos). Registram endereços de/para, quantidades, tempos de execução e metadados.
- **Regras de Staking** definem distribuições periódicas de recompensa baseadas em holdings de tokens, com requisitos configuráveis (titulares de coleção, titulares ERC20) e quantidades (fixas ou holder-factor).

---

## Formato de Expressão de Data

Vários campos usam um formato de "expressão de data" para agendar quando pontos ficam disponíveis. Suporta:

- **Duração estilo ms:** `1s`, `30m`, `1h`, `7d`, `30d` (string de milissegundos parseada pela biblioteca `ms`)
- **Duração com hora alvo:** `7d:hour:10` (7 dias a partir de agora, ajustado para 10:00)
- **Expressão cron:** Cron padrão para agendamentos recorrentes (usado em regras de staking)

Exemplos: `1s` (imediato), `24h`, `7d`, `30d:hour:0` (30 dias, meia-noite).

---

## Endpoints: Admin de Fidelidade (Programas e Regras)

Caminho base: `/{companyId}/loyalties/admin`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/` | Bearer | SuperAdmin, Admin | Criar novo programa de fidelidade |
| 2 | GET | `/` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Listar todos os programas (paginado) |
| 3 | GET | `/:loyaltyId` | Bearer | SuperAdmin, Admin | Obter programa por ID |
| 4 | PATCH | `/:loyaltyId` | Bearer | SuperAdmin, Admin | Atualizar programa |
| 5 | POST | `/:loyaltyId/rules` | Bearer | SuperAdmin, Admin | Adicionar regra a um programa |
| 6 | GET | `/:loyaltyId/rules/:ruleId` | Bearer | SuperAdmin, Admin | Obter uma regra específica |
| 7 | PATCH | `/:loyaltyId/rules/:ruleId` | Bearer | SuperAdmin, Admin | Atualizar uma regra |
| 8 | DELETE | `/:loyaltyId/rules/:ruleId` | Bearer | SuperAdmin, Admin | Excluir uma regra (soft-delete) |

### POST /{companyId}/loyalties/admin

Criar um novo programa de fidelidade.

**Requisição Mínima:**
```json
{
  "name": "My Loyalty Program",
  "priority": 1,
  "tokenIssuanceMethod": "mint",
  "tokenTransferabilityMethod": "transfer",
  "pointPrecision": "integer",
  "paymentViewSettings": {
    "pointsEquivalent": { "currency": "BRL", "currencyValue": 10, "pointsValue": 1 }
  },
  "active": true
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição do programa |
| `contractId` | UUID | Não | ID do contrato ERC20. Deve ser um contrato published válido na mesma empresa. Se null, a fidelidade opera sem suporte on-chain. |
| `priority` | integer | Sim | Prioridade de ordenação (maior = processado primeiro) |
| `tokenIssuanceMethod` | TokenIssuanceMethod | Sim | Como tokens são criados para recompensas |
| `tokenIssuanceAddress` | string (ETH address) | Condicional | Obrigatório quando `tokenIssuanceMethod` é `transfer`. |
| `tokenTransferabilityMethod` | TokenTransferabilityMethod | Sim | Como tokens se movem durante pagamentos/gastos |
| `tokenTransferabilityAddress` | string (ETH address) | Condicional | Obrigatório quando `tokenTransferabilityMethod` é `transfer`. |
| `pointPrecision` | PointPrecisionOptions | Sim | Se pontos são inteiros ou decimais |
| `paymentViewSettings` | object | Sim | Define equivalência pontos-para-moeda |
| `image` | string (URL) | Não | URL de logo/imagem do programa de fidelidade |
| `active` | boolean | Sim | Se o programa está ativo |

### POST /{companyId}/loyalties/admin/:loyaltyId/rules

Adicionar uma regra a um programa de fidelidade. Suporta tipos: `add` (valor fixo), `multiply` (multiplicador), `split` (divisão entre pagadores) e `cashback` (cashback com comissão multinível).

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `value` | string | Sim | Valor da recompensa. Interpretação depende de `type`: quantidade fixa para `add`, multiplicador para `multiply`, porcentagem para `cashback` (ex: `"0.05"` = 5%). |
| `available` | string (expressão de data) | Sim | Quando pontos ficam disponíveis após ganho (ex: `"1s"` = imediato, `"7d"` = 7 dias) |
| `type` | LoyaltiesRuleType | Sim | Tipo da regra: `add`, `multiply`, `split`, `cashback` |
| `name` | string | Sim | Identificador da regra (slugificado, minúsculas). Usado para corresponder ações de webhook. |
| `priority` | integer | Sim | Prioridade de execução. Deve ser única dentro do mesmo programa (exceto prioridade 0). |
| `whitelistId` | UUID | Não | Restringir regra a usuários em uma whitelist específica. |
| `descriptionTemplate` | string | Não | Template Handlebars para descrições de transação. |
| `poolAddress` | string (ETH address) | Não | Endereço de carteira de pool para transações agrupadas |
| `cashbackConfigurations` | object | Não | Obrigatório para regras tipo `cashback`. |

**Erros:**

| Status | Erro | Causa |
|--------|------|-------|
| 400 | AlreadyExistRuleWithSamePriorityException | Outra regra com a mesma prioridade existe nesta fidelidade |
| 400 | AlreadyExistRuleWithSameWhitelistException | Outra regra com mesma whitelist e nome existe |
| 404 | WhitelistNotFoundException | `whitelistId` fornecido não encontrado no tenant |

---

## Endpoints: Contratos ERC20

Caminho base: `/{companyId}/erc20-contracts`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/` | Bearer | SuperAdmin, Admin | Criar novo contrato ERC20 (rascunho) |
| 2 | GET | `/` | Bearer | SuperAdmin, Admin | Listar todos os contratos ERC20 (paginado) |
| 3 | GET | `/:id` | Bearer | SuperAdmin, Admin | Obter contrato por ID |
| 4 | PATCH | `/:id` | Bearer | SuperAdmin, Admin | Atualizar contrato em rascunho |
| 5 | PATCH | `/:id/publish` | Bearer | SuperAdmin, Admin | Implantar contrato on-chain |
| 6 | GET | `/:id/estimate-gas` | Bearer | SuperAdmin, Admin | Estimar custo de gas para deploy |
| 7 | PATCH | `/:id/transfer-config` | Bearer | SuperAdmin, Admin | Atualizar configuração de transferência |
| 8 | PATCH | `/has-role` | Bearer | SuperAdmin, Admin, Integration | Verificar se endereço tem papel em um contrato |
| 9 | PATCH | `/grant-role` | Bearer | SuperAdmin, Admin, Integration | Conceder papel a um endereço |

### POST /{companyId}/erc20-contracts

Criar um novo contrato ERC20 em status de rascunho.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome do contrato. Auto-sanitizado (caracteres latinos, alfanumérico + espaços). |
| `symbol` | string | Sim | Símbolo do token (ex: `"LPT"`). Auto-sanitizado (apenas alfanumérico). |
| `chainId` | integer | Sim | ID da cadeia blockchain (ex: `137` para Polygon) |
| `type` | ERC20ContractType | Sim | Tipo do contrato: `classic`, `permissioned` ou `in_custody` |
| `initialAmount` | string | Não | Suprimento inicial de tokens para mintar no deploy |
| `initialOwner` | string (ETH address) | Não | Endereço para receber suprimento inicial |
| `isBurnable` | boolean | Não | Se tokens podem ser queimados. Padrão: `false`. |
| `isWithdrawable` | boolean | Não | Se tokens podem ser sacados para carteiras externas. Padrão: `false`. |
| `transferConfig` | array | Não | Regras de restrição de transferência. Padrão: `[]` (sem restrições). |

---

## Endpoints: Tokens ERC20 (Mint / Transfer / Burn)

Caminho base: `/{companyId}/erc20-tokens`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | PATCH | `/:id/mint` | Bearer | SuperAdmin, Admin | Mintar tokens para um endereço |
| 2 | PATCH | `/:id/transfer/admin` | Bearer | SuperAdmin, Admin | Transferência admin entre endereços |
| 3 | PATCH | `/:id/transfer/user` | Bearer | SuperAdmin, Admin, User | Transferência iniciada pelo usuário |
| 4 | PATCH | `/:id/burn` | Bearer | SuperAdmin, Admin, User | Queimar tokens de um endereço |
| 5 | GET | `/:id/history` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Histórico de transações de um contrato |
| 6 | GET | `/:id/history/action/:actionId` | Bearer | SuperAdmin, Admin | Detalhes de ação específica |
| 7 | GET | `/:id/history/operator/:operatorId` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Histórico por operador |
| 8 | GET | `/:id/history/:userId` | Bearer | SuperAdmin, Admin, User | Histórico por usuário |

### PATCH /{companyId}/erc20-tokens/:id/mint

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `to` | string (ETH address) | Sim | Endereço da carteira destinatária |
| `amount` | string | Sim | Quantidade de tokens a mintar |
| `metadata` | object | Não | Metadados arbitrários anexados à ação |
| `sendEmail` | boolean | Não | Enviar notificação por email ao destinatário. Padrão: `true`. |
| `available` | string (expressão de data) | Não | Quando tokens mintados ficam disponíveis (ex: `"1s"` = imediato) |

**Resposta (204):** Sem conteúdo.

---

## Endpoints: Usuários de Fidelidade (Saldos, Transações, Staking)

Caminho base: `/{companyId}/loyalties/users`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | GET | `/balance/:userId` | Bearer | SuperAdmin, Admin, User | Obter todos os saldos de fidelidade de um usuário |
| 2 | GET | `/balance/:userId/:loyaltyId` | Bearer | SuperAdmin, Admin, User | Obter saldo para fidelidade específica |
| 3 | GET | `/deferred` | Bearer | SuperAdmin, Admin, User | Listar transações diferidas (paginado) |
| 4 | GET | `/deferred/xlsx` | Bearer | SuperAdmin, Admin, User | Exportar transações diferidas como Excel |
| 5 | GET | `/deferred/:userId` | Bearer | SuperAdmin, Admin, User | Listar transações diferidas de um usuário |
| 6 | GET | `/:userId/cashback-multilevel-report/:loyaltyId` | Bearer | User | Obter relatório de cashback multinível |
| 7 | GET | `/:userId/staking/:loyaltyId` | Bearer | SuperAdmin, Admin, User | Listar detalhes de staking com resumo |
| 8 | PATCH | `/:userId/staking/:loyaltyId/redeem` | Bearer | SuperAdmin, Admin, User | Resgatar recompensas de staking disponíveis |

### GET /{companyId}/loyalties/users/balance/:userId

Obter todos os saldos de programas de fidelidade de um usuário.

**Resposta (200):**
```json
[
  {
    "balance": "1500",
    "loyaltyId": "loyalty-uuid",
    "contractId": "erc20-contract-uuid",
    "pointsPrecision": "integer",
    "image": "https://cdn.w3block.io/loyalty-logo.png",
    "name": "Loyalty Points",
    "symbol": "LPT"
  }
]
```

**Notas:**
- Usuários regulares podem ver apenas seu próprio saldo. Admins e LoyaltyOperators podem ver saldo de qualquer usuário.
- Resultados são cacheados por 10 segundos.

### GET /{companyId}/loyalties/users/deferred

Listar todas as transações diferidas com filtros extensivos. Suporta filtragem por loyaltyId, actionId, ruleId, action, status, datas, endereços de carteira e mais.

---

## Endpoints: Recompensas de Fidelidade (Pré-visualizações e Pagamentos)

Caminho base: `/{companyId}/loyalties/rewards`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | PATCH | `/cashback/preview` | Bearer | SuperAdmin, Admin, User, LoyaltyOperator | Pré-visualizar cashback para valor de compra |
| 2 | PATCH | `/payment/preview` | Bearer | SuperAdmin, Admin, User, LoyaltyOperator | Pré-visualizar pagamento com pontos (desconto) |
| 3 | PATCH | `/payment` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Executar pagamento com pontos |

### PATCH /{companyId}/loyalties/rewards/cashback/preview

Pré-visualizar quanto cashback um usuário ganharia para um determinado valor.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | ID do programa de fidelidade |
| `amount` | string | Sim | Valor da compra |
| `userId` | UUID | Sim | Usuário recebendo cashback |
| `action` | string | Sim | Nome da regra a corresponder (ex: `"cashback_multilevel"`) |
| `destinationWalletAddress` | ETH address | Não | Sobrescrever carteira de destino |

### PATCH /{companyId}/loyalties/rewards/payment

Executar pagamento baseado em pontos. Queima/transfere pontos do usuário e registra a transação.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | ID do programa de fidelidade |
| `amount` | string | Sim | Valor total da compra (string numérica positiva) |
| `points` | string | Sim | Pontos a gastar |
| `userId` | UUID | Sim | Usuário fazendo o pagamento |
| `userCode` | string | Sim | Código de verificação do usuário (para pagamentos em loja) |
| `description` | string | Não | Descrição da transação |

---

## Endpoints: Webhooks de Fidelidade (Integração)

Caminho base: `/{companyId}/loyalties/webhook`

Estes endpoints são projetados para integração serviço-a-serviço. A maioria requer o papel `Integration` do tenant.

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/deposit` | Bearer | SuperAdmin, Admin | Acionar depósito (distribuição de recompensa) |
| 2 | POST | `/cashback-multilevel` | Bearer | Integration | Acionar distribuição de cashback multinível |
| 3 | POST | `/rollback` | Bearer | Integration | Reverter transações diferidas |
| 4 | GET | `/rollback/:requestId` | Bearer | Integration | Verificar status do rollback |
| 5 | POST | `/split-payees` | Bearer | Integration | Distribuir recompensas entre múltiplos pagadores |

### POST /{companyId}/loyalties/webhook/deposit

Acionar depósito de recompensa para um usuário baseado em regras de fidelidade.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | ID do programa de fidelidade |
| `userId` | UUID | Sim | Usuário recebendo o depósito |
| `action` | string | Sim | Nome da regra a corresponder |
| `amount` | string | Não | Valor base para cálculo |
| `description` | string | Não | Descrição da transação |
| `metadata` | object | Não | Metadados arbitrários (passados para templates de descrição via Handlebars) |

### POST /{companyId}/loyalties/webhook/split-payees

Distribuir recompensas entre múltiplos pagadores com splits baseados em porcentagem.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `loyaltyId` | UUID | Sim | ID do programa de fidelidade |
| `total` | string | Sim | Quantidade total a dividir |
| `payees` | array | Sim | Lista de pagadores com sua parte |
| `payees[].walletAddress` | ETH address | Sim | Carteira do destinatário |
| `payees[].percent` | string | Sim | Porcentagem da parte (ex: `"0.60"` = 60%) |
| `payees[].metadata` | object | Sim | Metadados do pagador |
| `payees[].descriptionTemplate` | string | Não | Template Handlebars para descrição |
| `payees[].available` | string | Não | Expressão de data para disponibilidade |
| `payees[].sendEmail` | boolean | Não | Enviar notificação por email |

---

## Referência de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 404 | LoyaltyNotFoundException | ID de fidelidade não encontrado na empresa | Verificar ID da fidelidade e da empresa |
| 400 | AlreadyExistRuleWithSamePriorityException | Prioridade de regra duplicada | Usar um valor de prioridade diferente |
| 400 | AlreadyExistRuleWithSameWhitelistException | Combinação duplicada de whitelist + nome | Usar uma whitelist ou nome diferente |
| 404 | LoyaltyRuleNotFound | ID da regra não encontrado | Verificar ID da regra, fidelidade e empresa |
| 404 | WhitelistNotFoundException | Whitelist não encontrada no tenant | Criar a whitelist primeiro ou verificar o ID |
| 404 | ThereIsNoRuleForThisActionException | Nenhuma regra corresponde ao nome da ação | Verificar se o nome da ação corresponde ao campo `name` de uma regra |
| 404 | UserByCodeNotFoundException | Código do usuário não encontrado | Verificar código do usuário na empresa |
| 400 | UserNotMatchException | ID do usuário não corresponde ao código | Verificar se userId e userCode pertencem ao mesmo usuário |
| 400 | LoyaltiesNotHaveContractException | Fidelidade não tem contrato ERC20 | Atribuir um contrato ERC20 ao programa de fidelidade |
| 400 | InsufficientBalanceException | Usuário não tem pontos suficientes | Verificar saldo antes de tentar pagamento |
| 400 | DuplicateRequestException | Requisição duplicada detectada | Evitar enviar a mesma requisição duas vezes |
