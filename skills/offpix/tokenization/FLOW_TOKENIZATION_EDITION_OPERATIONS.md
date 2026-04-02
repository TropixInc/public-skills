---
id: FLOW_TOKENIZATION_EDITION_OPERATIONS
title: "Tokenization - Edition Operations (Mint, Transfer, Burn, RFID, Metadata)"
module: offpix/tokenization
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - editions
  - mint
  - transfer
  - burn
  - rfid
  - metadata
depends_on:
  - TOKENIZATION_API_REFERENCE
  - FLOW_TOKENIZATION_COLLECTION_LIFECYCLE
---

# Token Edition Operations

## Overview

This flow covers all operations on individual token editions after a collection has been published to the blockchain. Operations include marking editions as ready to mint, minting on demand, transferring tokens between wallets, burning tokens, updating on-chain metadata, managing RFID assignments, estimating gas costs, and locking/unlocking editions for commerce flows.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin, Integration, or User role (per endpoint) | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Published collection | Token collection in PUBLISHED status | [Collection Lifecycle](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |

---

## Edition Status State Machine

```
                    +---------------+
                    | IMPORT_ERROR  |
                    +---------------+
                           ^
                           | (xlsx import failure)
                           |
+---------------+    +-----+-----+    +---------------+
| DRAFT_ERROR   |<---+   DRAFT   +--->| LOCKED_FOR_BUY|
+---------------+    +-----+-----+    +-------+-------+
                           |                  |
                           v                  | (unlock)
                    +------+--------+         |
                    | READY_TO_MINT |<--------+
                    +------+--------+
                           |
                           v
                    +------+--------+
                    |    MINTING    |
                    +------+--------+
                           |
                           v
                    +------+--------+
              +---->|    MINTED     |<----+
              |     +--+--------+--+     |
              |        |        |        |
              |        v        v        |
              |  +-----+--+ +--+------+ |
              |  |BURNING  | |TRANSFER-| |
              |  |         | |RING     | |
              |  +----+--++ +--+---+--+ |
              |       |  |     |   |    |
              |       v  |     v   |    |
              | +-----++ | +--+---++   |
              | |BURNED | | |TRANS- |   |
              | +-------+ | |FERRED +---+
              |           | +-------+
              |           v
              |    +------+--------+
              |    | BURN_FAILURE  |
              |    +---------------+
              |
              |    +---------------+
              +----+TRANSFER_      |
                   |FAILURE        |
                   +---------------+
```

**State transitions:**
- `DRAFT` --> `DRAFT_ERROR`: validation error during sync or configuration
- `DRAFT` --> `IMPORT_ERROR`: error during XLSX import
- `DRAFT` --> `READY_TO_MINT`: marked ready (manual or via rangeInitialToMint at publish)
- `DRAFT` --> `LOCKED_FOR_BUY`: locked for a commerce purchase
- `LOCKED_FOR_BUY` --> `READY_TO_MINT`: unlocked
- `READY_TO_MINT` --> `MINTING`: mint transaction submitted
- `MINTING` --> `MINTED`: mint transaction confirmed
- `MINTED` --> `TRANSFERRING`: transfer transaction submitted
- `MINTED` --> `BURNING`: burn transaction submitted
- `TRANSFERRING` --> `TRANSFERRED`: transfer confirmed
- `TRANSFERRING` --> `TRANSFER_FAILURE`: transfer failed
- `TRANSFERRED` --> `TRANSFERRING`: another transfer submitted
- `TRANSFERRED` --> `BURNING`: burn transaction submitted
- `BURNING` --> `BURNED`: burn confirmed
- `BURNING` --> `BURN_FAILURE`: burn failed
- `TRANSFER_FAILURE` --> `MINTED`: retry resets to minted
- `BURN_FAILURE` --> `MINTED`: retry resets to minted

---

## Flow: Mark Ready to Mint

Before minting, editions must be in READY_TO_MINT status. Editions in the `rangeInitialToMint` range are automatically set to this status on publish. For other editions, use these endpoints.

### Bulk Mark Ready

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/ready-to-mint` | Bearer (Admin) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [51, 52, 53, 54, 55]
}
```

**Response (200):**

```json
{
  "updated": 5
}
```

### Single Mark Ready

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/{id}/ready-to-mint` | Bearer (Admin) |

No request body required.

**Response (200):** Returns the updated TokenEditionEntity with status `readyToMint`.

**Notes:**
- Editions must be in DRAFT status to be marked as ready
- Editions with DRAFT_ERROR or IMPORT_ERROR cannot be marked ready (fix errors first)

---

## Flow: Mint on Demand

Mint specific editions to a wallet address.

### Mint Editions

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/mint-on-demand` | Bearer (Admin/Integration) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678"
}
```

**Response (200):**

```json
{
  "editions": [
    {
      "editionNumber": 1,
      "status": "minting"
    },
    {
      "editionNumber": 2,
      "status": "minting"
    },
    {
      "editionNumber": 3,
      "status": "minting"
    }
  ]
}
```

**Notes:**
- Editions must be in READY_TO_MINT status
- The `ownerAddress` is the wallet that will own the minted NFTs
- Minting is async -- editions move to MINTING and then to MINTED once the blockchain transaction is confirmed
- The collection must be in PUBLISHED status
- Each minted edition receives a `tokenId`, `mintedHash`, `mintedAt`, and `contractAddress` from the blockchain transaction
- If minting fails, the edition remains in MINTING status (retry via `retry-bulk-by-collection`)

---

## Flow: Transfer Tokens

Transfer minted tokens to a new owner. Three methods are available: bulk admin transfer, single transfer, and transfer by email.

### Bulk Transfer (Admin)

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/transfer-token` | Bearer (Admin) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef"
}
```

**Response (200):**

```json
{
  "editions": [
    {
      "editionNumber": 1,
      "status": "transferring"
    },
    {
      "editionNumber": 2,
      "status": "transferring"
    },
    {
      "editionNumber": 3,
      "status": "transferring"
    }
  ]
}
```

### Single Transfer

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/{id}/transfer-token` | Bearer (Admin/User) |

**Request Body:**

```json
{
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef"
}
```

**Response (200):** Returns the updated TokenEditionEntity with status `transferring`.

### Transfer by Email

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/{id}/transfer-token/email` | Bearer (Admin/User) |

**Request Body:**

```json
{
  "email": "recipient@example.com"
}
```

**Response (200):** Returns the updated TokenEditionEntity with status `transferring`.

**Notes:**
- Editions must be in MINTED or TRANSFERRED status
- Transfer is async -- editions move to TRANSFERRING and then to TRANSFERRED once confirmed
- Users (non-admin) can only transfer tokens they own (the `ownerAddress` must match the authenticated user's wallet)
- Email transfer resolves the recipient's email to their wallet address
- If the recipient email is not registered, the transfer may be deferred until the user registers and has a wallet
- On success, `ownerAddress` on the edition is updated to the new owner
- On failure, the edition moves to TRANSFER_FAILURE status

---

## Flow: Burn Tokens

Permanently destroy minted tokens on the blockchain.

### Burn Editions

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| DELETE | `/{companyId}/token-editions/burn` | Bearer (Admin/User) |

**Request Body:**

```json
{
  "tokens": [
    {
      "tokenCollectionId": "collection-uuid",
      "editionNumber": 1
    },
    {
      "tokenCollectionId": "collection-uuid",
      "editionNumber": 2
    }
  ]
}
```

**Response (200):**

```json
{
  "editions": [
    {
      "editionNumber": 1,
      "status": "burning"
    },
    {
      "editionNumber": 2,
      "status": "burning"
    }
  ]
}
```

**Notes:**
- Editions must be in MINTED or TRANSFERRED status
- Burn is async -- editions move to BURNING and then to BURNED once confirmed
- Users (non-admin) can only burn tokens they own
- Burned tokens are permanently destroyed on the blockchain
- Burned editions' RFIDs become available for reuse
- On failure, the edition moves to BURN_FAILURE status

---

## Flow: Update Token Metadata

Update the metadata of minted editions. This can trigger an on-chain metadata refresh and optional webhook notification.

### Update Metadata

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/update-token-metadata` | Bearer (Admin/Integration) |

**Minimal Request:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1],
  "tokenData": {
    "name": "Updated Asset Name"
  }
}
```

**Complete Request:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "tokenData": {
    "name": "Updated Asset Name",
    "description": "Updated description",
    "image": "https://example.com/updated-image.png",
    "attributes": [
      { "trait_type": "level", "value": "5" },
      { "trait_type": "experience", "value": "1500" }
    ]
  },
  "mergeTokenData": true,
  "keepStatus": true
}
```

**Response (200):**

```json
{
  "updated": 3
}
```

**Field behavior:**

| Field | Default | Description |
|-------|---------|-------------|
| `mergeTokenData` | false | When `true`, provided tokenData is shallow-merged with existing data (new keys added, existing keys overwritten). When `false`, the entire tokenData is replaced |
| `keepStatus` | false | When `true`, the edition's current status is preserved. When `false`, the status may be reset to draft for re-processing |

**Notes:**
- If the collection has `settings.sendWebhookWhenTokenEditionIsUpdated` set to `true`, a webhook notification is triggered after the update
- Metadata updates affect the off-chain metadata; on-chain metadata may require an additional refresh depending on the contract implementation
- The `mergeTokenData` option is useful for updating individual fields without losing existing data

---

## Flow: RFID Management

Assign and manage RFID tags on individual editions. RFIDs are globally unique identifiers that link physical items to digital tokens.

### Check RFID Availability

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/check-rfid` | Bearer (Admin) |

**Query:**

```
GET /{companyId}/token-editions/check-rfid?rfid=RFID-001-ABC
```

**Response (available):**

```json
{
  "available": true
}
```

**Response (in use):**

```json
{
  "available": false,
  "usedBy": {
    "editionId": "edition-uuid",
    "tokenCollectionId": "collection-uuid",
    "editionNumber": 42
  }
}
```

### Assign RFID

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/{id}/rfid` | Bearer (Admin) |

**Request Body:**

```json
{
  "rfid": "RFID-001-ABC"
}
```

**Response (200):** Returns the updated TokenEditionEntity with the assigned RFID.

**Notes:**
- RFIDs are globally unique across all active editions in the platform (excluding deleted and burned editions)
- Always check availability with `check-rfid` before assigning to avoid conflicts
- Assigning a new RFID to an edition that already has one replaces the existing RFID (the old one becomes available)
- RFIDs of burned editions are released and become available for reuse

---

## Flow: Gas Estimation

Estimate gas costs before executing on-chain operations (mint, transfer, burn).

### Estimate Mint Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/mint` | Bearer (Admin) |

**Response (200):**

```json
{
  "slow": {
    "gasPrice": "30000000000",
    "estimatedCost": "0.003",
    "estimatedTime": "120s"
  },
  "standard": {
    "gasPrice": "50000000000",
    "estimatedCost": "0.005",
    "estimatedTime": "60s"
  },
  "fast": {
    "gasPrice": "80000000000",
    "estimatedCost": "0.008",
    "estimatedTime": "30s"
  }
}
```

### Estimate Transfer Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/transfer?toAddress={address}` | Bearer (Admin/User) |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `toAddress` | string | Yes | Destination wallet address for the estimation |

**Response (200):** Same structure as mint gas estimation (slow/standard/fast tiers).

### Estimate Burn Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/burn` | Bearer (Admin/User) |

**Response (200):** Same structure as mint gas estimation (slow/standard/fast tiers).

**Notes:**
- Gas estimates reflect current network conditions and may change rapidly
- The three tiers provide different trade-offs between cost and confirmation speed
- Costs are denominated in the chain's native token (e.g., MATIC for Polygon, ETH for Ethereum)
- Estimates are approximations -- actual costs may vary slightly

---

## Flow: Lock/Unlock for Commerce

Lock editions for purchase flows and unlock them when the purchase is cancelled or completed without minting.

### Lock for Buy

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/locked-for-buy` | Bearer (Admin/Integration) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [10, 11, 12]
}
```

**Response (200):**

```json
{
  "updated": 3
}
```

### Unlock for Buy

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/unlocked-for-buy` | Bearer (Admin/Integration) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [10, 11, 12]
}
```

**Response (200):**

```json
{
  "updated": 3
}
```

**Notes:**
- Lock sets editions from DRAFT or READY_TO_MINT to LOCKED_FOR_BUY status
- Unlock returns editions to DRAFT status
- Used by the commerce module to reserve specific editions during a purchase flow
- Locked editions cannot be minted, transferred, or burned until unlocked

---

## Flow: Notify Externally Minted

Mark editions as minted when the minting was performed outside the W3Block platform.

### Notify External Mint

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/notify-externally-minted` | Bearer (Admin/Integration) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "contractAddress": "0xabcdef1234567890abcdef1234567890abcdef12",
  "chainId": 137
}
```

**Response (200):**

```json
{
  "updated": 3
}
```

**Notes:**
- Sets editions to MINTED status without executing a minting transaction on the platform
- Used for integrations where minting occurs on external platforms or custom smart contracts
- The `contractAddress` and `chainId` are recorded on the editions for reference

---

## Flow: Get Last Transaction

Retrieve the most recent transfer or burn transaction for an edition.

### Get Last Transaction

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/{id}/get-last/{type}` | Bearer (Admin) |

**Path Parameters:**

| Parameter | Values | Description |
|-----------|--------|-------------|
| `type` | `transfer`, `burn` | Transaction type to retrieve |

**Example:**

```
GET /{companyId}/token-editions/{id}/get-last/transfer
```

**Response (200):**

```json
{
  "id": "transaction-uuid",
  "type": "transfer",
  "transactionHash": "0xabc123def456...",
  "fromAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef",
  "status": "confirmed",
  "createdAt": "2026-04-02T14:00:00.000Z"
}
```

**Notes:**
- Returns only the most recent transaction of the specified type
- Useful for verifying transaction status or displaying transaction details in the UI

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | TokenEditionInvalidStatusException | Edition is not in the required status for the operation | Check the state machine diagram above for valid transitions |
| 400 | RfidAlreadyUsedException | RFID is assigned to another active edition | Use `check-rfid` before assigning; choose a different RFID |
| 400 | RfidDuplicateException | Duplicate RFIDs in the submitted list | Remove duplicates from the request |
| 404 | TokenEditionNotFoundException | Edition not found for the given company | Verify edition ID and companyId |
| 400 | TokenCollectionNotPublishedException | Collection is not published | Publish the collection first |
| 400 | GasEstimationException | Gas estimation failed | Check contract deployment and network connectivity |
| 403 | Forbidden | User trying to operate on tokens they do not own | Users can only transfer/burn their own tokens |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Minting editions that are not READY_TO_MINT | Mark editions as ready first with `ready-to-mint` endpoint |
| 2 | User cannot transfer/burn | Users can only operate on tokens where the `ownerAddress` matches their wallet. Admin role can operate on any token |
| 3 | Transfer/burn shows as failed | Check the edition status (TRANSFER_FAILURE or BURN_FAILURE). Use `retry-bulk-by-collection` to retry |
| 4 | Metadata update does not trigger webhook | Ensure `settings.sendWebhookWhenTokenEditionIsUpdated` is enabled on the collection |
| 5 | RFID assignment rejected | Check availability with `check-rfid` first. RFIDs must be globally unique across all active editions |
| 6 | Locked editions cannot be minted | Unlock them first with `unlocked-for-buy`, then mark as ready and mint |
| 7 | mergeTokenData unexpected behavior | When `mergeTokenData` is false (default), the entire tokenData is replaced. Use `true` to update individual fields |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Collection Lifecycle | Required: collection must be published first | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Bulk Import | Alternative: configure editions via Excel | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |
| API Reference | Full endpoint and DTO details | [TOKENIZATION_API_REFERENCE](./TOKENIZATION_API_REFERENCE.md) |
| Contracts NFT Lifecycle | Required: contract must exist for minting | [FLOW_CONTRACTS_NFT_LIFECYCLE](../contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Auth Sign-In | Required for Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
