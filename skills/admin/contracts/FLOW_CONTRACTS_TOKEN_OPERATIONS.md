---
id: FLOW_CONTRACTS_TOKEN_OPERATIONS
title: "Contratos - Operações de Token (Mint, Transferência, Burn)"
module: offpix/contracts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - tokens
  - mint
  - transfer
  - burn
  - gas
  - rfid
depends_on:
  - CONTRACTS_API_REFERENCE
  - FLOW_CONTRACTS_COLLECTION_MANAGEMENT
---

# Operações de Token (Mint, Transferência, Burn)

## Visão Geral

Uma vez que uma coleção é publicada, edições individuais de tokens podem ser mintadas sob demanda, transferidas entre carteiras ou queimadas (burn). Todas as operações blockchain requerem estimativa de gas primeiro e são processadas de forma assíncrona. No frontend, essas ações aparecem na página de detalhes da edição (`/dash/tokens/collections/{collectionId}/editions/{editionId}`) como botões de ação (QR Code, Certificado, Transferir, Queimar).

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da empresa | Fluxo de auth / configuração do ambiente |
| Coleção publicada | Com edições no status `readyToMint` ou `minted` | [Gerenciamento de Coleções](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |

---

## Fluxo: Fazer Mint de Tokens Sob Demanda

Faz mint de edições específicas que estão no status `readyToMint`.

### Passo 1: Estimar Gas

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/token-editions/{editionId}/estimate-gas/mint` | Bearer (Admin) |

**Query:** `?quantityToMint=5` (1–500)

**Resposta (200):**
```json
{
  "totalGas": 250000,
  "totalGasPrice": {
    "fast": "25000000000",
    "proposed": "20000000000",
    "safe": "15000000000"
  }
}
```

### Passo 2: Fazer Mint

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/token-editions/mint-on-demand` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "editionNumbers": [1, 2, 3, 4, 5],
  "collectionId": "collection-uuid",
  "ownerAddress": "0xOwnerWallet..."
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `editionNumbers` | number[] | Sim | Números das edições para mint |
| `collectionId` | UUID | Sim | Coleção pai |
| `ownerAddress` | string | Sim | Endereço da carteira do destinatário |

**Resposta:** 204 No Content

**Transição de estado:** `readyToMint` → `minting` → (assíncrono) → `minted`

Cada edição mintada com sucesso recebe:
- `tokenId` — ID do token on-chain
- `mintedHash` — hash da transação blockchain
- `mintedAt` — timestamp do mint
- `contractAddress` — endereço on-chain do contrato

---

## Fluxo: Transferir Tokens

Transfere tokens mintados entre endereços de carteira.

### Passo 1: Estimar Gas

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/token-editions/{editionId}/estimate-gas/transfer` | Bearer (Admin) |

**Query:** `?toAddress=0xRecipient...`

### Passo 2: Transferir (Único)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/token-editions/{editionId}/transfer-token` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "toAddress": "0xRecipientWallet...",
  "editionId": "edition-uuid"
}
```

### Passo 2 (Alternativo): Transferir (Lote)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/token-editions/transfer-token` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "toAddress": "0xRecipientWallet...",
  "editionId": ["edition-uuid-1", "edition-uuid-2", "edition-uuid-3"]
}
```

**Transição de estado:** `minted` → `transferring` → (assíncrono) → `transferred` ou `transferFailure`

### Passo 3: Consultar Status da Transferência

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/token-editions/{editionId}/get-last/transfer` | Bearer (Admin) |

Retorna o status da última ação de transferência para a edição. O frontend faz polling neste endpoint para atualizar a UI quando a transferência é confirmada.

---

## Fluxo: Fazer Burn de Tokens

Destrói tokens mintados permanentemente.

### Passo 1: Estimar Gas

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/token-editions/{editionId}/estimate-gas/burn` | Bearer (Admin) |

### Passo 2: Fazer Burn

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| DELETE | `/{companyId}/token-editions/burn` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "tokens": ["edition-uuid-1", "edition-uuid-2"]
}
```

**Transição de estado:** `minted` → `burning` → (assíncrono) → `burned` ou `burnFailure`

---

## Fluxo: Bloquear/Desbloquear para Compra

Edições podem ser bloqueadas quando um usuário inicia uma compra, impedindo operações concorrentes.

### Bloquear

```
PATCH /{companyId}/token-editions/locked-for-buy
```

```json
{
  "editionIds": ["edition-uuid-1"]
}
```

Estado: `draft` ou `readyToMint` → `lockedForBuy`

### Desbloquear

```
PATCH /{companyId}/token-editions/unlocked-for-buy
```

Mesmo corpo. Estado: `lockedForBuy` → estado anterior.

---

## Fluxo: Registrar Tokens Mintados Externamente

Para tokens mintados fora da W3Block que precisam ser rastreados no sistema.

```
PATCH /{companyId}/token-editions/notify-externally-minted
```

```json
{
  "editionIds": ["edition-uuid"],
  "tokenId": "on-chain-token-id",
  "mintedHash": "0xTransactionHash...",
  "contractAddress": "0xContractAddress..."
}
```

---

## Metadados Públicos & Certificados

### Obter Metadados do Token por RFID

```
GET /metadata/rfid/{rfid}
```

Retorna metadados do token para itens físicos etiquetados com RFID. Não requer autenticação.

### Obter Metadados do Token por Endereço na Chain

```
GET /metadata/address/{contractAddress}/{chainId}/{tokenId}
```

Retorna metadados do token on-chain. Não requer autenticação.

### Listar NFTs por Carteira

```
GET /metadata/nfts/{walletAddress}/{chainId}
```

Retorna todos os NFTs de propriedade de um endereço de carteira em uma chain específica.

### Gerar Certificado PDF

```
GET /certification/{contractAddress}/{chainId}/{tokenId}
```

Retorna um certificado PDF para o token. Usa o template de certificado da coleção.

### Pré-visualizar Certificado

```
POST /certification/preview
```

```json
{
  "pdfTemplate": "<html>...</html>"
}
```

Retorna um blob PDF para pré-visualização. Requer autenticação.

---

## Resumo de Estimativa de Gas

Todas as estimativas de gas retornam a mesma estrutura e possuem cache de 1 minuto:

```json
{
  "totalGas": 250000,
  "totalGasPrice": {
    "fast": "25000000000",
    "proposed": "20000000000",
    "safe": "15000000000"
  }
}
```

| Operação | Endpoint |
|----------|----------|
| Deploy de contrato NFT | `GET /contracts/{id}/estimate-gas` |
| Deploy de contrato ERC20 | `GET /erc20-contracts/{id}/estimate-gas` |
| Publicar coleção | `GET /token-collections/estimate-gas?contractId=...&initialQuantityToMint=...` |
| Mint de edições | `GET /token-editions/{id}/estimate-gas/mint?quantityToMint=...` |
| Transferir edição | `GET /token-editions/{id}/estimate-gas/transfer?toAddress=...` |
| Burn de edição | `GET /token-editions/{id}/estimate-gas/burn` |

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Transição de status inválida | Edição não está no estado correto para a operação | Verifique o status da edição primeiro |
| 400 | YouCannotTransferThisTokenException | Regras de transferência impedem a operação | Verifique as features e whitelists do contrato |
| 404 | NotFoundException | Edição não encontrada | Verifique o editionId e o companyId |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Mint falha sem gas | Certifique-se de que a carteira da empresa possui fundos suficientes na chain de destino |
| 2 | Transferência para endereço errado | Transferências blockchain são irreversíveis. Sempre verifique o endereço do destinatário |
| 3 | Transferência em lote falha parcialmente | Cada edição é processada independentemente. Algumas podem ter sucesso enquanto outras falham. Verifique os status individuais |
| 4 | Edição presa em `minting`/`transferring` | Congestionamento na blockchain pode atrasar a confirmação. Faça polling no endpoint de status |
| 5 | Falha no burn | Certifique-se de que o contrato possui a feature `admin:burner` ou o usuário tem permissão `user:burner` |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Gerenciamento de Coleções | Coleções devem ser publicadas primeiro | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Contrato NFT | Contrato deve estar implantado | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Operações ERC20 | Operações de tokens fungíveis | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
