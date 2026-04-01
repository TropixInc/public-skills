---
id: WHITELIST_API_REFERENCE
title: "Whitelist - API Reference"
module: offpix/whitelist
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - whitelist
  - user-groups
  - entries
  - access-control
  - api-reference
---

# Whitelist API Reference

Complete endpoint reference for the W3Block Whitelist module. This module manages two entities -- Whitelists (named access groups) and Whitelist Entries (individual access rules within a group) -- served by the PixwayID backend.

> **Frontend naming:** The Offpix frontend refers to whitelists as **"User Groups"** throughout the UI.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://pixwayid.w3block.io` |
| Staging | `https://pixwayid.stg.w3block.io` |
| Swagger | https://pixwayid.w3block.io/docs/ |

## Authentication

All endpoints require:

```
Authorization: Bearer {accessToken}
```

All whitelist endpoints require the **Admin** role by default. The `check-user` endpoints also accept self-user requests (a user checking their own access).

---

## Enums

### WhitelistEntryType

Defines the type of access rule for a whitelist entry. The `value` field meaning changes based on the type.

| Value | Frontend Label | Description | `value` field contains |
|-------|---------------|-------------|----------------------|
| `user_id` | User | Direct user access | User UUID |
| `email` | Email | Email-based access | Email address |
| `wallet_address` | Wallet | Blockchain wallet access | Wallet address (hex) |
| `collection_holder` | Contract | External NFT collection holder | Contract address (hex) |
| `kyc_approved_context` | -- | Access for users with approved KYC in a specific context | Context UUID |
| `key_collection_holder` | Collection NFT (disabled in UI) | W3Block KEY platform collection holder | Collection UUID |
| `key_erc20_holder` | -- | W3Block KEY platform ERC-20 token holder | ERC-20 contract UUID |

### WhitelistEntriesSortBy

| Value | Description |
|-------|-------------|
| `createdAt` | Sort by creation date (default) |
| `updatedAt` | Sort by last update date |

---

## Entities & Relationships

```
Whitelist (1) ---> (N) WhitelistEntry
    |
    +--- name: string (required)
    +--- tenantId: uuid
    +--- walletGroups: WalletGroupEntity[] (on-chain promotions)

WhitelistEntry:
    +--- type: WhitelistEntryType (required)
    +--- value: string (required, lowercased on save)
    +--- additionalData: JSON (required for holder types, null otherwise)
```

**Uniqueness constraint:** The combination `(whitelistId, type, value)` is unique, **except** for `collection_holder` and `key_collection_holder` types which allow duplicate values (to support different `additionalData` configurations such as different token ID ranges).

### AdditionalData Schemas

#### CollectionHolderAdditionalData (type: `collection_holder`)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `chainId` | number | Yes | Blockchain chain ID (e.g., 80001 for Mumbai, 137 for Polygon) |
| `startTokenId` | number | No | Start of token ID range (min: 1) |
| `endTokenId` | number | No | End of token ID range (min: 1) |
| `metadataRequirements` | object[] | No | Array of key-value pairs that token metadata must match |

#### KeyCollectionTokenHolderAdditionalData (type: `key_collection_holder`)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | No | Minimum number of tokens required (min: 0) |
| `metadataRequirements` | object[] | No | Array of key-value pairs that token metadata must match |

#### KeyErc20HolderAdditionalData (type: `key_erc20_holder`)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | No | Minimum token balance required (min: 0) |

---

## Endpoints

### Whitelist CRUD

#### POST /whitelists/{tenantId}

Create a new whitelist.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/whitelists/{tenantId}` | Bearer (Admin) | 201 Created |

**Request Body:**

```json
{
  "name": "VIP Members"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name for the whitelist |

**Response (201):**

```json
{
  "id": "a1b2c3d4-...",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

---

#### GET /whitelists/{tenantId}

List all whitelists for a tenant with pagination.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/whitelists/{tenantId}` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 10) |
| `search` | string | No | Filter by name (partial match) |
| `orderBy` | string | No | `ASC` or `DESC` |
| `sortBy` | string | No | Field to sort by |

**Response (200):**

```json
{
  "items": [
    {
      "id": "a1b2c3d4-...",
      "tenantId": "tenant-uuid",
      "name": "VIP Members",
      "createdAt": "2026-04-01T00:00:00.000Z",
      "updatedAt": "2026-04-01T00:00:00.000Z"
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

#### GET /whitelists/{tenantId}/{id}

Get a single whitelist by ID, including its wallet groups (on-chain promotions).

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/whitelists/{tenantId}/{id}` | Bearer (Admin) | 200 OK |

**Response (200):**

```json
{
  "id": "a1b2c3d4-...",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z",
  "walletGroups": []
}
```

---

#### PATCH /whitelists/{tenantId}/{id}

Update a whitelist name.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/whitelists/{tenantId}/{id}` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "name": "Updated Name"
}
```

**Response (200):** Same as GET by ID response with updated values.

---

#### DELETE /whitelists/{tenantId}/{id}

Soft-delete a whitelist.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| DELETE | `/whitelists/{tenantId}/{id}` | Bearer (Admin) | 204 No Content |

**Notes:**
- This is a soft delete (the record is not physically removed)
- Entries belonging to this whitelist are not explicitly deleted but become inaccessible

---

### Whitelist Entries

#### POST /whitelists/{tenantId}/{id}/entries

Add an entry to a whitelist.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| POST | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) | 201 Created |

**Minimal Request (email type):**

```json
{
  "type": "email",
  "value": "user@example.com"
}
```

**Complete Request (collection holder with additionalData):**

```json
{
  "type": "collection_holder",
  "value": "0xd3304183ec1fa687e380b67419875f97f1db05f5",
  "additionalData": {
    "chainId": 137,
    "startTokenId": 1,
    "endTokenId": 100,
    "metadataRequirements": [
      { "trait_type": "rarity", "value": "legendary" }
    ]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | WhitelistEntryType | Yes | Type of entry |
| `value` | string | Yes | The access identifier (email, address, UUID, etc.) -- validated per type |
| `additionalData` | object | Conditional | Required for `collection_holder`, `key_collection_holder`, `key_erc20_holder`. See AdditionalData Schemas above |

**Response (201):**

```json
{
  "id": "entry-uuid",
  "whitelistId": "a1b2c3d4-...",
  "type": "email",
  "value": "user@example.com",
  "additionalData": null,
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

**Notes:**
- The `value` field is **lowercased** on save (emails, wallet addresses, UUIDs are all stored lowercase)
- The `value` field is validated based on `type` using `IsValidWhitelistValueValidator`
- For `user_id` type, the service verifies the user exists and belongs to the same tenant

---

#### GET /whitelists/{tenantId}/{id}/entries

List entries for a whitelist with pagination and filtering.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 10) |
| `search` | string | No | Search by value; also searches matching user names/emails if >= 5 chars |
| `type` | WhitelistEntryType[] | No | Filter by entry type(s) |
| `sortBy` | WhitelistEntriesSortBy | No | Sort field (default: `createdAt`) |
| `orderBy` | string | No | `ASC` or `DESC` |
| `showWallets` | boolean | No | If `true`, includes resolved wallet addresses for email/user_id entries |

**Response (200):**

```json
{
  "items": [
    {
      "id": "entry-uuid",
      "whitelistId": "a1b2c3d4-...",
      "type": "email",
      "value": "user@example.com",
      "additionalData": null,
      "createdAt": "2026-04-01T00:00:00.000Z",
      "updatedAt": "2026-04-01T00:00:00.000Z",
      "wallets": []
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

**Notes:**
- When `search` is provided with >= 5 characters, the service also searches users by name/email and includes `user_id` type matches for found users
- The `wallets` field is only populated when `showWallets=true`

---

#### DELETE /whitelists/{tenantId}/{id}/entries/{entryId}

Remove an entry from a whitelist.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| DELETE | `/whitelists/{tenantId}/{id}/entries/{entryId}` | Bearer (Admin) | 204 No Content |

**Notes:**
- This is a soft delete
- If the whitelist has been promoted on-chain (has wallet groups), the entry's resolved wallet addresses are also removed from the associated wallet groups

---

### Access Check

#### GET /whitelists/{tenantId}/{id}/check-user

Check if a specific user has access through a single whitelist.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/whitelists/{tenantId}/{id}/check-user` | Bearer (Admin / Self) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | uuid | Yes | User ID to check |
| `disableCache` | boolean | No | Bypass the 120-second cache (default: false) |

**Response (200):**

```json
{
  "whitelistId": "a1b2c3d4-...",
  "userId": "user-uuid",
  "hasAccess": true
}
```

**Notes:**
- Results are cached for 120 seconds by default
- A user can check their own access (via `AcceptSelfUserRequest` decorator)
- The check evaluates all entry types: direct matches (user_id, email, wallet_address, kyc_approved_context) and holder-based matches (collection_holder, key_collection_holder, key_erc20_holder)
- Throws `WhitelistCheckException` if no check details are returned

---

#### GET /whitelists/{tenantId}/check-user

Check if a user has access through multiple whitelists at once.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/whitelists/{tenantId}/check-user` | Bearer (Admin / Self) | 200 OK |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | uuid | Yes | User ID to check |
| `whitelistsIds` | uuid[] | Yes | Array of whitelist IDs to check against |
| `disableCache` | boolean | No | Bypass the 120-second cache (default: false) |

**Response (200):**

```json
{
  "hasAccess": true,
  "details": [
    {
      "whitelistId": "whitelist-uuid-1",
      "userId": "user-uuid",
      "hasAccess": true
    },
    {
      "whitelistId": "whitelist-uuid-2",
      "userId": "user-uuid",
      "hasAccess": false
    }
  ]
}
```

**Notes:**
- `hasAccess` at the top level is `true` if the user has access to **any** of the listed whitelists
- `details` provides per-whitelist access results
- Same caching behavior as the single-whitelist check (120 seconds TTL)

---

### On-Chain Promotion

#### PATCH /whitelists/{tenantId}/{id}/promote-on-chain

Promote a whitelist to an on-chain wallet group, enabling blockchain-level access control.

| Method | Path | Auth | Response |
|--------|------|------|----------|
| PATCH | `/whitelists/{tenantId}/{id}/promote-on-chain` | Bearer (Admin) | 200 OK |

**Request Body:**

```json
{
  "chainId": 137
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `chainId` | number | Yes | Target blockchain chain ID |

**Response (200):**

Returns a `WalletGroupResponseDto` object representing the created on-chain wallet group.

**Notes:**
- The tenant must have an `operatorAddress` configured
- A whitelist can only be promoted once per `chainId`; attempting to promote again to the same chain throws `WhitelistAlreadyOnChainException`
- After creation, the wallet group is queued for sync and publish to the blockchain
- Deleting entries from a promoted whitelist also removes the corresponding wallets from the on-chain wallet group

---

## Error Reference

| Exception | HTTP Status | Cause |
|-----------|-------------|-------|
| `WhitelistNotFoundException` | 404 | Whitelist ID not found for the given tenant |
| `WhitelistDeleteException` | 400 | Soft-delete operation affected 0 rows |
| `WhitelistEntrySaveException` | 400 | Duplicate entry (same whitelistId + type + value) or unknown save error |
| `WhitelistEntryDeleteException` | 400 | Soft-delete of entry affected 0 rows |
| `WhitelistEntryNotFoundException` | 400 | Entry not found in the specified whitelist/tenant |
| `WhitelistEntryUserNotFoundException` | 400 | User ID does not exist or belongs to a different tenant |
| `WhitelistAlreadyOnChainException` | 400 | Whitelist already has a wallet group for the specified chainId |
| `WhitelistOnChainException` | 400 | Failed to create wallet group during on-chain promotion |
| `WhitelistCheckException` | 400 | Error during user access check (no details returned) |
