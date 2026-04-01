---
id: FLOW_CONTRACTS_TOKEN_OPERATIONS
title: "Contracts - Token Operations (Mint, Transfer, Burn)"
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

# Token Operations (Mint, Transfer, Burn)

## Overview

Once a collection is published, individual token editions can be minted on-demand, transferred between wallets, or burned. All blockchain operations require gas estimation first and are processed asynchronously. In the frontend, these actions appear in the edition detail page (`/dash/tokens/collections/{collectionId}/editions/{editionId}`) as action buttons (QR Code, Certificate, Transfer, Burn).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Published collection | With editions in `readyToMint` or `minted` status | [Collection Management](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |

---

## Flow: Mint Tokens On-Demand

Mint specific editions that are in `readyToMint` status.

### Step 1: Estimate Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{editionId}/estimate-gas/mint` | Bearer (Admin) |

**Query:** `?quantityToMint=5` (1–500)

**Response (200):**
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

### Step 2: Mint

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/token-editions/mint-on-demand` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "editionNumbers": [1, 2, 3, 4, 5],
  "collectionId": "collection-uuid",
  "ownerAddress": "0xOwnerWallet..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `editionNumbers` | number[] | Yes | Edition numbers to mint |
| `collectionId` | UUID | Yes | Parent collection |
| `ownerAddress` | string | Yes | Recipient wallet address |

**Response:** 204 No Content

**State transition:** `readyToMint` → `minting` → (async) → `minted`

Each successfully minted edition receives:
- `tokenId` — on-chain token ID
- `mintedHash` — blockchain transaction hash
- `mintedAt` — mint timestamp
- `contractAddress` — contract's on-chain address

---

## Flow: Transfer Tokens

Transfer minted tokens between wallet addresses.

### Step 1: Estimate Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{editionId}/estimate-gas/transfer` | Bearer (Admin) |

**Query:** `?toAddress=0xRecipient...`

### Step 2: Transfer (Single)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/token-editions/{editionId}/transfer-token` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "toAddress": "0xRecipientWallet...",
  "editionId": "edition-uuid"
}
```

### Step 2 (Alternative): Transfer (Batch)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/token-editions/transfer-token` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "toAddress": "0xRecipientWallet...",
  "editionId": ["edition-uuid-1", "edition-uuid-2", "edition-uuid-3"]
}
```

**State transition:** `minted` → `transferring` → (async) → `transferred` or `transferFailure`

### Step 3: Poll Transfer Status

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{editionId}/get-last/transfer` | Bearer (Admin) |

Returns the latest transfer action status for the edition. The frontend polls this endpoint to update the UI when the transfer confirms.

---

## Flow: Burn Tokens

Destroy minted tokens permanently.

### Step 1: Estimate Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{editionId}/estimate-gas/burn` | Bearer (Admin) |

### Step 2: Burn

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| DELETE | `/{companyId}/token-editions/burn` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "tokens": ["edition-uuid-1", "edition-uuid-2"]
}
```

**State transition:** `minted` → `burning` → (async) → `burned` or `burnFailure`

---

## Flow: Lock/Unlock for Purchase

Editions can be locked when a user initiates a purchase, preventing concurrent operations.

### Lock

```
PATCH /{companyId}/token-editions/locked-for-buy
```

```json
{
  "editionIds": ["edition-uuid-1"]
}
```

State: `draft` or `readyToMint` → `lockedForBuy`

### Unlock

```
PATCH /{companyId}/token-editions/unlocked-for-buy
```

Same body. State: `lockedForBuy` → previous state.

---

## Flow: Register Externally Minted Tokens

For tokens minted outside W3Block that need to be tracked in the system.

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

## Public Metadata & Certificates

### Get Token Metadata by RFID

```
GET /metadata/rfid/{rfid}
```

Returns token metadata for physical items tagged with RFID. No authentication required.

### Get Token Metadata by Chain Address

```
GET /metadata/address/{contractAddress}/{chainId}/{tokenId}
```

Returns on-chain token metadata. No authentication required.

### List NFTs by Wallet

```
GET /metadata/nfts/{walletAddress}/{chainId}
```

Returns all NFTs owned by a wallet address on a specific chain.

### Generate PDF Certificate

```
GET /certification/{contractAddress}/{chainId}/{tokenId}
```

Returns a PDF certificate for the token. Uses the collection's certificate template.

### Preview Certificate

```
POST /certification/preview
```

```json
{
  "pdfTemplate": "<html>...</html>"
}
```

Returns a PDF blob for preview. Requires authentication.

---

## Gas Estimation Summary

All gas estimations return the same structure and are cached for 1 minute:

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

| Operation | Endpoint |
|-----------|----------|
| Deploy NFT contract | `GET /contracts/{id}/estimate-gas` |
| Deploy ERC20 contract | `GET /erc20-contracts/{id}/estimate-gas` |
| Publish collection | `GET /token-collections/estimate-gas?contractId=...&initialQuantityToMint=...` |
| Mint editions | `GET /token-editions/{id}/estimate-gas/mint?quantityToMint=...` |
| Transfer edition | `GET /token-editions/{id}/estimate-gas/transfer?toAddress=...` |
| Burn edition | `GET /token-editions/{id}/estimate-gas/burn` |

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Invalid status transition | Edition not in correct state for operation | Check edition status first |
| 400 | YouCannotTransferThisTokenException | Transfer rules prevent operation | Check contract features and whitelists |
| 404 | NotFoundException | Edition not found | Verify editionId and companyId |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Mint fails with no gas | Ensure the company wallet has sufficient funds on the target chain |
| 2 | Transfer to wrong address | Blockchain transfers are irreversible. Always verify the recipient address |
| 3 | Batch transfer partially fails | Each edition is processed independently. Some may succeed while others fail. Check individual statuses |
| 4 | Edition stuck in `minting`/`transferring` | Blockchain congestion can delay confirmation. Poll the status endpoint |
| 5 | Burn failure | Ensure the contract has `admin:burner` feature or the user has `user:burner` permission |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Collection Management | Collections must be published first | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| NFT Contract | Contract must be deployed | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| ERC20 Operations | Fungible token operations | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
