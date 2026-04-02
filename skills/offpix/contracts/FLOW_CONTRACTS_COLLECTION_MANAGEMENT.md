---
id: FLOW_CONTRACTS_COLLECTION_MANAGEMENT
title: "Contracts - Token Collection Management"
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
  - collections
  - editions
  - publish
  - bulk
depends_on:
  - CONTRACTS_API_REFERENCE
  - FLOW_CONTRACTS_NFT_LIFECYCLE
---

# Token Collection Management

## Overview

A token collection groups multiple token editions (individual NFTs) under shared metadata and a published NFT contract. This flow covers creating collections, configuring metadata templates, managing editions, publishing to blockchain, and bulk import/export via XLSX. In the frontend, collections are managed under **Tokens** (`/dash/tokens/collections/{id}`).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Published NFT contract | Contract with `status: published` | [NFT Lifecycle](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Subcategory (template) | Collection template with metadata fields | Create via subcategories endpoint |

## State Machines

**Collection:** `DRAFT → PUBLISHED`

**Edition:**
```
DRAFT ──→ READY_TO_MINT ──→ MINTING ──→ MINTED
  │                                        │
  ├→ DRAFT_ERROR                          ├→ TRANSFERRING → TRANSFERRED
  ├→ IMPORT_ERROR                         ├→ BURNING → BURNED
  │                                        │
  └→ LOCKED_FOR_BUY → (back to DRAFT)    ├→ TRANSFER_FAILURE
                                           └→ BURN_FAILURE
```

---

## Flow: Create a Collection

### Step 1 (Optional): Create a Collection Template

Templates define which metadata fields are available for tokens in the collection.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/subcategories` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "name": "Digital Art Template",
  "categoryId": "category-uuid",
  "tokenTemplate": {
    "fields": [
      { "name": "artist", "type": "TEXTFIELD", "required": true },
      { "name": "year", "type": "YEAR", "required": true },
      { "name": "medium", "type": "SELECT", "options": ["Digital", "Mixed Media"] },
      { "name": "dimensions", "type": "DIMENSIONS_2D" },
      { "name": "highResImage", "type": "IMAGE" }
    ]
  }
}
```

**Template field types:** `DATE`, `BOOLEAN`, `DIMENSIONS_2D`, `DIMENSIONS_3D`, `NUMERIC`, `RADIOGROUP`, `SELECT`, `IMAGE`, `TEXTAREA`, `TEXTFIELD`, `YEAR`

### Step 2: Create Collection Draft

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/token-collections` | Bearer (Admin) | application/json |

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
  "contractId": "published-contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "Genesis Collection",
  "description": "Limited edition digital artworks",
  "mainImage": "https://cdn.example.com/collection.png",
  "tokenData": {
    "artist": "John Doe",
    "year": 2026,
    "medium": "Digital"
  },
  "quantity": 100,
  "rangeInitialToMint": "1-50",
  "ownerAddress": "0xOwnerWallet...",
  "similarTokens": true,
  "settings": {
    "sendWebhookWhenTokenEditionIsUpdated": true
  }
}
```

**Notes:**
- `similarTokens: true` means all editions share the same metadata (tokenData). Set to `false` for unique editions
- `rangeInitialToMint` specifies which editions to mint on publish (e.g., "1-50" mints editions 1 through 50)
- `quantity` sets the total number of editions in the collection
- If `contractId` is omitted, the collection is contract-independent (can be assigned later)

### Step 3: Sync Draft Editions

Create the individual draft token editions for the collection.

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-collections/{collectionId}/sync-draft-tokens` | Bearer (Admin) |

**Response (201):**
```json
{
  "jobId": "job-uuid"
}
```

This is an **async operation** processed via Bull queue. The job creates `quantity` draft editions numbered sequentially.

---

## Flow: Manage Editions (Individual Tokens)

### List Editions

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions` | Bearer (Admin) |

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `tokenCollectionId` | UUID | — | Filter by collection |
| `status` | TokenEditionStatus[] | — | Filter by status |
| `editionNumber` | number | — | Specific edition |
| `walletAddresses` | string[] | — | Filter by owner wallet |
| `userId` | UUID | — | Filter by user |
| `search` | string | — | Search in name/description |
| `sortBy` | string | `editionNumber` | Sort column |
| `orderBy` | `ASC` \| `DESC` | `ASC` | Sort direction |
| `page`, `limit` | integer | 1, 10 | Pagination |

### Update Edition Metadata

For collections with `similarTokens: false`, each edition can have unique metadata.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/token-editions/update-token-metadata` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "tokenEditionIds": ["edition-uuid-1", "edition-uuid-2"],
  "tokenData": {
    "artist": "Updated Name",
    "year": 2027
  }
}
```

If the collection has `settings.sendWebhookWhenTokenEditionIsUpdated: true`, a webhook fires after the update.

### Assign RFID

Link a physical RFID tag to a token edition.

**Step 1: Check RFID availability**
```
GET /{companyId}/token-editions/check-rfid?rfid=RFID-VALUE-123
```
Response: `{ "used": false }`

**Step 2: Assign RFID**
```
PATCH /{companyId}/token-editions/{editionId}/rfid
```
```json
{ "rfid": "RFID-VALUE-123" }
```

---

## Flow: Publish a Collection

### Step 1: Mark Editions as Ready

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/token-editions/ready-to-mint` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "editionId": ["edition-uuid-1", "edition-uuid-2", "edition-uuid-3"]
}
```

Status transition: `draft` → `readyToMint`

### Step 2: Estimate Gas

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-collections/estimate-gas` | Bearer (Admin) |

**Query:** `?contractId={uuid}&initialQuantityToMint={count}&ownerAddress={addr}`

**Response:** Gas estimation with fast/proposed/safe tiers.

### Step 3: Publish

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/token-collections/publish/{collectionId}` | Bearer (Admin) | application/json |

The request body can include updated collection metadata (same as update DTO).

**Response (201):**
```json
{
  "tokenCollection": {
    "id": "collection-uuid",
    "status": "published",
    "quantityMinted": 50,
    "..."
  },
  "validationErrors": []
}
```

**What happens internally:**
1. All `readyToMint` editions transition to `minting`
2. Blockchain minting jobs are created
3. As each mint confirms: edition → `minted`, gets `tokenId`, `mintedHash`, `mintedAt`
4. Collection status → `published`

**If validation fails:** Returns `validationErrors` array with field-level errors using `PublishValidationsEnum`.

---

## Flow: Bulk Import/Export via XLSX

### Export Template

```
GET /{companyId}/token-collections/{collectionId}/export/xlsx
```

Returns a binary XLSX file with current edition data and template columns.

### Import Editions

```
POST /{companyId}/token-collections/{collectionId}/import/xlsx
Content-Type: multipart/form-data
```

Upload a filled XLSX file (max 10MB). Returns `{ "jobId": "..." }` for async processing.

**Notes:**
- The XLSX must follow the template structure from the export
- Invalid rows create editions with `importError` status
- Validation errors are stored in the edition's `errorFields` JSONB

---

## Flow: Delete / Burn Collection

### Delete Draft Collection

```
DELETE /{companyId}/token-collections/{collectionId}/draft
```

Only works on `draft` collections. Soft-deletes the collection and all draft editions.

### Burn Published Collection

```
DELETE /{companyId}/token-collections/{collectionId}/burn
```

Burns all minted tokens in the collection. This is an on-chain operation.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | PublishValidation errors | Editions have invalid data | Fix errors listed in `validationErrors` response |
| 400 | CONTRACT_HAS_NO_ADDRESS | Contract not published yet | Publish the contract first |
| 400 | CONTRACT_NO_ENABLE_MINTER | Contract missing `admin:minter` feature | Create new contract with minter feature |
| 400 | RFID_HAS_ALREADY_BEEN_USED | RFID assigned to another edition | Use a different RFID |
| 400 | JOB_RUNNING | Another async operation in progress | Wait for current job to complete |
| 404 | NotFoundException | Collection or edition not found | Verify IDs |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Editions not created after collection | Call `sync-draft-tokens` to generate the edition records |
| 2 | Publish fails with validation errors | Each edition must pass template validation. Check `errorFields` on failed editions |
| 3 | RFID conflicts | Check RFID availability before assigning. RFIDs are globally unique (not per collection) |
| 4 | XLSX import creates `importError` editions | Download the template first, fill it correctly, then re-import |
| 5 | Can't delete published collection | Use burn instead of delete for published collections |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| NFT Contract | Create the contract before collections | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Token Operations | Mint, transfer, burn individual tokens | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| API Reference | Full endpoint details | [CONTRACTS_API_REFERENCE](./CONTRACTS_API_REFERENCE.md) |
