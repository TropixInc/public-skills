---
id: FLOW_LOYALTY_BALANCE_OPERATIONS
title: "Fidelidade - Operações de Saldo (Mint, Transferência, Queima)"
module: offpix/loyalty
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - loyalty
  - erc20
  - mint
  - transfer
  - burn
  - balance
depends_on:
  - FLOW_LOYALTY_CONTRACT_SETUP
---

# Fidelidade - Operações de Saldo (Mint, Transferência, Queima)

## Visão Geral

Este fluxo cobre as operações diretas de tokens ERC-20: mintar novos tokens, transferir tokens entre carteiras, queimar tokens e verificar saldos. Estas são as operações administrativas principais para gerenciar manualmente saldos de pontos de fidelidade, independente do sistema automatizado de recompensas/cashback. O frontend agrupa estas operações na seção "Fidelidade" com modais dedicados para cada ação.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | Token de acesso JWT (role Admin para mint/transferência de admin) | Autenticação do serviço ID |
| Contrato ERC-20 publicado | Contrato com status `published` | [Fluxo de Configuração de Contrato](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Programa de fidelidade (para consultas de saldo) | Fidelidade ativa com contrato vinculado | [Fluxo de Configuração de Contrato](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Endereço de carteira | Endereço ETH do destinatário/origem | Carteira do usuário no serviço ID |

## Entidades e Relacionamentos

```
ERC20ContractEntity ──→ Erc20ActionEntity (ações on-chain: mint, transferência, queima)
                              │
LoyaltiesEntity ──→ LoyaltiesDeferredEntity (registros de transação)
      │
      └── Referencia ERC20ContractEntity via contractId
```

**Conceitos-chave:**
- **Mint** cria novos tokens e os envia para um endereço de carteira. Apenas Admins podem mintar.
- **Transferência (Admin)** move tokens de uma carteira para outra. Requer role Admin.
- **Transferência (Usuário)** permite que usuários transfiram seus próprios tokens. Sujeita às restrições de `transferConfig` no contrato.
- **Queima** destrói tokens de uma carteira. Disponível para Admins e o proprietário do token.
- **Saldo** é consultado através do endpoint de loyalty users, que agrega o saldo on-chain com metadados do programa de fidelidade.

---

## Fluxo: Mintar Tokens

### Passo 1: Encontrar Endereço de Carteira do Usuário

Antes de mintar, você precisa do endereço de carteira do destinatário. O frontend usa o hook `useGetUserByWallet` para buscar usuários por endereço de carteira, ou você pode obtê-lo do perfil do usuário.

Para encontrar um usuário por endereço de carteira (via SDK do serviço ID):

```
GET /tenants/{companyId}/users?address={walletAddress}&includeOwnerInfo=true
```

### Passo 2: Mintar Tokens

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/mint` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "to": "0x1234567890abcdef1234567890abcdef12345678",
  "amount": "100"
}
```

**Requisição Completa (exemplo de produção do frontend):**
```json
{
  "to": "0x1234567890abcdef1234567890abcdef12345678",
  "amount": "500",
  "metadata": {
    "description": "Manual bonus points for loyalty program"
  },
  "sendEmail": false,
  "available": "1s"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `to` | string (endereço ETH) | Sim | Endereço da carteira do destinatário |
| `amount` | string | Sim | Número de tokens a mintar (string para lidar com números grandes) |
| `metadata` | object | Não | Metadados arbitrários. O frontend envia `{ description: "..." }`. |
| `sendEmail` | boolean | Não | Enviar email de notificação. O frontend usa `false` por padrão. |
| `available` | string (expressão de data) | Não | Quando os tokens se tornam disponíveis. Comportamento padrão: imediato. |

**Resposta (204):** Sem conteúdo.

**Observações:**
- O `contractId` na URL é o UUID do contrato ERC-20 (não o endereço on-chain).
- O contrato deve estar com status `published`.
- Mint é uma operação on-chain e pode levar alguns segundos para confirmar.
- Se o contrato tiver um `maxSupply` definido e este mint exceder o limite, a operação falhará.
- O componente `MintModal` do frontend coleta o endereço `to` (via busca de carteira), `amount` e `description` opcional.

---

## Fluxo: Transferir Tokens (Admin)

### Passo 3: Transferência de Admin

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/transfer/admin` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "from": "0xSourceWallet...",
  "to": "0xDestinationWallet...",
  "amount": "50"
}
```

**Requisição Completa (exemplo de produção do frontend):**
```json
{
  "from": "0xSourceWallet...",
  "to": "0xDestinationWallet...",
  "amount": "50",
  "metadata": {
    "description": "Transfer between user wallets"
  },
  "sendEmail": false,
  "available": "1s"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `from` | string (endereço ETH) | Sim | Endereço da carteira de origem |
| `to` | string (endereço ETH) | Sim | Endereço da carteira de destino |
| `amount` | string | Sim | Quantidade a transferir |
| `metadata` | object | Não | Metadados arbitrários |
| `sendEmail` | boolean | Não | Enviar email de notificação. O frontend usa `false` por padrão. |
| `available` | string (expressão de data) | Não | Quando os tokens transferidos se tornam disponíveis. O frontend envia `"1s"`. |

**Resposta (204):** Sem conteúdo.

**Observações:**
- Transferências de admin ignoram restrições de configuração de transferência.
- A carteira de origem deve ter saldo suficiente.
- O componente `TransferModal` do frontend coleta `from`, `to` (via busca de carteira), `amount` e `description` opcional.

---

## Fluxo: Transferir Tokens (Usuário)

### Passo 4: Transferência de Usuário

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/transfer/user` | Bearer token (User) | application/json |

O corpo da requisição é idêntico ao da transferência de admin. Porém:
- O endereço `from` deve ser a própria carteira do usuário autenticado.
- Restrições de configuração de transferência no contrato são aplicadas (ex.: `max_limit`, `percentage`, `period`).
- Usuários só podem transferir seus próprios tokens.

---

## Fluxo: Queimar Tokens

### Passo 5: Queimar Tokens

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/burn` | Bearer token (Admin ou User) | application/json |

**Requisição:**
```json
{
  "from": "0xWalletAddress...",
  "amount": "25",
  "metadata": {
    "description": "Burn excess tokens"
  },
  "sendEmail": false
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `from` | string (endereço ETH) | Sim | Carteira da qual queimar tokens |
| `amount` | string | Sim | Quantidade a queimar |
| `metadata` | object | Não | Metadados arbitrários |
| `sendEmail` | boolean | Não | Enviar email de notificação |
| `available` | string | Não | Expressão de data para quando a queima entra em efeito |

**Resposta (204):** Sem conteúdo.

**Observações:**
- O contrato ERC-20 deve ter `isBurnable: true`.
- Usuários só podem queimar seus próprios tokens.

---

## Fluxo: Verificar Saldos

### Passo 6a: Obter Todos os Saldos de Fidelidade de um Usuário

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/loyalties/users/balance/{userId}` | Bearer token | -- |

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

**Observações:**
- Retorna saldos para TODOS os programas de fidelidade ativos dos quais o usuário participa.
- Cache de 10 segundos. Após mintar/transferir, aguarde brevemente antes de verificar o saldo atualizado.
- Usuários regulares só podem consultar seu próprio saldo. Admins e LoyaltyOperators podem consultar qualquer usuário.

### Passo 6b: Obter Saldo para uma Fidelidade Específica

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/loyalties/users/balance/{userId}/{loyaltyId}` | Bearer token | -- |

**Resposta (200):** Objeto de saldo único (mesmo formato acima, mas não encapsulado em um array).

**Observações:**
- Cache de 5 segundos (menor que o endpoint de todos os saldos).
- Útil para verificações de saldo em tempo real em fluxos de pagamento.

---

## Fluxo: Visualizar Histórico de Transações

### Passo 7a: Histórico de Transações do Contrato

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/erc20-tokens/{contractId}/history` | Bearer token (Admin, LoyaltyOperator) | -- |

Retorna lista paginada de todas as ações on-chain (mints, transferências, queimas) para um contrato.

### Passo 7b: Histórico de Transações do Usuário

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/erc20-tokens/{contractId}/history/{userId}` | Bearer token | -- |

Retorna lista paginada de ações on-chain para um usuário específico em um contrato.

### Passo 7c: Obter Detalhes de uma Ação Específica

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/erc20-tokens/{contractId}/history/action/{actionId}` | Bearer token (Admin) | -- |

Retorna detalhes de uma única ação on-chain.

### Passo 7d: Transações Diferidas (Camada de Fidelidade)

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/loyalties/users/deferred` | Bearer token | -- |

Retorna transações diferidas paginadas com filtragem extensiva. Consulte a [Referência da API](./LOYALTY_API_REFERENCE.md) para a lista completa de parâmetros de query.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 403 | ForbiddenException | Usuário tentando acessar dados de outro usuário | Usuários só podem acessar seu próprio saldo/histórico |
| 400 | Contrato não publicado | Tentativa de operações em um contrato em rascunho | Publique o contrato primeiro |
| 400 | Saldo insuficiente | Valor de transferência/queima excede o saldo | Verifique o saldo antes de operar |
| 400 | Violação de configuração de transferência | Transferência de usuário excede limites configurados | Verifique as regras de `transferConfig` do contrato |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Saldo não atualiza imediatamente após mint | Saldos são cacheados (5-10 segundos). Aguarde ou implemente polling no cliente. |
| 2 | Usando endereço on-chain em vez do UUID do contrato na URL | Os endpoints da API usam o UUID do contrato, não o endereço blockchain deployado |
| 3 | Mintando para ID de usuário em vez de endereço de carteira | O campo `to` requer um endereço Ethereum, não um UUID de usuário. Busque a carteira do usuário primeiro. |
| 4 | Frontend envia `sendEmail: false` mas emails ainda são enviados | O campo `sendEmail` tem como padrão `true` no backend. Sempre defina explicitamente como `false` se não quiser emails. |
| 5 | Transferência de usuário falha sem erro claro | Verifique o array `transferConfig` do contrato -- restrições como `max_limit` ou `period` podem estar bloqueando a transferência |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Configuração de Contrato e Programa | Deve ser feito primeiro | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Recompensas e Pagamentos | Distribuição automatizada de recompensas | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) |
