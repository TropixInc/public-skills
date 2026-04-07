---
id: FLOW_LOYALTY_CONTRACT_SETUP
title: "Fidelidade - Configuração de Contrato e Programa"
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
  - contract
  - setup
depends_on: []
---

# Fidelidade - Configuração de Contrato e Programa

## Visão Geral

Este fluxo cobre a configuração completa de um sistema de fidelidade: criação de um contrato de token ERC-20, deploy on-chain, criação de um programa de fidelidade vinculado a esse contrato e configuração de regras de recompensa. Este é o fluxo fundamental -- todas as outras operações de fidelidade (mint, transferências, pagamentos, cashback) dependem desta configuração estar completa.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | Token de acesso JWT (role Admin ou SuperAdmin) | Autenticação do serviço ID |
| ID da chain blockchain | Chain de destino para deploy do contrato (ex.: `137` para Polygon) | Configuração da plataforma |

## Entidades e Relacionamentos

```
ERC20ContractEntity (1) ←── (N) LoyaltiesEntity (1) ──→ (N) LoyaltiesRulesEntity
      │                            │
      │                            └── Config do programa de fidelidade: método de emissão,
      │                                precisão de pontos, configurações de visualização de pagamento
      │
      └── Contrato de token: nome, símbolo, chainId, status (draft → published)
```

**Conceitos-chave:**
- Um **Contrato ERC-20** é o token on-chain. Ele começa com status `draft` e deve ser `published` (deployado) antes que tokens possam ser mintados ou transferidos.
- Um **Programa de Fidelidade** define como o token é usado: como pontos são emitidos (mint vs transferência), como são gastos (transferência vs queima) e sua equivalência em moeda.
- **Regras** definem a lógica automatizada de recompensas. Cada regra tem um tipo (`add`, `multiply`, `split`, `cashback`), uma prioridade e escopo opcional por whitelist.

---

## Fluxo: Criar e Deployar Contrato ERC-20

### Passo 1: Criar Contrato ERC-20 (Rascunho)

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/erc20-contracts` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned"
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "0",
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": []
}
```

**Resposta (201):**
```json
{
  "id": "erc20-contract-uuid",
  "companyId": "company-uuid",
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned",
  "status": "draft",
  "address": null,
  "operators": [],
  "roles": [],
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": [],
  "createdAt": "2026-04-01T10:00:00Z",
  "updatedAt": "2026-04-01T10:00:00Z"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome do token (sanitizado para alfanumérico + espaços) |
| `symbol` | string | Sim | Símbolo do token (sanitizado para alfanumérico) |
| `chainId` | integer | Sim | Blockchain de destino (137 = Polygon, 80001 = Mumbai) |
| `type` | enum | Sim | `classic`, `permissioned` ou `in_custody`. O frontend usa o enum `TypeERC20Contract`. |
| `initialAmount` | string | Não | Tokens a mintar no deploy |
| `initialOwner` | endereço ETH | Não | Endereço que receberá o supply inicial |
| `isBurnable` | boolean | Não | Permitir queima de tokens. Padrão: `false`. |
| `isWithdrawable` | boolean | Não | Permitir saque para carteiras externas. Padrão: `false`. |
| `transferConfig` | array | Não | Regras de restrição de transferência |

**Observações:**
- Os campos `name` e `symbol` são automaticamente sanitizados: acentos removidos, caracteres não alfanuméricos eliminados.
- O contrato é criado com status `draft`. Nenhuma interação on-chain acontece nesta etapa.
- A maioria dos casos de uso de fidelidade usa o tipo `permissioned`, que permite controle de acesso baseado em roles no contrato.

### Passo 2: Estimar Gas (Opcional)

Antes de publicar, estime o custo de gas do deploy.

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| GET | `/{companyId}/erc20-contracts/{contractId}/estimate-gas` | Bearer token (Admin) | -- |

**Resposta (200):**
```json
{
  "totalGas": 2500000,
  "totalGasPrice": {
    "fast": "50000000000",
    "proposed": "30000000000",
    "safe": "20000000000"
  }
}
```

**Observações:**
- Os preços de gas estão em wei. As faixas `fast`, `proposed` e `safe` representam diferentes prioridades de velocidade de confirmação.
- Os resultados são cacheados por 1 minuto.
- O frontend exibe isso no componente `NewERC20ContractGasController` antes do usuário confirmar o deploy.

### Passo 3: Publicar Contrato (Deploy On-Chain)

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| PATCH | `/{companyId}/erc20-contracts/{contractId}/publish` | Bearer token (Admin) | -- |

**Corpo da requisição:** Nenhum.

**Resposta (204):** Sem conteúdo.

**Observações:**
- Esta é uma operação assíncrona. O status do contrato transiciona: `draft` -> `publishing` -> `published` (ou `failed`).
- Após a publicação, o campo `address` do contrato é preenchido com o endereço on-chain deployado.
- Faça polling em `GET /{companyId}/erc20-contracts/{contractId}` para verificar quando o `status` se torna `published`.
- Uma vez publicado, o contrato não pode ser atualizado (exceto `transferConfig`).

---

## Fluxo: Criar Programa de Fidelidade

### Passo 4: Criar Programa de Fidelidade

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/loyalties/admin` | Bearer token (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "name": "My Loyalty Program",
  "priority": 1,
  "tokenIssuanceMethod": "mint",
  "tokenTransferabilityMethod": "transfer",
  "pointPrecision": "integer",
  "paymentViewSettings": {
    "pointsEquivalent": {
      "currency": "BRL",
      "currencyValue": 10,
      "pointsValue": 1
    }
  },
  "active": true
}
```

**Requisição Completa (com contrato ERC-20):**
```json
{
  "name": "My Loyalty Program",
  "contractId": "erc20-contract-uuid",
  "priority": 1,
  "tokenIssuanceMethod": "mint",
  "tokenTransferabilityMethod": "transfer",
  "pointPrecision": "integer",
  "paymentViewSettings": {
    "pointsEquivalent": {
      "currency": "BRL",
      "currencyValue": 10,
      "pointsValue": 1
    }
  },
  "image": "https://cdn.example.com/logo.png",
  "active": true
}
```

**Resposta (201):** Entidade completa de fidelidade com `id` gerado.

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição do programa |
| `contractId` | UUID | Não | Contrato ERC-20 publicado. Sem ele, operações on-chain não funcionarão. |
| `priority` | integer | Sim | Prioridade de ordenação do programa (maior = primeiro) |
| `tokenIssuanceMethod` | enum | Sim | `mint` (novos tokens) ou `transfer` (de endereço de origem) |
| `tokenIssuanceAddress` | endereço ETH | Condicional | Obrigatório quando `tokenIssuanceMethod = "transfer"` |
| `tokenTransferabilityMethod` | enum | Sim | `transfer` (mover tokens) ou `burn` (destruir tokens) |
| `tokenTransferabilityAddress` | endereço ETH | Condicional | Obrigatório quando `tokenTransferabilityMethod = "transfer"` |
| `pointPrecision` | enum | Sim | `integer` ou `decimal` |
| `paymentViewSettings.pointsEquivalent.currency` | string | Sim | Código da moeda (ex.: `"BRL"`) |
| `paymentViewSettings.pointsEquivalent.currencyValue` | number | Sim | Pontos por 1 unidade de moeda (ex.: 10 = 10 pontos por 1 BRL) |
| `paymentViewSettings.pointsEquivalent.pointsValue` | number | Sim | Valor em moeda por 1 ponto (ex.: 0.1 = 1 ponto vale 0.1 BRL) |
| `image` | string URL | Não | URL do logo do programa |
| `active` | boolean | Sim | Habilitar/desabilitar o programa |

**Observações:**
- O `contractId` deve referenciar um contrato ERC-20 pertencente ao mesmo `companyId`. Se você tentar usar um contrato de outra empresa, um erro será lançado.
- Você pode criar um programa de fidelidade sem um contrato e vincular um depois via PATCH, mas nenhuma operação on-chain funcionará até que um contrato esteja vinculado.
- `paymentViewSettings` é crítico para o fluxo de pagamento -- determina como pontos são convertidos de/para valores em moeda.

---

## Fluxo: Configurar Regras de Recompensa

### Passo 5: Adicionar Regra ao Programa de Fidelidade

**Endpoint:**

| Método | Caminho | Autenticação | Content-Type |
|--------|---------|--------------|-------------|
| POST | `/{companyId}/loyalties/admin/{loyaltyId}/rules` | Bearer token (Admin) | application/json |

**Requisição Mínima (regra simples de adição):**
```json
{
  "value": "100",
  "available": "1s",
  "type": "add",
  "name": "welcome-bonus",
  "priority": 1
}
```

**Requisição Completa (cashback com comissões multinível):**
```json
{
  "value": "0.05",
  "available": "7d",
  "type": "cashback",
  "name": "cashback_multilevel",
  "description": "5% cashback with 3-level commission",
  "priority": 1,
  "descriptionTemplate": "Cashback: {{value}} points from order {{orderId}}",
  "cashbackConfigurations": {
    "indirectCashback": [0.03, 0.02, 0.01],
    "indirectCashbackAvailable": "7d",
    "indirectCashbackDescriptionTemplate": "L{{level}} commission from {{buyerName}}",
    "indirectCashbackSendEmail": true,
    "finalRecipientRate": 0.01,
    "finalRecipientAvailable": "1s",
    "finalRecipientSendEmail": false,
    "remainingPointsAvailable": "1s",
    "remainingPointsSendEmail": false
  }
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `value` | string | Sim | Valor da regra. O significado depende do tipo: quantidade fixa para `add`, multiplicador para `multiply`, porcentagem para `cashback` (ex.: `"0.05"` = 5%). |
| `available` | string | Sim | Expressão de data para quando os pontos se tornam disponíveis. `"1s"` = imediato, `"7d"` = 7 dias. |
| `type` | enum | Sim | `add`, `multiply`, `split` ou `cashback` |
| `name` | string | Sim | Identificador da regra (slugificado). Usado para corresponder ações de webhook. |
| `priority` | integer | Sim | Único dentro da fidelidade (exceto 0). Determina a ordem de processamento. |
| `description` | string | Não | Descrição legível por humanos |
| `descriptionTemplate` | string | Não | Template Handlebars para descrições dinâmicas |
| `whitelistId` | UUID | Não | Escopo da regra para uma whitelist |
| `poolAddress` | endereço ETH | Não | Endereço do pool para transações agrupadas |
| `cashbackConfigurations` | object | Não | Obrigatório para tipo `cashback`. Consulte a Referência da API para o schema completo. |

**Observações:**
- O campo `name` é automaticamente slugificado (minúsculas, caracteres especiais substituídos por hífens).
- A prioridade deve ser única por programa de fidelidade. A prioridade 0 está isenta da restrição de unicidade.
- Ao usar um `whitelistId`, a whitelist deve existir no mesmo tenant. A combinação de `loyaltyId + whitelistId + name` também deve ser única.
- `cashbackConfigurations.indirectCashback` é um array de taxas por nível de indicação. `[0.03, 0.02, 0.01]` significa 3% para N1, 2% para N2, 1% para N3.
- Templates de descrição usam sintaxe Handlebars. As variáveis disponíveis dependem dos metadados do webhook passados durante a distribuição de recompensas.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 404 | LoyaltyNotFoundException | ID de fidelidade não encontrado na empresa | Verifique o ID de fidelidade e a empresa |
| 400 | AlreadyExistRuleWithSamePriorityException | Prioridade duplicada nas regras | Use um valor de prioridade único |
| 400 | AlreadyExistRuleWithSameWhitelistException | Whitelist+nome duplicado | Altere a whitelist ou o nome da regra |
| 404 | WhitelistNotFoundException | Whitelist não encontrada | Crie a whitelist primeiro |
| 400 | Erro de validação na expressão de data | Formato `available` inválido | Use string compatível com `ms` como `1s`, `7d` |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Fidelidade criada sem `contractId`, então mint/transferência falha | Sempre crie e publique um contrato ERC-20 primeiro, depois vincule-o ao programa de fidelidade |
| 2 | Contrato ainda com status `draft` ou `publishing` ao tentar usar | Faça polling do status do contrato até que ele alcance `published` antes de tentar operações |
| 3 | Colisão de prioridade de regra causando erro 400 | Verifique as regras existentes antes de adicionar. A prioridade 0 está isenta das verificações de unicidade. |
| 4 | `name` e `symbol` contendo caracteres acentuados são silenciosamente modificados | A API remove acentos e caracteres não alfanuméricos. Use nomes apenas com ASCII. |
| 5 | `cashbackConfigurations` ignorado para tipos de regra que não são cashback | Apenas regras do tipo `cashback` processam configurações de cashback. Outros tipos ignoram o campo. |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Mint e Transferência de Tokens | Próximo passo após a configuração | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) |
| Recompensas e Pagamentos | Requer regras configuradas | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) |
| Autenticação (Sign-in) | Obrigatório antes de qualquer chamada de API | Módulo de autenticação |
