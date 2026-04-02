---
id: TOKENIZATION_API_REFERENCE
title: "Tokenization - API Reference"
module: offpix/tokenization
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - token-collections
  - token-editions
  - nft
  - api-reference
---

# Tokenization API Reference

Complete endpoint reference for the W3Block Tokenization module. This module manages two primary entities -- Token Collections (groups of tokens with shared configuration) and Token Editions (individual token instances within a collection) -- served by the KEY backend.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://api.w3block.io` |
| Swagger | https://api.w3block.io/docs/ |

## Authentication

All endpoints require:

```
Authorization: Bearer {accessToken}
```

Most endpoints require the **Admin** role. Some endpoints also accept **Integration** or **User** roles as noted per endpoint.

---

## Enums

### TokenCollectionStatus

Defines the lifecycle state of a token collection.

| Value | Description |
|-------|-------------|
| `draft` | Collection is being configured, not yet on-chain |
| `published` | Collection has been published to the blockchain |

### TokenEditionStatusEnum

Defines the lifecycle state of an individual token edition.

| Value | Description |
|-------|-------------|
| `importError` | Error occurred during XLSX import |
| `draftError` | Error occurred during draft sync or validation |
| `draft` | Edition created, awaiting configuration |
| `lockedForBuy` | Edition locked for a commerce purchase flow |
| `readyToMint` | Edition configured and ready to be minted on-chain |
| `minting` | Minting transaction submitted, awaiting confirmation |
| `minted` | Successfully minted on-chain |
| `burning` | Burn transaction submitted, awaiting confirmation |
| `burned` | Successfully burned on-chain |
| `burnFailure` | Burn transaction failed |
| `transferring` | Transfer transaction submitted, awaiting confirmation |
| `transferred` | Successfully transferred to new owner |
| `transferFailure` | Transfer transaction failed |

### PublishValidationsEnum

Validation error codes returned when publish validation fails.

| Value | Description |
|-------|-------------|
| `IS_REQUIRED` | A required field is missing |
| `GREATER_THAN_ZERO` | Value must be greater than zero |
| `LESS_THAN_QUANTITY` | Value must be less than total quantity |
| `INVALID_RANGE` | Range format is invalid |
| `INVALID_VALUE` | Generic invalid value |
| `INVALID_STRING` | Value is not a valid string |
| `INVALID_DATE` | Value is not a valid date |
| `INVALID_YEAR` | Year value is invalid |
| `INVALID_NUMBER` | Value is not a valid number |
| `INVALID_URL` | Value is not a valid URL |
| `INVALID_2D_DIMENSIONS` | 2D dimension format is invalid |
| `INVALID_3D_DIMENSIONS` | 3D dimension format is invalid |
| `INVALID_ARRAY` | Value is not a valid array |
| `INVALID_BOOLEAN` | Value is not a valid boolean |
| `INVALID_TYPE` | Value type does not match expected type |
| `CONTRACT_HAS_NO_ADDRESS` | Contract has no deployed address |
| `CONTRACT_NO_ENABLE_MINTER` | Contract does not have ADMIN_MINTER feature enabled |
| `COMPANY_HAS_NO_DEFAULT_OWNER_ADDRESS` | Company has no default owner address configured |
| `RFID_HAS_ALREADY_BEEN_USED` | RFID is already assigned to another active edition |
| `RFID_HAS_ITEMS_DUPLICATED_ON_LIST` | Duplicate RFIDs found in the submitted list |
| `RFID_USED` | RFID is in use |
| `JOB_RUNNING` | An async job is still running for this collection |
| `EDITIONS_WITH_ERRORS` | Some editions have error status (DRAFT_ERROR or IMPORT_ERROR) |
| `EDITIONS_NOT_SYNC` | Editions have not been synced (sync-draft-tokens not called or not complete) |
| `YEAR_OUT_OF_RANGE` | Year value is outside acceptable range |

---

## Entities & Relationships

```
TokenCollection (1) ---> (N) TokenEdition
    |
    +--- status: TokenCollectionStatus
    +--- companyId: uuid
    +--- contractId?: uuid (links to Contract entity from Contracts module)
    +--- subcategoryId?: uuid (defines the token metadata template)
    +--- name: string
    +--- description?: string
    +--- mainImage?: string (URL)
    +--- tokenData?: JSON (metadata matching subcategory template)
    +--- publishedTokenTemplate?: JSON (snapshot of template at publish time)
    +--- quantity: number (total editions)
    +--- initialQuantityToMint: number
    +--- initialQuantity: number
    +--- quantityMinted: number
    +--- rfids: string[] (list of RFIDs assigned to editions)
    +--- ownerAddress?: string (default owner wallet)
    +--- similarTokens: boolean (all editions share same metadata)
    +--- pass: boolean (whether this collection acts as a token-pass)
    +--- settings?: JSON (webhook config, etc.)
    +--- rangeInitialToMint?: string (e.g., "1-50" or "1,5,10")

TokenEdition:
    +--- editionNumber: number (sequential within collection)
    +--- tokenCollectionId: uuid
    +--- companyId: uuid
    +--- contractId?: uuid
    +--- status: TokenEditionStatusEnum
    +--- rfid?: string (globally unique across active editions)
    +--- contractAddress?: string
    +--- ownerAddress?: string
    +--- chainId?: number
    +--- tokenId?: number (on-chain token ID)
    +--- mintedHash?: string (transaction hash)
    +--- mintedAt?: datetime
    +--- nftMintingId?: uuid
    +--- name?: string
    +--- description?: string
    +--- mainImage?: string (URL)
    +--- tokenData?: JSON
    +--- rawData?: JSON
    +--- errorFields?: JSON (validation errors from publish or import)
```

---

## DTOs

### CreateTokenCollectionDto

Used when creating a new collection draft.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | -- | Collection display name |
| `description` | string | No | null | Collection description |
| `mainImage` | string | No | null | URL to main image |
| `contractId` | uuid | No | null | Contract to associate (can be set later) |
| `subcategoryId` | uuid | No | null | Metadata template subcategory |
| `quantity` | number | No | 0 | Total number of editions |
| `initialQuantityToMint` | number | No | 0 | How many editions to mint on publish |
| `tokenData` | object | No | null | Metadata matching subcategory template |
| `similarTokens` | boolean | No | true | Whether all editions share same metadata |
| `ownerAddress` | string | No | null | Default owner wallet address |
| `rangeInitialToMint` | string | No | null | Range of editions to mint (e.g., "1-50") |

### UpdateTokenCollectionDto

Used when editing a draft collection.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Collection display name |
| `description` | string | No | Collection description |
| `mainImage` | string | No | URL to main image |
| `contractId` | uuid | No | Contract to associate |
| `subcategoryId` | uuid | No | Metadata template subcategory |
| `quantity` | number | No | Total number of editions |
| `initialQuantityToMint` | number | No | How many editions to mint on publish |
| `tokenData` | object | No | Metadata matching subcategory template |
| `similarTokens` | boolean | No | Whether all editions share same metadata |
| `ownerAddress` | string | No | Default owner wallet address |
| `rangeInitialToMint` | string | No | Range of editions to mint (e.g., "1-50") |
| `rfids` | string[] | No | List of RFIDs to assign |
| `settings` | object | No | Collection settings (webhook config) |

### EstimateCollectionGasDto

Used for gas estimation on collection publish.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contractId` | uuid | Yes | Contract ID for the collection |
| `initialQuantityToMint` | number | Yes | Number of tokens to mint |

### ChangeStatusReadyToMintDto

Used to mark editions as ready to mint (bulk).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection ID |
| `editionNumbers` | number[] | Yes | Edition numbers to mark ready |

### MintOnDemandDto

Used to mint specific editions on demand.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection ID |
| `editionNumbers` | number[] | Yes | Edition numbers to mint |
| `ownerAddress` | string | Yes | Wallet address to receive the minted tokens |

### UpdateTokenMetadataDto

Used to update metadata on minted editions.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection ID |
| `editionNumbers` | number[] | Yes | Edition numbers to update |
| `tokenData` | object | Yes | New metadata values |
| `mergeTokenData` | boolean | No | If true, merges with existing data; if false, replaces entirely |
| `keepStatus` | boolean | No | If true, keeps current status; if false, resets to draft |

### BurnTokensDto

Used to burn one or more editions.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokens` | object[] | Yes | Array of tokens to burn |
| `tokens[].tokenCollectionId` | uuid | Yes | Collection ID |
| `tokens[].editionNumber` | number | Yes | Edition number to burn |

### TransferTokensDto

Used for bulk token transfer (admin).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection ID |
| `editionNumbers` | number[] | Yes | Edition numbers to transfer |
| `toAddress` | string | Yes | Destination wallet address |

### TransferTokenByEditionDto

Used for single edition transfer.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `toAddress` | string | Yes | Destination wallet address |

### ChangeRfidDto

Used to assign an RFID to an edition.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rfid` | string | Yes | RFID value to assign (must be globally unique) |

### EstimateMintGasDto

Query parameters for mint gas estimation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection ID |

### EstimateTransferGasDto

Query parameters for transfer gas estimation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `toAddress` | string | Yes | Destination wallet address |

### EstimateBurnGasDto

No additional parameters beyond the edition ID in the path.

---

## Endpoints

### Token Collection CRUD

#### POST /{companyId}/token-collections

Create a new token collection in DRAFT status.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/{companyId}/token-collections` | Bearer (Admin) | 201 Created |

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
    "name": "Asset #",
    "description": "Unique digital asset",
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
    "name": "Asset #",
    "description": "Unique digital asset",
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
- `contractId` is optional at creation but required before publish
- `subcategoryId` defines the metadata template for validation
- `quantity` defaults to 0 if not provided
- `similarTokens` defaults to true (all editions share the same tokenData)

---

#### GET /{companyId}/token-collections

List all token collections for a company with pagination.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-collections` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 10) |
| `search` | string | No | Filter by name (partial match) |
| `status` | string | No | Filter by status (`draft` or `published`) |
| `orderBy` | string | No | `ASC` or `DESC` |
| `sortBy` | string | No | Field to sort by |

**Response (200):**

```json
{
  "items": [
    {
      "id": "collection-uuid",
      "status": "draft",
      "companyId": "company-uuid",
      "name": "My NFT Collection",
      "quantity": 100,
      "quantityMinted": 0,
      "similarTokens": true,
      "pass": false,
      "createdAt": "2026-04-02T12:00:00.000Z",
      "updatedAt": "2026-04-02T12:00:00.000Z"
    }
  ],
  "meta": {
    "totalItems": 1,
    "itemCount": 1,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

---

#### GET /{companyId}/token-collections/{id}

Get a single token collection by ID.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-collections/{id}` | Bearer (Admin) | 200 OK |

**Response (200):**

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
    "name": "Asset #",
    "description": "Unique digital asset",
    "image": "https://example.com/asset.png"
  },
  "publishedTokenTemplate": null,
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

---

#### PUT /{companyId}/token-collections/{id}

Edit a draft collection. Only allowed when status is DRAFT.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PUT | `/{companyId}/token-collections/{id}` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "name": "Updated Collection Name",
  "description": "Updated description",
  "quantity": 200,
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "tokenData": {
    "name": "Updated Asset",
    "description": "Updated description"
  },
  "rangeInitialToMint": "1-100"
}
```

**Response (200):** Returns the updated TokenCollectionEntity.

**Notes:**
- Only allowed when collection status is `draft`
- Changing `quantity` may require re-syncing editions

---

### Token Collection Lifecycle

#### PATCH /{companyId}/token-collections/{id}/sync-draft-tokens

Generate or re-sync edition drafts for a collection. This is an async operation.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-collections/{id}/sync-draft-tokens` | Bearer (Admin) | 200 OK |

**Response (200):**

```json
{
  "jobId": "job-uuid"
}
```

**Notes:**
- Creates `quantity` editions with sequential `editionNumber` starting from 1
- This is an async job -- poll the job status endpoint for completion
- Must complete before publish is allowed
- If editions already exist, re-syncs them (adds missing, removes excess)

---

#### PATCH /{companyId}/token-collections/publish/{id}

Publish a draft collection to the blockchain.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-collections/publish/{id}` | Bearer (Admin) | 200 OK |

**Response (200):**

```json
{
  "id": "collection-uuid",
  "status": "published",
  "companyId": "company-uuid",
  "contractId": "contract-uuid",
  "name": "My NFT Collection",
  "quantity": 100,
  "quantityMinted": 0,
  "publishedTokenTemplate": {
    "fields": []
  },
  "createdAt": "2026-04-02T12:00:00.000Z",
  "updatedAt": "2026-04-02T12:30:00.000Z"
}
```

**Validation Checks (all must pass):**
- Contract must be in PUBLISHED status
- Contract must have ADMIN_MINTER feature enabled
- All editions must be in DRAFT status (no error states)
- Edition count must match collection quantity
- If `similarTokens` is true, tokenData is validated against the subcategory template
- No async jobs running for this collection
- RFIDs must be unique (no duplicates in list, no conflicts with other collections)
- Company must have a default owner address if `ownerAddress` is not set

**Notes:**
- On publish, if `similarTokens` is true, the collection's tokenData is applied to all editions
- Editions in the `rangeInitialToMint` range are set to READY_TO_MINT status
- The subcategory template is snapshot as `publishedTokenTemplate`
- Status changes from `draft` to `published` (irreversible)

---

#### DELETE /{companyId}/token-collections/{id}/draft

Delete a draft collection and all its editions.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| DELETE | `/{companyId}/token-collections/{id}/draft` | Bearer (Admin) | 204 No Content |

**Notes:**
- Only allowed when status is `draft`
- Deletes all associated edition drafts

---

#### DELETE /{companyId}/token-collections/{id}/burn

Burn a published collection and all its minted editions.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| DELETE | `/{companyId}/token-collections/{id}/burn` | Bearer (Admin) | 200 OK |

**Notes:**
- Only allowed when status is `published`
- Initiates burn transactions for all minted editions on-chain

---

#### PATCH /{companyId}/token-collections/{id}/increase-editions

Increase the number of editions in a published collection.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-collections/{id}/increase-editions` | Bearer (Integration) | 200 OK |

**Request Body:**

```json
{
  "quantity": 50
}
```

**Notes:**
- Only allowed when status is `published`
- Adds additional editions starting from the next available edition number
- Requires Integration role

---

#### PATCH /{companyId}/token-collections/{id}/pass/enable

Mark a collection as a token-pass.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-collections/{id}/pass/enable` | Bearer (Integration) | 200 OK |

**Notes:**
- Sets `pass` to `true` on the collection
- Token-passes have special behavior in the commerce module

---

#### PATCH /{companyId}/token-collections/{id}/pass/disable

Unmark a collection as a token-pass.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-collections/{id}/pass/disable` | Bearer (Integration) | 200 OK |

---

#### GET /{companyId}/token-collections/estimate-gas

Estimate gas costs for publishing a collection.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-collections/estimate-gas` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `contractId` | uuid | Yes | Contract ID |
| `initialQuantityToMint` | number | Yes | Number of tokens to mint |

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

---

### Token Collection Import/Export

#### GET /{companyId}/token-collections/{id}/export/xlsx

Export an Excel template with current edition data.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-collections/{id}/export/xlsx` | Bearer (Admin) | 200 OK (file download) |

**Response:** Binary XLSX file download.

**Notes:**
- Template columns include: editionNumber, name, description, rfid, plus custom fields from the subcategory template
- Used as a starting point for bulk import

---

#### POST /{companyId}/token-collections/{id}/import/xlsx

Import editions from an Excel file.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/{companyId}/token-collections/{id}/import/xlsx` | Bearer (Admin) | 200 OK |

**Request:** `multipart/form-data` with file field.

**Response (200):**

```json
{
  "jobId": "job-uuid"
}
```

**Notes:**
- Async operation -- poll job status for completion
- File must follow the exported template structure
- Editions with errors receive IMPORT_ERROR status
- Collection must be in DRAFT status

---

### Token Edition CRUD

#### GET /{companyId}/token-editions

List editions with pagination and filtering.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 10) |
| `tokenCollectionId` | uuid | No | Filter by collection |
| `status` | string | No | Filter by edition status |
| `search` | string | No | Search by name or edition number |
| `orderBy` | string | No | `ASC` or `DESC` |
| `sortBy` | string | No | Field to sort by |

**Response (200):**

```json
{
  "items": [
    {
      "id": "edition-uuid",
      "editionNumber": 1,
      "tokenCollectionId": "collection-uuid",
      "companyId": "company-uuid",
      "status": "draft",
      "rfid": null,
      "name": "Asset #1",
      "description": "Unique digital asset",
      "mainImage": "https://example.com/asset-1.png",
      "tokenData": {},
      "errorFields": null,
      "createdAt": "2026-04-02T12:00:00.000Z",
      "updatedAt": "2026-04-02T12:00:00.000Z"
    }
  ],
  "meta": {
    "totalItems": 100,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 10,
    "currentPage": 1
  }
}
```

---

#### GET /{companyId}/token-editions/{id}

Get a single edition by ID.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/{id}` | Bearer (Admin) | 200 OK |

**Response (200):**

```json
{
  "id": "edition-uuid",
  "editionNumber": 1,
  "tokenCollectionId": "collection-uuid",
  "companyId": "company-uuid",
  "contractId": "contract-uuid",
  "status": "minted",
  "rfid": "RFID-001",
  "contractAddress": "0xabcdef1234567890abcdef1234567890abcdef12",
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "chainId": 137,
  "tokenId": 1,
  "mintedHash": "0xabc123...",
  "mintedAt": "2026-04-02T12:30:00.000Z",
  "nftMintingId": "minting-uuid",
  "name": "Asset #1",
  "description": "Unique digital asset",
  "mainImage": "https://example.com/asset-1.png",
  "tokenData": {
    "name": "Asset #1",
    "description": "Unique digital asset",
    "image": "https://example.com/asset-1.png"
  },
  "rawData": null,
  "errorFields": null,
  "createdAt": "2026-04-02T12:00:00.000Z",
  "updatedAt": "2026-04-02T12:30:00.000Z"
}
```

---

#### PATCH /{companyId}/token-editions/{id}

Edit a single edition.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/{id}` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "name": "Updated Asset #1",
  "description": "Updated description",
  "mainImage": "https://example.com/updated-asset-1.png",
  "tokenData": {
    "name": "Updated Asset #1",
    "rarity": "legendary"
  }
}
```

**Response (200):** Returns the updated TokenEditionEntity.

---

### Token Edition Status Operations

#### PATCH /{companyId}/token-editions/ready-to-mint

Mark multiple editions as ready to mint (bulk).

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/ready-to-mint` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3, 4, 5]
}
```

**Response (200):**

```json
{
  "updated": 5
}
```

---

#### PATCH /{companyId}/token-editions/{id}/ready-to-mint

Mark a single edition as ready to mint.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/{id}/ready-to-mint` | Bearer (Admin) | 200 OK |

**Response (200):** Returns the updated TokenEditionEntity with status `readyToMint`.

---

#### PATCH /{companyId}/token-editions/mint-on-demand

Mint specific editions on demand.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/mint-on-demand` | Bearer (Admin/Integration) | 200 OK |

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
- Editions must be in READY_TO_MINT or DRAFT status
- Sets editions to MINTING status and submits blockchain transactions
- Collection must be PUBLISHED

---

#### PATCH /{companyId}/token-editions/locked-for-buy

Lock editions for a commerce purchase flow.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/locked-for-buy` | Bearer (Admin/Integration) | 200 OK |

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
- Editions must be in DRAFT or READY_TO_MINT status
- Sets editions to LOCKED_FOR_BUY status
- Prevents other operations on these editions until unlocked

---

#### PATCH /{companyId}/token-editions/unlocked-for-buy

Unlock editions previously locked for purchase.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/unlocked-for-buy` | Bearer (Admin/Integration) | 200 OK |

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
- Returns editions to DRAFT status

---

#### PATCH /{companyId}/token-editions/notify-externally-minted

Notify the system that editions were minted externally (outside the platform).

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/notify-externally-minted` | Bearer (Admin/Integration) | 200 OK |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "contractAddress": "0xabcdef1234567890abcdef1234567890abcdef12",
  "chainId": 137
}
```

**Notes:**
- Marks editions as MINTED without the platform executing the minting transaction
- Used for integrations where minting happens on an external system

---

### Token Edition Metadata

#### PATCH /{companyId}/token-editions/update-token-metadata

Update metadata on one or more editions. Optionally triggers a webhook.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/update-token-metadata` | Bearer (Admin/Integration) | 200 OK |

**Minimal Request:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1],
  "tokenData": {
    "name": "Updated Name"
  }
}
```

**Complete Request:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "tokenData": {
    "name": "Updated Name",
    "description": "Updated description",
    "image": "https://example.com/updated.png",
    "attributes": [
      { "trait_type": "rarity", "value": "legendary" }
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

**Notes:**
- `mergeTokenData`: when true, merges the provided tokenData with existing data (shallow merge); when false, replaces entirely
- `keepStatus`: when true, keeps current edition status; when false, may reset to draft
- If `collection.settings.sendWebhookWhenTokenEditionIsUpdated` is enabled, a webhook notification is sent after the update

---

### Token Edition RFID

#### GET /{companyId}/token-editions/check-rfid

Check if an RFID is available for assignment.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/check-rfid` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `rfid` | string | Yes | RFID value to check |

**Response (200):**

```json
{
  "available": true
}
```

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

**Notes:**
- RFIDs are globally unique across all active editions (excluding deleted and burned)

---

#### PATCH /{companyId}/token-editions/{id}/rfid

Assign an RFID to a specific edition.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/{id}/rfid` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "rfid": "RFID-001-ABC"
}
```

**Response (200):** Returns the updated TokenEditionEntity with the RFID assigned.

**Notes:**
- The RFID must be globally unique (check with `check-rfid` first)
- Assigning a new RFID to an edition that already has one will replace the existing RFID

---

### Token Edition Transfer

#### PATCH /{companyId}/token-editions/transfer-token

Transfer multiple editions to a wallet address (admin bulk transfer).

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/transfer-token` | Bearer (Admin) | 200 OK |

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

**Notes:**
- Editions must be in MINTED or TRANSFERRED status
- Sets editions to TRANSFERRING status and submits blockchain transactions

---

#### PATCH /{companyId}/token-editions/{id}/transfer-token

Transfer a single edition to a wallet address.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/{id}/transfer-token` | Bearer (Admin/User) | 200 OK |

**Request Body:**

```json
{
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef"
}
```

**Response (200):** Returns the updated TokenEditionEntity with status `transferring`.

**Notes:**
- Users can transfer their own tokens (owner must match the authenticated user's wallet)

---

#### PATCH /{companyId}/token-editions/{id}/transfer-token/email

Transfer a single edition to a user by email address.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/{id}/transfer-token/email` | Bearer (Admin/User) | 200 OK |

**Request Body:**

```json
{
  "email": "recipient@example.com"
}
```

**Response (200):** Returns the updated TokenEditionEntity with status `transferring`.

**Notes:**
- The system resolves the email to a wallet address
- If the email is not associated with a user, the transfer may be deferred until the user registers

---

### Token Edition Burn

#### DELETE /{companyId}/token-editions/burn

Burn one or more editions.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| DELETE | `/{companyId}/token-editions/burn` | Bearer (Admin/User) | 200 OK |

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
- Sets editions to BURNING status and submits blockchain burn transactions
- Users can burn their own tokens (owner must match the authenticated user's wallet)

---

### Token Edition Gas Estimation

#### GET /{companyId}/token-editions/{id}/estimate-gas/mint

Estimate gas cost for minting a specific edition.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/mint` | Bearer (Admin) | 200 OK |

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

---

#### GET /{companyId}/token-editions/{id}/estimate-gas/transfer

Estimate gas cost for transferring a specific edition.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/transfer` | Bearer (Admin/User) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `toAddress` | string | Yes | Destination wallet address |

**Response (200):** Same structure as mint gas estimation (slow/standard/fast tiers).

---

#### GET /{companyId}/token-editions/{id}/estimate-gas/burn

Estimate gas cost for burning a specific edition.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/burn` | Bearer (Admin/User) | 200 OK |

**Response (200):** Same structure as mint gas estimation (slow/standard/fast tiers).

---

### Token Edition Transaction History

#### GET /{companyId}/token-editions/{id}/get-last/{type}

Get the last transaction of a given type for an edition.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/{id}/get-last/{type}` | Bearer (Admin) | 200 OK |

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | Yes | Transaction type: `transfer` or `burn` |

**Response (200):**

```json
{
  "id": "transaction-uuid",
  "type": "transfer",
  "transactionHash": "0xabc123...",
  "fromAddress": "0x1234...",
  "toAddress": "0x5678...",
  "status": "confirmed",
  "createdAt": "2026-04-02T14:00:00.000Z"
}
```

---

### Token Edition Async Export

#### GET /{companyId}/token-editions/xls

Request an async export of editions as an Excel file.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/{companyId}/token-editions/xls` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection to export |

**Response (200):**

```json
{
  "jobId": "job-uuid"
}
```

**Notes:**
- Async operation -- poll job status for the download URL
- Useful for large collections where synchronous export would timeout

---

### Token Edition Retry

#### PATCH /{companyId}/token-editions/retry-bulk-by-collection

Retry failed bulk operations for all editions in a collection.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/{companyId}/token-editions/retry-bulk-by-collection` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid"
}
```

**Response (200):**

```json
{
  "retried": 5
}
```

**Notes:**
- Retries editions with error statuses (DRAFT_ERROR, IMPORT_ERROR, BURN_FAILURE, TRANSFER_FAILURE)
- Resets their status to the appropriate previous state and re-submits the operation

---

## Error Reference

| Exception | HTTP Status | Cause |
|-----------|-------------|-------|
| `TokenCollectionNotFoundException` | 404 | Collection ID not found for the given company |
| `TokenCollectionNotDraftException` | 400 | Operation requires DRAFT status but collection is PUBLISHED |
| `TokenCollectionNotPublishedException` | 400 | Operation requires PUBLISHED status but collection is DRAFT |
| `TokenCollectionPublishValidationException` | 400 | Publish validation failed (see PublishValidationsEnum for details) |
| `TokenEditionNotFoundException` | 404 | Edition ID not found for the given company |
| `TokenEditionInvalidStatusException` | 400 | Edition is not in the required status for the operation |
| `RfidAlreadyUsedException` | 400 | RFID is already assigned to another active edition |
| `RfidDuplicateException` | 400 | Duplicate RFIDs found in the submitted list |
| `ContractNotFoundException` | 404 | Contract ID not found |
| `ContractNotPublishedException` | 400 | Contract is not in PUBLISHED status |
| `ContractNoMinterException` | 400 | Contract does not have ADMIN_MINTER feature enabled |
| `JobRunningException` | 400 | An async job is still running for this collection |
| `InvalidRangeException` | 400 | rangeInitialToMint format is invalid |
| `GasEstimationException` | 400 | Gas estimation failed (contract or network issue) |
