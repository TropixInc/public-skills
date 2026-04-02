---
id: FLOW_TOKENIZATION_COLLECTION_LIFECYCLE
title: "Tokenization - Collection Lifecycle (Draft to Published)"
module: offpix/tokenization
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - collections
  - draft
  - publish
  - lifecycle
depends_on:
  - TOKENIZATION_API_REFERENCE
---

# Token Collection Lifecycle

## Overview

This flow covers the full lifecycle of a token collection from creation to blockchain publication. A token collection starts in DRAFT status, where editions are generated and configured, and transitions to PUBLISHED status when deployed on-chain. The flow includes creating a draft, syncing edition drafts, configuring metadata, estimating gas costs, and publishing to the blockchain.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Deployed contract | Contract in PUBLISHED status with ADMIN_MINTER | [Contracts NFT Lifecycle](../contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Subcategory template | Defines the metadata schema for token editions | Platform configuration |

---

## Entities & Relationships

```
Contract (PUBLISHED, ADMIN_MINTER)
    |
    +---> TokenCollection (DRAFT -> PUBLISHED)
              |
              +---> TokenEdition #1 (DRAFT -> READY_TO_MINT -> MINTING -> MINTED)
              +---> TokenEdition #2 (DRAFT -> READY_TO_MINT -> MINTING -> MINTED)
              +---> TokenEdition #3 (DRAFT)
              +---> ...
              +---> TokenEdition #N (DRAFT)

Subcategory Template
    |
    +---> Defines tokenData fields and validation rules
    +---> Snapshot saved as publishedTokenTemplate on publish
```

**Key relationships:**
- A Contract can have multiple TokenCollections
- A TokenCollection has exactly `quantity` TokenEditions
- Each TokenEdition has a sequential `editionNumber` (1 to quantity)
- If `similarTokens` is true, all editions share the collection-level tokenData
- If `similarTokens` is false, each edition has individual tokenData

---

## Flow: Create Collection Draft

### Step 1: Create the Collection

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| POST | `/{companyId}/token-collections` | Bearer (Admin) |

**Minimal Request:**

```json
{
  "name": "My NFT Collection"
}
```

**Complete Request:**

```json
{
  "name": "My NFT Collection",
  "description": "A collection of 100 unique digital assets",
  "mainImage": "https://example.com/collection-cover.png",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "quantity": 100,
  "initialQuantityToMint": 50,
  "similarTokens": true,
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "rangeInitialToMint": "1-50",
  "tokenData": {
    "name": "Digital Asset",
    "description": "A unique piece",
    "image": "https://example.com/asset.png"
  }
}
```

**Response (201):**

```json
{
  "id": "collection-uuid",
  "status": "draft",
  "companyId": "company-uuid",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "My NFT Collection",
  "description": "A collection of 100 unique digital assets",
  "mainImage": "https://example.com/collection-cover.png",
  "tokenData": {
    "name": "Digital Asset",
    "description": "A unique piece",
    "image": "https://example.com/asset.png"
  },
  "quantity": 100,
  "initialQuantityToMint": 50,
  "initialQuantity": 100,
  "quantityMinted": 0,
  "rfids": [],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "similarTokens": true,
  "pass": false,
  "settings": null,
  "rangeInitialToMint": "1-50",
  "createdAt": "2026-04-02T12:00:00.000Z",
  "updatedAt": "2026-04-02T12:00:00.000Z"
}
```

**Notes:**
- `contractId` is optional at creation but **required** before publish
- `subcategoryId` is required for metadata validation at publish time
- `quantity` defaults to 0 if not provided; update it before syncing editions
- `similarTokens` defaults to true -- all editions will share the same tokenData from the collection level
- Save the returned `id` for all subsequent operations

---

## Flow: Sync Draft Editions

After creating a collection and setting the `quantity`, generate the individual edition drafts.

### Step 2: Trigger Edition Sync

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-collections/{id}/sync-draft-tokens` | Bearer (Admin) |

No request body required.

**Response (200):**

```json
{
  "jobId": "job-uuid"
}
```

### Step 3: Poll for Job Completion

The sync is an async operation. Poll the job status endpoint until completion.

**Poll Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/jobs/{jobId}` | Bearer (Admin) |

**Response (in progress):**

```json
{
  "id": "job-uuid",
  "status": "processing",
  "progress": 45,
  "createdAt": "2026-04-02T12:05:00.000Z"
}
```

**Response (completed):**

```json
{
  "id": "job-uuid",
  "status": "completed",
  "progress": 100,
  "result": {
    "created": 100,
    "updated": 0,
    "deleted": 0
  },
  "createdAt": "2026-04-02T12:05:00.000Z",
  "completedAt": "2026-04-02T12:06:00.000Z"
}
```

**Notes:**
- Creates `quantity` editions with sequential `editionNumber` from 1 to N
- All editions start in DRAFT status
- If editions already exist (from a previous sync), the job re-syncs: adds missing editions, removes excess ones
- If `quantity` was changed after a previous sync, re-syncing adjusts the editions accordingly
- **You must wait for this job to complete before attempting to publish**

---

## Flow: Configure Collection (Optional)

Before publishing, you can update the collection configuration.

### Step 4: Update Collection

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PUT | `/{companyId}/token-collections/{id}` | Bearer (Admin) |

**Request Body (partial -- only include fields to update):**

```json
{
  "name": "Updated Collection Name",
  "description": "Updated description with more detail",
  "mainImage": "https://example.com/new-cover.png",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "quantity": 150,
  "initialQuantityToMint": 75,
  "rangeInitialToMint": "1-75",
  "tokenData": {
    "name": "Digital Asset v2",
    "description": "An updated unique piece",
    "image": "https://example.com/asset-v2.png",
    "attributes": [
      { "trait_type": "collection", "value": "Genesis" }
    ]
  },
  "ownerAddress": "0xnewowner1234567890abcdef1234567890abcdef",
  "similarTokens": true,
  "rfids": ["RFID-001", "RFID-002", "RFID-003"],
  "settings": {
    "sendWebhookWhenTokenEditionIsUpdated": true
  }
}
```

**Response (200):** Returns the updated TokenCollectionEntity.

**Notes:**
- Only allowed when status is `draft`
- If `quantity` is changed, you must re-sync editions (step 2) before publishing
- `contractId` must reference a contract that is PUBLISHED with ADMIN_MINTER enabled
- `rangeInitialToMint` must be a valid range format: `"1-50"` (continuous range) or `"1,5,10"` (specific editions)
- `tokenData` should conform to the subcategory template schema
- `rfids` list must contain unique values that are not used by other active editions

---

## Flow: Publish Collection

When the collection is fully configured and editions are synced, publish it to the blockchain.

### Step 5: Publish to Blockchain

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-collections/publish/{id}` | Bearer (Admin) |

No request body required.

**Response (200):**

```json
{
  "id": "collection-uuid",
  "status": "published",
  "companyId": "company-uuid",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "My NFT Collection",
  "description": "A collection of 100 unique digital assets",
  "mainImage": "https://example.com/collection-cover.png",
  "tokenData": {
    "name": "Digital Asset",
    "description": "A unique piece",
    "image": "https://example.com/asset.png"
  },
  "publishedTokenTemplate": {
    "fields": [
      { "name": "name", "type": "string", "required": true },
      { "name": "description", "type": "string", "required": false },
      { "name": "image", "type": "string", "required": true }
    ]
  },
  "quantity": 100,
  "initialQuantityToMint": 50,
  "initialQuantity": 100,
  "quantityMinted": 0,
  "rfids": [],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "similarTokens": true,
  "pass": false,
  "rangeInitialToMint": "1-50",
  "createdAt": "2026-04-02T12:00:00.000Z",
  "updatedAt": "2026-04-02T12:30:00.000Z"
}
```

**Validation Checks (all must pass):**

| Check | Validation | Error Code |
|-------|-----------|------------|
| Contract exists and is published | Contract status must be `published` | `CONTRACT_HAS_NO_ADDRESS` |
| Contract has ADMIN_MINTER | Contract must have minter feature enabled | `CONTRACT_NO_ENABLE_MINTER` |
| Owner address set | Either collection `ownerAddress` or company default owner address | `COMPANY_HAS_NO_DEFAULT_OWNER_ADDRESS` |
| Editions synced | All editions must exist (count matches quantity) | `EDITIONS_NOT_SYNC` |
| All editions in DRAFT status | No editions with DRAFT_ERROR or IMPORT_ERROR | `EDITIONS_WITH_ERRORS` |
| No running jobs | No async jobs in progress for this collection | `JOB_RUNNING` |
| RFIDs unique | No duplicate RFIDs in the list | `RFID_HAS_ITEMS_DUPLICATED_ON_LIST` |
| RFIDs available | No RFIDs already used by other collections | `RFID_HAS_ALREADY_BEEN_USED` |
| TokenData valid | If `similarTokens`, tokenData must match subcategory template | Various field validation codes |
| Range valid | `rangeInitialToMint` must be valid format | `INVALID_RANGE` |

**What happens on publish:**

1. The subcategory template is snapshot as `publishedTokenTemplate`
2. If `similarTokens` is true, the collection-level `tokenData` is copied to all editions
3. Editions in the `rangeInitialToMint` range have their status set to `readyToMint`
4. The collection status changes from `draft` to `published` (this is **irreversible**)
5. Editions outside the `rangeInitialToMint` range remain in `draft` status

---

## Flow: Estimate Gas Before Publish

Estimate the gas cost before committing to a publish operation.

### Estimate Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-collections/estimate-gas` | Bearer (Admin) |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `contractId` | uuid | Yes | Contract ID for the collection |
| `initialQuantityToMint` | number | Yes | Number of tokens to mint on publish |

**Response (200):**

```json
{
  "slow": {
    "gasPrice": "30000000000",
    "estimatedCost": "0.005",
    "estimatedTime": "120s"
  },
  "standard": {
    "gasPrice": "50000000000",
    "estimatedCost": "0.008",
    "estimatedTime": "60s"
  },
  "fast": {
    "gasPrice": "80000000000",
    "estimatedCost": "0.012",
    "estimatedTime": "30s"
  }
}
```

**Notes:**
- Gas estimation uses current network conditions and may fluctuate
- The three tiers (slow/standard/fast) represent different gas price strategies
- Costs are in the chain's native token (e.g., MATIC for Polygon)

---

## Flow: Delete Draft Collection

Remove a collection that has not been published.

### Delete Draft

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| DELETE | `/{companyId}/token-collections/{id}/draft` | Bearer (Admin) |

No request body required.

**Response:** 204 No Content

**Notes:**
- Only allowed when status is `draft`
- Deletes the collection and all associated edition drafts
- This operation is irreversible
- Cannot be called on published collections (use burn instead)

---

## Flow: Burn Published Collection

Remove a published collection and burn all its on-chain tokens.

### Burn Collection

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| DELETE | `/{companyId}/token-collections/{id}/burn` | Bearer (Admin) |

No request body required.

**Response (200):**

```json
{
  "id": "collection-uuid",
  "status": "published",
  "burnInitiated": true
}
```

**Notes:**
- Only allowed when status is `published`
- Initiates burn transactions for all minted editions on-chain
- This is an async operation -- editions transition through BURNING to BURNED status
- The collection itself remains in PUBLISHED status but editions become BURNED

---

## Flow: Increase Editions Post-Publish

Add more editions to a published collection.

### Increase Editions

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-collections/{id}/increase-editions` | Bearer (Integration) |

**Request Body:**

```json
{
  "quantity": 50
}
```

**Response (200):** Returns the updated TokenCollectionEntity with increased quantity.

**Notes:**
- Only allowed when status is `published`
- Requires Integration role (not standard Admin)
- New editions are created starting from the next sequential edition number
- New editions start in DRAFT status
- The collection's `quantity` is increased by the specified amount

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | TokenCollectionPublishValidationException | One or more publish validations failed | Check the `validations` array in the error response for specific issues |
| 400 | TokenCollectionNotDraftException | Attempting to edit/delete a published collection | Use published-state operations (burn, increase-editions) instead |
| 400 | JobRunningException | Async job still in progress | Wait for the current job to complete before retrying |
| 400 | InvalidRangeException | rangeInitialToMint format is invalid | Use format like "1-50" or "1,5,10" |
| 400 | ContractNotPublishedException | Contract is not published | Publish the contract first via the Contracts module |
| 400 | ContractNoMinterException | Contract lacks ADMIN_MINTER | Enable ADMIN_MINTER on the contract via the Contracts module |
| 404 | TokenCollectionNotFoundException | Collection not found | Verify the collection ID and companyId |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Publish fails with EDITIONS_NOT_SYNC | Call `sync-draft-tokens` and wait for the job to complete before publishing |
| 2 | Publish fails with EDITIONS_WITH_ERRORS | List editions filtered by error status, fix issues (via edit or re-import), then retry publish |
| 3 | Changing quantity after sync | If you update `quantity` after syncing, you must re-sync editions before publishing |
| 4 | Contract not ready | Ensure the contract is PUBLISHED with ADMIN_MINTER before attempting to publish the collection |
| 5 | Missing owner address | Set `ownerAddress` on the collection or configure a default owner address on the company |
| 6 | RFID conflicts at publish | Check all RFIDs for uniqueness with `check-rfid` before assigning them to the collection |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Contracts NFT Lifecycle | Required: contract must be deployed and published first | [FLOW_CONTRACTS_NFT_LIFECYCLE](../contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Edition Operations | Next: manage editions after collection is published | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Bulk Import | Alternative: configure editions via Excel before publish | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |
| Auth Sign-In | Required for Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
