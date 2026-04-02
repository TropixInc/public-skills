---
id: CONTRACTS_API_REFERENCE
title: "Contracts & Tokens - API Reference"
module: offpix/contracts
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - tokens
  - nft
  - erc20
  - collections
  - editions
  - royalty
  - blockchain
  - api-reference
---

# Contracts & Tokens API Reference

Complete endpoint reference for the W3Block Contracts & Tokens module. Covers NFT (ERC721A) contracts, token collections and editions, ERC20 fungible tokens, royalty management, gas estimation, and blockchain operations. All served by the Pixway Registry backend.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://api.w3block.io` |
| Staging | *(staging environment available — use staging base URL)* |
| Swagger | https://api.w3block.io/docs/ |
| PDF Certificates | `https://pdf.w3block.io` |

## Authentication

All endpoints require Bearer token unless noted:

```
Authorization: Bearer {accessToken}
```

All endpoints are scoped to a company via `{companyId}` path parameter.

---

## Enums

### ContractStatus (NFT & ERC20)

| Value | Description |
|-------|-------------|
| `draft` | Initial state, editable |
| `publishing` | Blockchain deployment in progress |
| `published` | Live on blockchain |
| `failed` | Deployment failed (can retry) |

### NftContractFeature

| Value | Description |
|-------|-------------|
| `admin:minter` | Admin can mint new tokens |
| `admin:burner` | Admin can burn tokens |
| `admin:mover` | Admin can transfer tokens |
| `user:burner` | Users can burn their own tokens |
| `user:mover` | Users can transfer their own tokens |

### ContractOperatorRole

| Value | Description |
|-------|-------------|
| `minter` | Can mint new tokens |
| `mover` | Can transfer tokens |

### ERC20ContractType

| Value | Frontend Label | Description |
|-------|---------------|-------------|
| `classic` | Classic | Standard ERC20 — direct blockchain transfers |
| `permissioned` | Permissioned | Transfer rules enforced (fee, limits, period) |
| `in_custody` | In Custody | Company holds tokens on behalf of users |

### TokenCollectionStatus

| Value | Description |
|-------|-------------|
| `draft` | Editable, not yet published |
| `published` | Published to blockchain |

### TokenEditionStatusEnum

| Value | Frontend Label | Description |
|-------|---------------|-------------|
| `draft` | Draft | Initial state |
| `draftError` | Draft Error | Data validation error |
| `importError` | Import Error | XLS import validation failed |
| `readyToMint` | Ready to Mint | Validated, awaiting publish |
| `lockedForBuy` | Locked for Buy | Reserved for purchase |
| `minting` | Minting | Blockchain deployment in progress |
| `minted` | Minted | Successfully on blockchain |
| `burning` | Burning | Burn in progress |
| `burned` | Burned | Destroyed |
| `burnFailure` | Burn Failure | Burn transaction failed |
| `transferring` | Transferring | Transfer in progress |
| `transferred` | Transferred | Successfully transferred |
| `transferFailure` | Transfer Failure | Transfer transaction failed |

### ERC20TransferConfigModel

| Value | Description |
|-------|-------------|
| `FREE` | No restrictions (or forbidden/erc20ReceiverOnly policy) |
| `FIXED` | Fixed fee deducted per transfer |
| `PERCENTAGE` | Percentage fee deducted per transfer |
| `PERIOD` | Max transfers within a time window |
| `MAX_LIMIT` | Max total transfers per user |

### ChainId (Supported Blockchains)

| Value | Network |
|-------|---------|
| `1` | Ethereum Mainnet |
| `137` | Polygon Mainnet |
| `80001` | Polygon Mumbai (testnet) |
| `1284` | Moonbeam |
| `1337` | Localhost (dev) |

---

## Entities & Relationships

```
NftContract (1) ──→ (1) RoyaltyContract ──→ (N) RoyaltyEligible
     │                                              │
     │                                              └── ExternalContact or User
     │
     └──→ (N) TokenCollection ──→ (N) TokenEdition
               │                        │
               │ name, quantity,         │ editionNumber, status,
               │ tokenData template,     │ ownerAddress, rfid,
               │ subcategory             │ tokenId, mintedHash
               │
               └── Subcategory (template definition)

ERC20Contract (standalone)
     │
     └──→ transferConfig[] (transfer rules)
     └──→ (N) ERC20TransferUserAction (audit trail)
```

---

## Endpoints: NFT Contracts

Base path: `/{companyId}/contracts`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create NFT contract draft |
| 2 | GET | `/` | Bearer | Admin | List company's NFT contracts |
| 3 | GET | `/:id` | Bearer | Admin | Get contract details |
| 4 | PATCH | `/:id` | Bearer | Admin | Update draft contract |
| 5 | PATCH | `/:id/publish` | Bearer | Admin | Deploy to blockchain |
| 6 | GET | `/:id/estimate-gas` | Bearer | Admin | Estimate deployment gas (cached 1 min) |
| 7 | PATCH | `/has-role` | Bearer | Admin, Integration | Check if address has role |
| 8 | PATCH | `/grant-role` | Bearer | Admin, Integration | Grant role to address |

### POST /{companyId}/contracts

Create a new NFT contract draft with royalty configuration.

**Minimal Request:**
```json
{
  "name": "My NFT Collection",
  "symbol": "MNC",
  "chainId": 137,
  "participants": []
}
```

**Complete Request (production example):**
```json
{
  "name": "My NFT Collection",
  "symbol": "MNC",
  "chainId": 137,
  "description": "A digital art collection",
  "image": "https://example.com/contract-image.png",
  "externalLink": "https://example.com",
  "participants": [
    {
      "name": "Artist",
      "payee": "0x1234567890abcdef1234567890abcdef12345678",
      "share": 5.0
    },
    {
      "name": "Platform",
      "payee": "0xabcdef1234567890abcdef1234567890abcdef12",
      "share": 2.5
    }
  ],
  "features": ["admin:minter", "admin:burner", "user:mover"],
  "transferWhitelistId": "whitelist-uuid",
  "minterWhitelistId": "whitelist-uuid",
  "maxSupply": "10000"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Contract name (latinized, alphanumeric) |
| `symbol` | string | Yes | — | Token symbol (uppercase, alphanumeric) |
| `chainId` | ChainId | Yes | — | Target blockchain network |
| `description` | string | No | — | Contract description |
| `image` | URL | No | — | Contract image |
| `externalLink` | URL | No | — | External link |
| `participants` | array | Yes | `[]` | Royalty recipients (see below) |
| `features` | NftContractFeature[] | No | — | Enabled capabilities |
| `transferWhitelistId` | UUID | No | — | Whitelist for transfer restrictions |
| `minterWhitelistId` | UUID | No | — | Whitelist for mint restrictions |
| `maxSupply` | string (bigint) | No | `null` | Max mintable tokens (null = unlimited) |

**Participant object:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Participant display name |
| `payee` | string | Yes | Ethereum address for royalty payments |
| `share` | number | Yes | Royalty percentage (0.01–10.0) |
| `contactId` | UUID | No | Link to RoyaltyEligible record |

**Response (201):**
```json
{
  "id": "contract-uuid",
  "companyId": "company-uuid",
  "name": "My NFT Collection",
  "symbol": "MNC",
  "chainId": 137,
  "address": null,
  "status": "draft",
  "features": ["admin:minter", "admin:burner", "user:mover"],
  "maxSupply": "10000",
  "royalty": {
    "id": "royalty-uuid",
    "fee": 7.5,
    "participants": [
      { "name": "Artist", "payee": "0x1234...", "share": 5.0 },
      { "name": "Platform", "payee": "0xabcd...", "share": 2.5 }
    ],
    "status": "draft"
  },
  "operators": [],
  "contractAction": null,
  "createdAt": "2026-04-01T10:00:00Z"
}
```

### PATCH /{companyId}/contracts/:id/publish

Deploys the contract to the blockchain. Only works on `draft` or `failed` status.

**Response:** 204 No Content

**State transition:** `draft` → `publishing` → (async) → `published` or `failed`

The deployment creates a `ContractAction` entity that tracks the blockchain transaction. The actual deployment is processed asynchronously — poll the contract status or use webhooks.

### GET /{companyId}/contracts/:id/estimate-gas

Returns gas estimation for deploying the contract. Cached for 1 minute.

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

---

## Endpoints: Token Collections

Base path: `/{companyId}/token-collections`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create collection draft |
| 2 | GET | `/` | Bearer | Admin | List collections (paginated, filterable) |
| 3 | GET | `/:id` | Bearer | Admin | Get collection details |
| 4 | PUT | `/:id` | Bearer | Admin | Update draft collection |
| 5 | PATCH | `/publish/:id` | Bearer | Admin | Publish collection to blockchain |
| 6 | PATCH | `/:id/sync-draft-tokens` | Bearer | Admin | Create draft editions (async) |
| 7 | PATCH | `/:id/increase-editions` | Bearer | Integration | Increase edition quantity (async) |
| 8 | GET | `/estimate-gas` | Bearer | Admin | Estimate publish gas (cached 1 min) |
| 9 | PATCH | `/:id/pass/enable` | Bearer | Integration | Enable token pass |
| 10 | PATCH | `/:id/pass/disable` | Bearer | Integration | Disable token pass |
| 11 | DELETE | `/:id/burn` | Bearer | Admin | Burn entire collection |
| 12 | DELETE | `/:id/draft` | Bearer | Admin | Delete draft collection |
| 13 | GET | `/:id/export/xlsx` | Bearer | Admin | Export editions as XLSX |
| 14 | POST | `/:id/import/xlsx` | Bearer | Admin | Import editions from XLSX (async) |

### POST /{companyId}/token-collections

**Minimal Request:**
```json
{
  "subcategoryId": "subcategory-uuid",
  "name": "Genesis Collection"
}
```

**Complete Request:**
```json
{
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "Genesis Collection",
  "description": "First edition collectibles",
  "mainImage": "https://example.com/collection.png",
  "tokenData": {
    "artist": "John Doe",
    "year": 2026,
    "medium": "Digital Art"
  },
  "quantity": 100,
  "rangeInitialToMint": "1-100",
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "similarTokens": true,
  "settings": {
    "sendWebhookWhenTokenEditionIsUpdated": true
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `contractId` | UUID | No | — | Associated NFT contract |
| `subcategoryId` | UUID | Yes | — | Collection template/subcategory |
| `name` | string | Yes | — | Collection name |
| `description` | string | No | — | Description |
| `mainImage` | URL | No | — | Cover image |
| `tokenData` | object | No | `{}` | Custom metadata fields (defined by template) |
| `quantity` | integer | No | `0` | Number of editions |
| `rangeInitialToMint` | string | No | — | Range notation for initial mints (e.g., "1-100") |
| `ownerAddress` | string | No | — | Ethereum address for minted tokens |
| `similarTokens` | boolean | No | `true` | Whether editions share same metadata |
| `settings` | object | No | — | Collection settings |

### PATCH /{companyId}/token-collections/publish/:id

Publishes the collection — validates all editions, mints tokens marked as `readyToMint`.

**Response (201):**
```json
{
  "tokenCollection": { "..." : "..." },
  "validationErrors": []
}
```

If validation fails, returns detailed errors per field using `PublishValidationsEnum`.

### PATCH /{companyId}/token-collections/:id/sync-draft-tokens

Creates draft token editions for the collection. Async operation via Bull queue.

**Response (201):**
```json
{
  "jobId": "job-uuid"
}
```

### GET/POST /{companyId}/token-collections/:id/export/xlsx / import/xlsx

- **Export:** Downloads XLSX template with current edition data
- **Import:** Upload XLSX with edition data (FormData, max 10MB). Returns `{ jobId }` for async processing

---

## Endpoints: Token Editions

Base path: `/{companyId}/token-editions`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | GET | `/` | Bearer | Admin | List editions (paginated, filterable) |
| 2 | GET | `/:id` | Bearer | Admin | Get edition details |
| 3 | GET | `/xls` | Bearer | Admin | Request XLS report |
| 4 | GET | `/check-rfid` | Bearer | Admin | Check RFID availability |
| 5 | PATCH | `/ready-to-mint` | Bearer | Admin | Mark editions as ready to mint |
| 6 | PATCH | `/mint-on-demand` | Bearer | Admin, Integration, Application | Mint specific editions |
| 7 | PATCH | `/locked-for-buy` | Bearer | Admin, Integration, Application | Lock editions for purchase |
| 8 | PATCH | `/unlocked-for-buy` | Bearer | Admin, Integration, Application | Unlock editions |
| 9 | PATCH | `/notify-externally-minted` | Bearer | Admin, Integration, Application | Register external mint |
| 10 | PATCH | `/update-token-metadata` | Bearer | Admin, Integration, Application | Update edition metadata |
| 11 | PATCH | `/transfer-token` | Bearer | Admin | Transfer tokens (bulk) |
| 12 | PATCH | `/:id/transfer-token` | Bearer | Admin | Transfer single token |
| 13 | DELETE | `/burn` | Bearer | Admin | Burn tokens |
| 14 | PATCH | `/:id/rfid` | Bearer | Admin | Assign/update RFID |
| 15 | GET | `/:id/estimate-gas/mint` | Bearer | Admin | Estimate mint gas |
| 16 | GET | `/:id/estimate-gas/transfer` | Bearer | Admin | Estimate transfer gas |
| 17 | GET | `/:id/estimate-gas/burn` | Bearer | Admin | Estimate burn gas |
| 18 | GET | `/:id/get-last/transfer` | Bearer | Admin | Get last transfer status |

### PATCH /{companyId}/token-editions/ready-to-mint

Mark editions as ready to mint (transition from `draft` to `readyToMint`).

**Request:**
```json
{
  "editionId": ["edition-uuid-1", "edition-uuid-2"]
}
```

### PATCH /{companyId}/token-editions/mint-on-demand

Mint specific editions immediately.

**Request:**
```json
{
  "editionNumbers": [1, 2, 3],
  "collectionId": "collection-uuid",
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678"
}
```

### PATCH /{companyId}/token-editions/transfer-token

Transfer tokens between wallets.

**Request:**
```json
{
  "toAddress": "0xrecipient...",
  "editionId": ["edition-uuid-1", "edition-uuid-2"]
}
```

### DELETE /{companyId}/token-editions/burn

Burn (destroy) token editions.

**Request:**
```json
{
  "tokens": ["edition-uuid-1", "edition-uuid-2"]
}
```

### PATCH /{companyId}/token-editions/update-token-metadata

Update metadata for published editions.

**Request:**
```json
{
  "tokenEditionIds": ["edition-uuid-1"],
  "tokenData": {
    "artist": "Updated Artist Name",
    "year": 2027
  }
}
```

### GET /{companyId}/token-editions/check-rfid

Check if an RFID is already in use.

**Query:** `?rfid=RFID-VALUE`

**Response:** `{ "used": true/false }`

---

## Endpoints: ERC20 Contracts

Base path: `/{companyId}/erc20-contracts`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create ERC20 contract draft |
| 2 | GET | `/` | Bearer | Admin | List ERC20 contracts |
| 3 | GET | `/:id` | Bearer | Admin | Get ERC20 contract details |
| 4 | PATCH | `/:id` | Bearer | Admin | Update draft ERC20 |
| 5 | PATCH | `/:id/transfer-config` | Bearer | Admin | Update transfer rules |
| 6 | PATCH | `/:id/publish` | Bearer | Admin | Deploy ERC20 to blockchain |
| 7 | GET | `/:id/estimate-gas` | Bearer | Admin | Estimate deployment gas |
| 8 | PATCH | `/has-role` | Bearer | Admin, Integration | Check role |
| 9 | PATCH | `/grant-role` | Bearer | Admin, Integration | Grant role |

### POST /{companyId}/erc20-contracts

**Minimal Request:**
```json
{
  "name": "Loyalty Token",
  "symbol": "LYT",
  "chainId": 137,
  "type": "classic"
}
```

**Complete Request:**
```json
{
  "name": "Loyalty Token",
  "symbol": "LYT",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "1000000",
  "initialOwner": "0x1234567890abcdef1234567890abcdef12345678",
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": [
    { "model": "PERCENTAGE", "value": "0.05" },
    { "model": "MAX_LIMIT", "value": "10" },
    { "model": "PERIOD", "value": "5", "period": "7d" }
  ]
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Token name (latinized) |
| `symbol` | string | Yes | — | Token symbol (uppercase) |
| `chainId` | ChainId | Yes | — | Target blockchain |
| `type` | ERC20ContractType | Yes | — | `classic`, `permissioned`, or `in_custody` |
| `initialAmount` | string (bigint) | No | — | Initial mint amount |
| `initialOwner` | string | No | — | Initial owner address |
| `isBurnable` | boolean | No | `false` | Enable burn capability |
| `isWithdrawable` | boolean | No | `false` | Enable withdrawal |
| `transferConfig` | array | No | `[]` | Transfer rules (for permissioned/in_custody) |
| `maxSupply` | string (bigint) | No | — | Maximum supply |

**Transfer config object:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | ERC20TransferConfigModel | Yes | Rule type |
| `value` | string | Yes | Rule value (percentage 0–1, count, or fixed amount) |
| `period` | string | Conditional | Time window for PERIOD model (e.g., "7d", "24h") |

---

## Endpoints: ERC20 Token Operations

Base path: `/{companyId}/erc20-tokens`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | PATCH | `/:id/mint` | Bearer | Admin | Mint new tokens |
| 2 | PATCH | `/:id/transfer/admin` | Bearer | Admin | Admin transfer (bypasses rules) |
| 3 | PATCH | `/:id/transfer/user` | Bearer | Admin, User | User transfer (rules applied) |
| 4 | PATCH | `/:id/burn` | Bearer | Admin, User | Burn tokens |
| 5 | GET | `/:id/history` | Bearer | Admin, LoyaltyOperator | Transaction history (cached 10s) |
| 6 | GET | `/:id/history/action/:actionId` | Bearer | Admin | Specific action details |
| 7 | GET | `/:id/history/operator/:operatorId` | Bearer | Admin, LoyaltyOperator | Operator activity |
| 8 | GET | `/:id/history/:userId` | Bearer | Admin, User | User activity |

### PATCH /{companyId}/erc20-tokens/:id/mint

**Request:**
```json
{
  "to": "0xrecipient...",
  "amount": "1000",
  "metadata": { "reason": "Welcome bonus" },
  "sendEmail": true,
  "available": "1s"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | Yes | Recipient Ethereum address |
| `amount` | string | Yes | Amount to mint |
| `metadata` | object | No | Custom metadata for the transaction |
| `sendEmail` | boolean | No | Send notification email (default: true) |
| `available` | string | No | Availability delay expression (e.g., "1s", "24h") |

### PATCH /{companyId}/erc20-tokens/:id/transfer/user

User-initiated transfer. For `permissioned` and `in_custody` contracts, this goes through the **transfer handler chain**:

1. **FreeTransferHandler** — checks if transfers are free, forbidden, or restricted
2. **MaxLimitTransferHandler** — checks user hasn't exceeded max transfer count
3. **PeriodTransferHandler** — checks transfers within time window
4. **PercentageTransferHandler** — deducts percentage fee (creates 2 transactions)
5. **FixedTransferHandler** — deducts fixed fee (creates 2 transactions)

For `classic` contracts, the transfer executes directly.

**Request:** Same as admin transfer.

### PATCH /{companyId}/erc20-tokens/:id/burn

**Request:**
```json
{
  "from": "0xburner...",
  "amount": "500",
  "metadata": { "reason": "Redemption" },
  "sendEmail": true
}
```

---

## Endpoints: Royalty Eligible

Base path: `/{companyId}/contracts/royalty-eligible`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/create` | Bearer | Admin | Create royalty eligible record |
| 2 | GET | `/` | Bearer | Admin | List royalty eligible |
| 3 | GET | `/:id` | Bearer | Admin | Get details |
| 4 | PATCH | `/` | Bearer | Admin | Activate/deactivate |

### POST /{companyId}/contracts/royalty-eligible/create

```json
{
  "active": true,
  "displayName": "Artist Name",
  "userId": "user-uuid"
}
```

---

## Endpoints: Metadata & Certificates (Public)

| # | Method | Path | Auth | Description |
|---|--------|------|------|-------------|
| 1 | GET | `/metadata/rfid/{rfid}` | None | Get token metadata by RFID |
| 2 | GET | `/metadata/address/{contractAddress}/{chainId}/{tokenId}` | None | Get metadata by chain address + token ID |
| 3 | GET | `/metadata/contract/{address}/{chainId}` | None | Get contract metadata |
| 4 | GET | `/metadata/nfts/{walletAddress}/{chainId}` | None | List NFTs by wallet |
| 5 | GET | `/certification/{address}/{chainId}/{tokenId}` | None | Get PDF certificate |
| 6 | POST | `/certification/preview` | Bearer | Preview PDF certificate |

---

## Subcategories (Collection Templates)

Base path: `/{companyId}/subcategories`

| # | Method | Path | Auth | Description |
|---|--------|------|------|-------------|
| 1 | POST | `/` | Bearer (Admin) | Create template |
| 2 | PATCH | `/:id` | Bearer (Admin) | Update template |
| 3 | GET | `/` | Bearer (Admin) | List templates |
| 4 | GET | `/:id` | Bearer (Admin) | Get template |

Templates define the dynamic metadata fields for token collections (field types: DATE, BOOLEAN, DIMENSIONS_2D, DIMENSIONS_3D, NUMERIC, RADIOGROUP, SELECT, IMAGE, TEXTAREA, TEXTFIELD, YEAR).
