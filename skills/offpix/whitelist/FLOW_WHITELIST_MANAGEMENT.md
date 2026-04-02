---
id: FLOW_WHITELIST_MANAGEMENT
title: "Whitelist - Management (Create, Entries, Access Check, On-Chain)"
module: offpix/whitelist
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - whitelist
  - user-groups
  - entries
  - access-control
  - on-chain
depends_on:
  - WHITELIST_API_REFERENCE
---

# Whitelist Management

## Overview

This flow covers the full lifecycle of whitelists (called **"User Groups"** in the frontend): creating a whitelist, adding entries of various types (email, wallet, user ID, contract holder), listing and filtering entries, checking user access, and optionally promoting a whitelist on-chain. Whitelists are used to gate access to products, promotions, and blockchain operations within a tenant.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |

---

## Flow: Create a Whitelist

### Step 1: Create the Whitelist

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| POST | `/whitelists/{tenantId}` | Bearer (Admin) |

**Minimal Request:**

```json
{
  "name": "VIP Members"
}
```

**Response (201):**

```json
{
  "id": "a1b2c3d4-5678-9abc-def0-123456789abc",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T12:00:00.000Z",
  "updatedAt": "2026-04-01T12:00:00.000Z"
}
```

**Notes:**
- The frontend redirects to the edit page after creation, using the returned `id`
- Only the `name` field is accepted; `tenantId` is extracted from the URL path

---

## Flow: Edit a Whitelist

### Step 1: Get Current Whitelist

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/whitelists/{tenantId}/{id}` | Bearer (Admin) |

**Response (200):**

```json
{
  "id": "a1b2c3d4-...",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T12:00:00.000Z",
  "updatedAt": "2026-04-01T12:00:00.000Z",
  "walletGroups": []
}
```

### Step 2: Update the Name

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/whitelists/{tenantId}/{id}` | Bearer (Admin) |

**Request Body:**

```json
{
  "name": "Updated VIP Members"
}
```

**Response (200):** Returns the updated whitelist object.

---

## Flow: Add Entries to a Whitelist

After creating a whitelist, add entries to define who has access. The frontend presents four entry type options (with a fifth, "Collection NFT", currently disabled).

### Step 1: Add an Entry

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| POST | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) |

Choose the appropriate payload based on entry type:

**Email entry (minimal):**

```json
{
  "type": "email",
  "value": "user@example.com"
}
```

**Wallet address entry:**

```json
{
  "type": "wallet_address",
  "value": "0xd3304183ec1fa687e380b67419875f97f1db05f5"
}
```

**User ID entry:**

```json
{
  "type": "user_id",
  "value": "user-uuid-here"
}
```

**Collection holder entry (complete with additionalData):**

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

**KEY collection holder entry:**

```json
{
  "type": "key_collection_holder",
  "value": "collection-uuid",
  "additionalData": {
    "amount": 1,
    "metadataRequirements": [
      { "trait_type": "tier", "value": "gold" }
    ]
  }
}
```

**KEY ERC-20 holder entry:**

```json
{
  "type": "key_erc20_holder",
  "value": "erc20-contract-uuid",
  "additionalData": {
    "amount": 100
  }
}
```

**KYC approved context entry:**

```json
{
  "type": "kyc_approved_context",
  "value": "context-uuid"
}
```

**Response (201):**

```json
{
  "id": "entry-uuid",
  "whitelistId": "a1b2c3d4-...",
  "type": "email",
  "value": "user@example.com",
  "additionalData": null,
  "createdAt": "2026-04-01T12:00:00.000Z",
  "updatedAt": "2026-04-01T12:00:00.000Z"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | WhitelistEntryType | Yes | One of: `email`, `wallet_address`, `user_id`, `collection_holder`, `kyc_approved_context`, `key_collection_holder`, `key_erc20_holder` |
| `value` | string | Yes | The identifier -- validated per type. Lowercased on save |
| `additionalData` | object | Conditional | Required for `collection_holder` (must include `chainId`), optional for `key_collection_holder` and `key_erc20_holder` |

**Notes:**
- For `user_id` type, the backend verifies the user exists and belongs to the same tenant before saving
- Duplicate entries (same whitelist + type + value) are rejected with a "duplicate key" error, except for `collection_holder` and `key_collection_holder` which allow duplicates (for different additionalData configurations)
- The frontend invalidates the entry list cache on success

---

## Flow: List and Filter Entries

### Step 1: Fetch Entries with Filters

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) |

**Minimal query:**

```
GET /whitelists/{tenantId}/{id}/entries?page=1&limit=10
```

**Complete query (with filters):**

```
GET /whitelists/{tenantId}/{id}/entries?page=1&limit=10&search=user@example&type=email&type=wallet_address&sortBy=createdAt&orderBy=DESC&showWallets=true
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `search` | string | Searches by entry value. For searches >= 5 chars, also looks up users by name/email and includes matching `user_id` entries |
| `type` | string[] | Filter by one or more entry types. Pass multiple `type` params to filter by several types |
| `sortBy` | string | `createdAt` (default) or `updatedAt` |
| `showWallets` | boolean | When `true`, resolves wallet addresses for `email` and `user_id` entries |

**Notes:**
- The frontend uses a `MultipleSelect` component to filter by type and a text search field
- The search behavior is enhanced: for queries of 5+ characters, it also searches the users table and matches `user_id` entries for found users

---

## Flow: Delete an Entry

### Step 1: Remove the Entry

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| DELETE | `/whitelists/{tenantId}/{id}/entries/{entryId}` | Bearer (Admin) |

No request body required.

**Response:** 204 No Content

**Notes:**
- This is a soft delete
- If the whitelist has been promoted on-chain (has wallet groups), the entry's resolved wallet addresses are removed from the associated wallet groups as part of the same transaction
- The frontend shows a confirmation modal before deleting

---

## Flow: Delete a Whitelist

### Step 1: Delete the Whitelist

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| DELETE | `/whitelists/{tenantId}/{id}` | Bearer (Admin) |

No request body required.

**Response:** 204 No Content

**Notes:**
- This is a soft delete performed within a database transaction
- The frontend invalidates the whitelist list cache on success

---

## Flow: Check User Access

### Single Whitelist Check

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/whitelists/{tenantId}/{id}/check-user?userId={userId}` | Bearer (Admin / Self) |

**Response (200):**

```json
{
  "whitelistId": "a1b2c3d4-...",
  "userId": "user-uuid",
  "hasAccess": true
}
```

### Multiple Whitelists Check

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/whitelists/{tenantId}/check-user?userId={userId}&whitelistsIds={id1}&whitelistsIds={id2}` | Bearer (Admin / Self) |

**Response (200):**

```json
{
  "hasAccess": true,
  "details": [
    { "whitelistId": "id1", "userId": "user-uuid", "hasAccess": true },
    { "whitelistId": "id2", "userId": "user-uuid", "hasAccess": false }
  ]
}
```

**How access is evaluated:**

The check evaluates multiple entry types in this order:

1. **Direct matches:** Searches for entries where the user's ID, email, or wallet addresses match `user_id`, `email`, or `wallet_address` entries. Also checks `kyc_approved_context` entries against the user's approved KYC contexts.
2. **Holder-based matches:** For `collection_holder`, `key_collection_holder`, and `key_erc20_holder` entries, calls the W3Block KEY service to verify the user's wallets hold the required tokens, matching on contract address/collection ID, chain ID, minimum amount, and metadata requirements.

**Notes:**
- Results are cached for **120 seconds**. Pass `disableCache=true` to bypass
- A user can check their own access (the `AcceptSelfUserRequest` decorator allows self-requests on the `userId` query parameter)
- The top-level `hasAccess` is `true` if **any** whitelist grants access

---

## Flow: Promote Whitelist On-Chain

### Step 1: Promote to Wallet Group

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/whitelists/{tenantId}/{id}/promote-on-chain` | Bearer (Admin) |

**Request Body:**

```json
{
  "chainId": 137
}
```

**Notes:**
- The tenant must have an `operatorAddress` configured
- Creates a `WalletGroup` linked to the whitelist for the specified chain
- A whitelist can only be promoted once per chain ID
- After creation, the wallet group is queued for sync and blockchain publish
- Subsequent entry deletions from the whitelist will also update the on-chain wallet group

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | WhitelistEntrySaveException | Duplicate entry (same whitelist + type + value) | Check for existing entry before adding |
| 400 | WhitelistEntryUserNotFoundException | User ID not found or belongs to different tenant | Verify user exists in the same tenant |
| 400 | WhitelistAlreadyOnChainException | Whitelist already promoted for this chainId | Use the existing wallet group or choose a different chain |
| 400 | WhitelistOnChainException | Wallet group creation failed | Check tenant operator address configuration |
| 400 | WhitelistCheckException | Access check returned no details | Verify the whitelist and user IDs are valid |
| 400 | WhitelistEntryNotFoundException | Entry not found in whitelist/tenant | Verify entry ID and whitelist ownership |
| 404 | WhitelistNotFoundException | Whitelist not found for tenant | Verify whitelist ID and tenant |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Entry value case mismatch | All values are lowercased on save. Always compare in lowercase |
| 2 | Adding user_id entry for wrong tenant | The service validates that the user belongs to the same tenant. Use the correct tenantId |
| 3 | Missing additionalData for collection_holder | The `chainId` field in `additionalData` is required for `collection_holder` type entries |
| 4 | Cache returns stale access check | Access check results are cached 120 seconds. Pass `disableCache=true` for real-time results |
| 5 | On-chain promotion fails silently | Ensure the tenant has an `operatorAddress` configured before promoting |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Auth Sign-In | Required for Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| Commerce Promotions | Whitelists can be linked to promotions | [FLOW_COMMERCE_PROMOTIONS](../commerce/FLOW_COMMERCE_PROMOTIONS.md) |
| Contacts / KYC | `kyc_approved_context` entries depend on KYC approval | [FLOW_CONTACTS_KYC_APPROVAL](../contacts/FLOW_CONTACTS_KYC_APPROVAL.md) |
