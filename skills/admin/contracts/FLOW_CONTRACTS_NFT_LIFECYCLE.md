---
id: FLOW_CONTRACTS_NFT_LIFECYCLE
title: "Contratos - Ciclo de Vida do Contrato NFT"
module: offpix/contracts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - nft
  - erc721
  - blockchain
  - deploy
  - royalty
depends_on:
  - CONTRACTS_API_REFERENCE
---

# Ciclo de Vida do Contrato NFT

## Visão Geral

Um contrato NFT na W3Block é um smart contract ERC721A implantado em uma blockchain suportada (Polygon, Ethereum, Moonbeam). Este fluxo abrange o ciclo de vida completo: criar um rascunho com configuração de royalty, opcionalmente definir features e whitelists, estimar gas e publicar na blockchain. No frontend, isso é gerenciado em **Tokens > Contracts** (`/dash/tokens/contracts`).

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da empresa | Fluxo de auth / configuração do ambiente |
| Carteira blockchain | Endereço da carteira proprietária para deploy | Configurações da empresa |

## Máquina de Estado

```
DRAFT ──[publish]──→ PUBLISHING ──[blockchain confirms]──→ PUBLISHED
  ↑                                       │
  └──────────────[failed]─────────────────┘
                  FAILED ──[retry publish]──→ PUBLISHING
```

- **DRAFT:** Totalmente editável — nome, símbolo, royalty, features, whitelists
- **PUBLISHING:** Transação blockchain enviada, aguardando confirmação
- **PUBLISHED:** Ativo on-chain — contrato possui endereço, pode criar coleções
- **FAILED:** Deploy falhou — pode editar e tentar novamente

---

## Fluxo: Criar e Fazer Deploy de um Contrato NFT

### Passo 1: Criar Rascunho do Contrato

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/{companyId}/contracts` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "name": "My Collection",
  "symbol": "MYC",
  "chainId": 137,
  "participants": []
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "name": "Digital Art Collection",
  "symbol": "DAC",
  "chainId": 137,
  "description": "Limited edition digital artworks",
  "image": "https://cdn.example.com/contract-cover.png",
  "externalLink": "https://example.com/collection",
  "participants": [
    {
      "name": "Artist",
      "payee": "0xArtistWalletAddress...",
      "share": 5.0
    },
    {
      "name": "Platform Fee",
      "payee": "0xPlatformWalletAddress...",
      "share": 2.5
    }
  ],
  "features": [
    "admin:minter",
    "admin:burner",
    "admin:mover",
    "user:mover"
  ],
  "transferWhitelistId": "whitelist-uuid",
  "minterWhitelistId": "whitelist-uuid",
  "maxSupply": "10000"
}
```

**Resposta (201):** Objeto completo do contrato com `status: "draft"`, objeto `royalty` aninhado com `fee` calculado (soma de todos os shares).

**Observações:**
- O `name` é latinizado (caracteres especiais removidos) e o `symbol` é convertido para maiúsculas automaticamente
- Um `RoyaltyContract` é criado atomicamente com o contrato NFT
- `participants` definem a divisão de royalties — cada um recebe sua porcentagem `share` em vendas secundárias
- O `fee` total de royalty é calculado como a soma dos shares de todos os participantes
- `maxSupply` com valor `null` ou `0` significa minting ilimitado

### Passo 2 (Opcional): Atualizar Rascunho

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| PATCH | `/{companyId}/contracts/{contractId}` | Bearer (Admin) |

Mesmo corpo da criação. Funciona apenas com status `draft` ou `failed`.

### Passo 3: Estimar Gas

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/contracts/{contractId}/estimate-gas` | Bearer (Admin) |

**Resposta (200):**
```json
{
  "totalGas": 5000000,
  "totalGasPrice": {
    "fast": "25000000000",
    "proposed": "20000000000",
    "safe": "15000000000"
  }
}
```

O frontend exibe três níveis de gas: Fast, Proposed (padrão) e Safe — cada um com diferentes trade-offs de velocidade/custo.

### Passo 4: Publicar na Blockchain

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| PATCH | `/{companyId}/contracts/{contractId}/publish` | Bearer (Admin) |

Sem corpo na requisição. **Resposta:** 204 No Content.

**O que acontece internamente:**
1. O status muda para `publishing`
2. Uma `ContractAction` é criada com tipo `FACTORY_ERC721A`
3. O processador blockchain faz o deploy do contrato
4. Em caso de sucesso: status → `published`, contrato recebe seu `address`
5. Em caso de falha: status → `failed`, pode tentar novamente

**Após a publicação:** O contrato possui um `address` on-chain e pode ser usado para criar coleções de tokens.

---

## Fluxo: Gerenciar Participantes de Royalty

A configuração de royalty faz parte do contrato. Cada participante recebe uma parcela dos rendimentos de vendas secundárias.

O frontend mostra um **gráfico de pizza** (ShareChart) visualizando a divisão de royalties.

**Labels do frontend vs API:**

| Frontend | Campo da API | Observações |
|----------|-------------|-------------|
| Nome do Participante | `participants[].name` | Nome de exibição |
| Endereço da Carteira | `participants[].payee` | Endereço Ethereum |
| % de Participação | `participants[].share` | Porcentagem de 0.01–10.0 |
| Taxa Total | `royalty.fee` | Calculado (somente leitura) |

**Participantes podem ser vinculados a:**
- Usuários internos (via `userId` em RoyaltyEligible)
- Contatos externos (via `externalContactId` em RoyaltyEligible)

Para gerenciar entidades elegíveis a royalty, use os endpoints `/contracts/royalty-eligible` (veja [Referência da API](./CONTRACTS_API_REFERENCE.md)).

---

## Fluxo: Gerenciamento de Roles

Após a publicação, você pode conceder roles de blockchain a endereços de carteira.

### Verificar se Endereço Possui Role

```
PATCH /{companyId}/contracts/has-role
```

```json
{
  "contractId": "contract-uuid",
  "address": "0xWalletAddress...",
  "role": "minter"
}
```

### Conceder Role

```
PATCH /{companyId}/contracts/grant-role
```

Mesmo corpo. Adiciona o endereço aos `operators` do contrato com a role especificada.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | BadRequestException | Dados ou status do contrato inválidos | Verifique os campos obrigatórios e o status atual |
| 400 | Contrato não está em draft/failed | Tentando atualizar/publicar um contrato publicado | Contratos publicados não podem ser modificados |
| 404 | NotFoundException | Contrato não encontrado | Verifique o contractId e o companyId |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Publicação falha silenciosamente | Verifique o campo `contractAction.status` — pode mostrar `FAILED` com detalhes do erro |
| 2 | Não é possível editar contrato publicado | Contratos publicados são imutáveis on-chain. Crie um novo |
| 3 | Estimativa de gas retorna 0 | Certifique-se de que o contrato possui configuração válida (name, symbol, chainId) |
| 4 | Shares de royalty não somam corretamente | O `share` de cada participante é independente (0.01–10.0). O `fee` é a soma |
| 5 | Whitelist não aplicada | Whitelists são aplicadas apenas on-chain. Devem ser configuradas antes da publicação |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Gerenciamento de Coleções | Criar coleções sob um contrato publicado | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Operações de Token | Mint, transferência, burn de tokens | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Ciclo de Vida ERC20 | Contratos de tokens fungíveis | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Whitelist | Configurar restrições de acesso | [FLOW_WHITELIST_MANAGEMENT](../whitelist/FLOW_WHITELIST_MANAGEMENT.md) |
