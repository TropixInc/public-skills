---
id: FLOW_CONTRACTS_NFT_LIFECYCLE
title: "Contracts - NFT Contract Lifecycle"
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

# NFT Contract Lifecycle

## Overview

An NFT contract in W3Block is an ERC721A smart contract deployed to a supported blockchain (Polygon, Ethereum, Moonbeam). This flow covers the full lifecycle: create a draft with royalty configuration, optionally set features and whitelists, estimate gas, and publish to the blockchain. In the frontend, this is managed under **Tokens > Contracts** (`/dash/tokens/contracts`).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Blockchain wallet | Owner wallet address for deployment | Company settings |

## State Machine

```
DRAFT ──[publish]──→ PUBLISHING ──[blockchain confirms]──→ PUBLISHED
  ↑                                       │
  └──────────────[failed]─────────────────┘
                  FAILED ──[retry publish]──→ PUBLISHING
```

- **DRAFT:** Fully editable — name, symbol, royalty, features, whitelists
- **PUBLISHING:** Blockchain transaction submitted, waiting for confirmation
- **PUBLISHED:** Live on-chain — contract has an address, can create collections
- **FAILED:** Deployment failed — can edit and retry

---

## Flow: Create and Deploy an NFT Contract

### Step 1: Create Contract Draft

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/contracts` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "My Collection",
  "symbol": "MYC",
  "chainId": 137,
  "participants": []
}
```

**Complete Request (production example):**
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

**Response (201):** Full contract object with `status: "draft"`, nested `royalty` object with calculated `fee` (sum of all shares).

**Notes:**
- The `name` is latinized (special characters removed) and `symbol` is uppercased automatically
- A `RoyaltyContract` is created atomically with the NFT contract
- `participants` define royalty splits — each receives their `share` percentage on secondary sales
- The total royalty `fee` is calculated as the sum of all participant shares
- `maxSupply` of `null` or `0` means unlimited minting

### Step 2 (Optional): Update Draft

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/contracts/{contractId}` | Bearer (Admin) |

Same body as create. Only works on `draft` or `failed` status.

### Step 3: Estimate Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/contracts/{contractId}/estimate-gas` | Bearer (Admin) |

**Response (200):**
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

The frontend displays three gas tiers: Fast, Proposed (default), and Safe — each with different speed/cost tradeoffs.

### Step 4: Publish to Blockchain

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/contracts/{contractId}/publish` | Bearer (Admin) |

No request body. **Response:** 204 No Content.

**What happens internally:**
1. Status changes to `publishing`
2. A `ContractAction` is created with type `FACTORY_ERC721A`
3. The blockchain processor deploys the contract
4. On success: status → `published`, contract gets its `address`
5. On failure: status → `failed`, can retry

**After publishing:** The contract has an on-chain `address` and can be used to create token collections.

---

## Flow: Manage Royalty Participants

Royalty configuration is part of the contract. Each participant gets a share of secondary sale proceeds.

The frontend shows a **pie chart** (ShareChart) visualizing the royalty split.

**Frontend labels vs API:**

| Frontend | API Field | Notes |
|----------|-----------|-------|
| Participant Name | `participants[].name` | Display name |
| Wallet Address | `participants[].payee` | Ethereum address |
| Share % | `participants[].share` | 0.01–10.0 percentage |
| Total Fee | `royalty.fee` | Calculated (read-only) |

**Participants can be linked to:**
- Internal users (via `userId` in RoyaltyEligible)
- External contacts (via `externalContactId` in RoyaltyEligible)

To manage royalty eligible entities, use the `/contracts/royalty-eligible` endpoints (see [API Reference](./CONTRACTS_API_REFERENCE.md)).

---

## Flow: Role Management

After publishing, you can grant blockchain roles to wallet addresses.

### Check if Address Has Role

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

### Grant Role

```
PATCH /{companyId}/contracts/grant-role
```

Same body. Adds the address to the contract's `operators` with the specified role.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | BadRequestException | Invalid contract data or status | Check required fields and current status |
| 400 | Contract not in draft/failed | Trying to update/publish a published contract | Published contracts cannot be modified |
| 404 | NotFoundException | Contract not found | Verify contractId and companyId |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Publish fails silently | Check the `contractAction.status` field — it may show `FAILED` with error details |
| 2 | Can't edit published contract | Published contracts are immutable on-chain. Create a new one |
| 3 | Gas estimation returns 0 | Ensure the contract has valid configuration (name, symbol, chainId) |
| 4 | Royalty shares don't add up | Each participant's `share` is independent (0.01–10.0). The `fee` is the sum |
| 5 | Whitelist not applied | Whitelists are only enforced on-chain. They must be configured before publishing |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Collection Management | Create collections under a published contract | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Token Operations | Mint, transfer, burn tokens | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| ERC20 Lifecycle | Fungible token contracts | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Whitelist | Configure access restrictions | [FLOW_WHITELIST_MANAGEMENT](../whitelist/FLOW_WHITELIST_MANAGEMENT.md) |
