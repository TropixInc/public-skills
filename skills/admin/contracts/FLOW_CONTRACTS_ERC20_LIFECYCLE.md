---
id: FLOW_CONTRACTS_ERC20_LIFECYCLE
title: "Contratos - Ciclo de Vida do Token ERC20"
module: offpix/contracts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - erc20
  - fungible
  - loyalty
  - transfer-rules
  - mint
  - burn
depends_on:
  - CONTRACTS_API_REFERENCE
---

# Ciclo de Vida do Token ERC20

## Visão Geral

Contratos ERC20 na W3Block representam tokens fungíveis — usados para pontos de fidelidade, moedas in-app e tokens customizados. Este fluxo abrange a criação, deploy e operação de contratos ERC20, incluindo o sofisticado sistema de handlers de transferência para contratos permissionados. No frontend, o ERC20 é gerenciado em **Loyalty > Contracts** (`/dash/loyalty/contracts`).

A W3Block suporta três tipos de ERC20:
- **Classic** — ERC20 padrão, transferências diretas na blockchain
- **Permissioned** — transferências passam por uma cadeia de regras (taxas, limites, períodos)
- **In Custody** — empresa mantém os tokens em nome dos usuários

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da empresa | Fluxo de auth / configuração do ambiente |
| Carteira da empresa | Endereço padrão do proprietário | Configurações da empresa |

## Máquina de Estado

Mesma dos contratos NFT:

```
DRAFT ──[publish]──→ PUBLISHING ──[confirms]──→ PUBLISHED
  ↑                                    │
  └──────────[failed]──────────────────┘
              FAILED ──[retry]──→ PUBLISHING
```

---

## Fluxo: Criar e Fazer Deploy de Contrato ERC20

### Passo 1: Criar Rascunho

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/{companyId}/erc20-contracts` | Bearer (Admin) | application/json |

**Requisição Mínima (classic):**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "classic"
}
```

**Requisição Completa (permissioned com regras de transferência):**
```json
{
  "name": "Platform Token",
  "symbol": "PTK",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "1000000",
  "initialOwner": "0xCompanyWallet...",
  "isBurnable": true,
  "isWithdrawable": false,
  "maxSupply": "10000000",
  "transferConfig": [
    {
      "model": "PERCENTAGE",
      "value": "0.05"
    },
    {
      "model": "MAX_LIMIT",
      "value": "100"
    },
    {
      "model": "PERIOD",
      "value": "10",
      "period": "7d"
    }
  ]
}
```

**Resposta (201):** Objeto completo do contrato ERC20 com `status: "draft"`.

### Passo 2: Estimar Gas

```
GET /{companyId}/erc20-contracts/{contractId}/estimate-gas
```

### Passo 3: Publicar

```
PATCH /{companyId}/erc20-contracts/{contractId}/publish
```

**Resposta:** 204 No Content. O contrato é implantado de forma assíncrona.

---

## Regras de Transferência (Permissioned & In Custody)

Quando um usuário chama o endpoint `/transfer/user` em um contrato permissioned ou in-custody, a transferência passa por uma **cadeia de handlers** que avalia as regras em ordem:

```
Request → FreeTransferHandler → MaxLimitTransferHandler → PeriodTransferHandler
            → PercentageTransferHandler → FixedTransferHandler → Execute
```

Cada handler processa a transferência (se sua regra corresponder) ou passa para o próximo. Se nenhum handler aceitar, a transferência é rejeitada.

### Regra: FREE

**Config:** `{ "model": "FREE", "value": "free" }`

| Política | Comportamento |
|----------|---------------|
| `free` | Transferência permitida sem restrições |
| `forbidden` | Todas as transferências de usuário são bloqueadas |
| `erc20ReceiverOnly` | Apenas endereços com role `keyErc20Receiver` podem receber |

### Regra: MAX_LIMIT

**Config:** `{ "model": "MAX_LIMIT", "value": "100" }`

Limita o número total de transferências que um usuário pode fazer. Conta todas as transferências bem-sucedidas (não falhas) do usuário.

### Regra: PERIOD

**Config:** `{ "model": "PERIOD", "value": "10", "period": "7d" }`

Limita transferências dentro de uma janela de tempo. Value = máximo de transferências, period = janela de tempo (ex.: "7d", "24h", "30d").

### Regra: PERCENTAGE

**Config:** `{ "model": "PERCENTAGE", "value": "0.05" }`

Deduz uma taxa percentual de cada transferência. Cria **duas transações na blockchain**:

1. Destinatário recebe: `amount * (1 - percentage)`
2. Empresa recebe: `amount * percentage` (enviado para `company.defaultOwnerAddress`)

**Exemplo:** Transferir 1000 tokens com taxa de 5% → destinatário recebe 950, empresa recebe 50.

### Regra: FIXED

**Config:** `{ "model": "FIXED", "value": "10" }`

Deduz um valor fixo de cada transferência. Aplica-se apenas se o valor da transferência > taxa fixa. Cria **duas transações na blockchain**:

1. Destinatário recebe: `amount - fixed`
2. Empresa recebe: valor `fixed`

**Exemplo:** Transferir 1000 tokens com taxa fixa de 10 → destinatário recebe 990, empresa recebe 10.

### Múltiplas Regras

As regras podem ser combinadas. Elas são avaliadas na ordem da cadeia — a primeira regra correspondente processa a transferência:

```json
"transferConfig": [
  { "model": "MAX_LIMIT", "value": "100" },
  { "model": "PERIOD", "value": "10", "period": "7d" },
  { "model": "PERCENTAGE", "value": "0.05" }
]
```

Isso significa: máximo de 100 transferências no total, máximo de 10 por semana, com taxa de 5% em cada.

### Atualizar Regras de Transferência (Pós-Deploy)

```
PATCH /{companyId}/erc20-contracts/{contractId}/transfer-config
```

```json
{
  "transferConfig": [
    { "model": "PERCENTAGE", "value": "0.03" }
  ]
}
```

As regras de transferência podem ser atualizadas mesmo em contratos publicados.

---

## Fluxo: Fazer Mint de Tokens ERC20

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/mint` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "to": "0xRecipientWallet...",
  "amount": "5000",
  "metadata": {
    "reason": "Welcome bonus",
    "campaign": "onboarding-q1"
  },
  "sendEmail": true,
  "available": "1s"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `to` | string | Sim | Endereço da carteira do destinatário |
| `amount` | string | Sim | Quantidade a mintar |
| `metadata` | object | Não | Metadados customizados da transação |
| `sendEmail` | boolean | Não | Enviar notificação (padrão: true) |
| `available` | string | Não | Atraso de disponibilidade (ex.: "1s", "24h") |

---

## Fluxo: Transferir Tokens ERC20

### Transferência Admin (Ignora Regras)

```
PATCH /{companyId}/erc20-tokens/{contractId}/transfer/admin
```

```json
{
  "from": "0xSenderWallet...",
  "to": "0xRecipientWallet...",
  "amount": "1000",
  "metadata": { "reason": "Manual adjustment" },
  "sendEmail": true
}
```

Transferências admin ignoram todas as regras de configuração de transferência — são executadas diretamente na blockchain.

### Transferência de Usuário (Regras Aplicadas)

```
PATCH /{companyId}/erc20-tokens/{contractId}/transfer/user
```

Mesmo corpo. Para contratos `permissioned` e `in_custody`, a cadeia de handlers de transferência é avaliada. Para contratos `classic`, a execução é direta.

---

## Fluxo: Fazer Burn de Tokens ERC20

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/burn` | Bearer (Admin, User) | application/json |

**Requisição:**
```json
{
  "from": "0xBurnerWallet...",
  "amount": "500",
  "metadata": { "reason": "Redemption for reward" },
  "sendEmail": true
}
```

Requer `isBurnable: true` no contrato.

---

## Fluxo: Visualizar Histórico de Transações

### Todas as Transações

```
GET /{companyId}/erc20-tokens/{contractId}/history
```

Histórico paginado de todas as ações (mint, transferência, burn). Cache de 10 segundos.

### Ação Específica

```
GET /{companyId}/erc20-tokens/{contractId}/history/action/{actionId}
```

### Transações do Usuário

```
GET /{companyId}/erc20-tokens/{contractId}/history/{userId}
```

Usuários podem ver apenas seu próprio histórico. Admins podem ver o histórico de qualquer usuário.

### Atividade do Operador

```
GET /{companyId}/erc20-tokens/{contractId}/history/operator/{operatorId}
```

Para a role LoyaltyOperator — ver todas as operações realizadas por um operador específico.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | YouCannotTransferThisTokenException | Regras de transferência impedem a operação | Verifique a política FREE, limites, período |
| 400 | Contrato não publicado | Tentando operar em contrato draft/failed | Publique o contrato primeiro |
| 400 | Não é queimável | Tentando fazer burn em contrato não queimável | Defina `isBurnable: true` na criação |
| 403 | Permissões insuficientes | Usuário não pode realizar esta operação | Verifique as roles e políticas de transferência |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Transferência de usuário rejeitada | Verifique quais regras de configuração de transferência estão ativas. A cadeia avalia em ordem |
| 2 | Taxa percentual cria transação extra | Isso é por design — a taxa vai para a carteira da empresa como uma ação separada na blockchain |
| 3 | Limite de transferência atingido | MAX_LIMIT conta todas as transferências bem-sucedidas. Limites baseados em período são resetados após a janela |
| 4 | Atraso `available` confuso | Tokens são mintados imediatamente, mas podem não ser utilizáveis até o atraso expirar |
| 5 | Transferência admin vs usuário | Transferência admin sempre ignora as regras. Use `/transfer/user` para aplicar a configuração de transferência |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Contratos NFT | Ciclo de vida de tokens não fungíveis | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Operações de Token | Mint/transferência/burn de NFT | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Referência da API | Detalhes completos dos endpoints | [CONTRACTS_API_REFERENCE](./CONTRACTS_API_REFERENCE.md) |
