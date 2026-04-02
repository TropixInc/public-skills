---
id: FLOW_COMMERCE_PRODUCT_LIFECYCLE
title: "Commerce - Product Lifecycle"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - products
  - variants
  - order-rules
  - tags
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
---

# Commerce — Product Lifecycle

## Overview

Products in W3Block represent sellable items — NFTs (ERC721/1155), fungible tokens (ERC20), or external goods. A product goes through a defined lifecycle: **Draft → Publishing → Published → (Sold | Cancelled)**. This flow covers creating products, adding variants and order rules, publishing, and managing tags.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token with Admin role | ID service authentication |
| Smart contract | Deployed contract (for NFT/ERC20 products) | Tokens module |
| Currencies | At least one currency configured | `GET /globals/currencies` |

## Entities & Relationships

```
Product (1) ──→ (N) ProductToken       ← individual sellable units
Product (1) ──→ (N) ProductVariant     ← variant dimensions (Size, Color)
  ProductVariant (1) ──→ (N) ProductVariantValue  ← options (S, M, L)
Product (1) ──→ (N) ProductOrderRule   ← purchase restrictions
Product (M) ←──→ (N) Tag              ← categorization
```

### Product Status Lifecycle

```
DRAFT ──→ PUBLISHING ──→ PUBLISHED ──→ SOLD (all tokens sold)
  ↑          │                │
  │          ↓                ↓
  │       (failure)       CANCELLED
  │          │
  └──────────┘ (retry)

PUBLISHED ──→ UPDATING ──→ PUBLISHED
```

---

## Flow: Create a Product

### Step 1: Upload Product Images

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/assets` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "type": "image",
  "target": "PRODUCT"
}
```

**Response (200):**
```json
{
  "id": "asset-uuid",
  "uploadParams": {
    "url": "https://api.cloudinary.com/v1_1/...",
    "fields": {
      "api_key": "...",
      "signature": "...",
      "timestamp": "..."
    }
  }
}
```

**Notes:**
- Upload the actual file to the Cloudinary URL returned in `uploadParams`
- Save the `assetId` and resulting `original` / `thumb` URLs for the product

---

### Step 2: Create the Product (Draft)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/products` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "My NFT Collection",
  "contractAddress": "0x1234...abcd",
  "chainId": 137,
  "prices": [
    { "currencyId": "currency-uuid", "amount": "1000000000000000000" }
  ],
  "distributionType": "random"
}
```

**Complete Request (production example):**
```json
{
  "name": "My NFT Collection",
  "description": "A limited edition digital collectible",
  "contractAddress": "0x1234...abcd",
  "chainId": 137,
  "prices": [
    { "currencyId": "currency-uuid-brl", "amount": "50000000000000000000" },
    { "currencyId": "currency-uuid-matic", "amount": "1000000000000000000" }
  ],
  "images": [
    { "assetId": "asset-uuid", "original": "https://...", "thumb": "https://..." }
  ],
  "distributionType": "random",
  "pricingType": "product",
  "type": "nft",
  "slug": "my-nft-collection",
  "startSaleAt": "2026-04-01T00:00:00.000Z",
  "endSaleAt": "2026-06-30T23:59:59.000Z",
  "canResale": true,
  "onDemandMintEnabled": false,
  "htmlContent": "<p>Rich description with HTML</p>",
  "tags": ["tag-uuid-1", "tag-uuid-2"],
  "draftData": {
    "keyCollectionId": "collection-uuid",
    "range": "1-100",
    "quantity": "100"
  }
}
```

**Response (201):**
```json
{
  "id": "product-uuid",
  "companyId": "company-uuid",
  "name": "My NFT Collection",
  "status": "draft",
  "prices": [...],
  "distributionType": "random",
  "createdAt": "2026-03-31T12:00:00.000Z",
  "updatedAt": "2026-03-31T12:00:00.000Z"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Product display name |
| `contractAddress` | string | Yes (NFT/ERC20) | Smart contract address |
| `chainId` | integer | Yes (NFT/ERC20) | Blockchain chain ID (137=Polygon, 1=Ethereum) |
| `prices` | ProductPrice[] | Yes | Multi-currency pricing. Amount in wei/smallest unit |
| `distributionType` | enum | Yes | `random`, `fixed`, or `sequential` |
| `description` | string | No | Product description |
| `images` | ProductImage[] | No | Product images (from asset upload) |
| `pricingType` | string | No | `product` (same price all tokens) or `token` (per-token pricing) |
| `type` | enum | No | `nft`, `erc20`, `external`. Default: `nft` |
| `slug` | string | No | URL slug, unique per company |
| `startSaleAt` | ISO 8601 | No | Sale start date |
| `endSaleAt` | ISO 8601 | No | Sale end date |
| `canResale` | boolean | No | Enable secondary sales |
| `onDemandMintEnabled` | boolean | No | Mint tokens on purchase instead of pre-minting |
| `htmlContent` | string | No | Rich HTML content for product page |
| `tags` | string[] | No | Tag UUIDs for categorization |
| `draftData` | object | No | Collection reference, token range, quantity |
| `disableSelfPurchase` | boolean | No | Prevent the product creator from buying |
| `requirements` | object | No | Purchase requirements (KYC, etc.) |

**Notes:**
- Product is created in `draft` status — it is not visible to buyers yet
- Amounts are **big numbers** (wei for crypto, smallest currency unit for fiat)
- `draftData.keyCollectionId` links to a token collection in the registry service
- Multiple prices allow selling in different currencies simultaneously

---

### Step 3: Add Variants (Optional)

Variants add dimensions like Size or Color. Each variant has values with optional price modifiers.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/products/{productId}/variants` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "name": "Size",
  "keyLabel": "size",
  "values": [
    { "name": "Small", "keyValue": "small", "extraAmount": "0" },
    { "name": "Medium", "keyValue": "medium", "extraAmount": "5000000000000000000" },
    { "name": "Large", "keyValue": "large", "extraAmount": "10000000000000000000" }
  ]
}
```

**Notes:**
- `extraAmount` is added to the base product price for that variant value
- Variants are soft-deleted (`deletedAt`) when removed
- Buyers select variant values during order creation via `variantIds`

---

### Step 4: Add Order Rules (Optional)

Order rules control who can purchase, when, and how many.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/products/{productId}/order-rules` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "purchaseLimit": 5
}
```

**Complete Request:**
```json
{
  "whitelistId": "whitelist-uuid",
  "startAt": "2026-04-01T00:00:00.000Z",
  "endAt": "2026-04-15T23:59:59.000Z",
  "purchaseLimit": 2
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `purchaseLimit` | number | No | Max purchases per user (default: 5) |
| `whitelistId` | string | No | Restrict to users in this whitelist |
| `startAt` | ISO 8601 | No | Rule effective start |
| `endAt` | ISO 8601 | No | Rule effective end |

**Notes:**
- Multiple rules can be added to a product; all must be satisfied
- Rules are evaluated during order creation
- If `whitelistId` is set, only users in that whitelist can purchase

---

### Step 5: Publish the Product

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/products/{productId}/publish` | Bearer (Admin) | application/json |

**Request:**
Same shape as the create DTO — pass any final updates along with the publish action.

```json
{
  "name": "My NFT Collection",
  "prices": [
    { "currencyId": "currency-uuid", "amount": "50000000000000000000" }
  ]
}
```

**Response (200):**
```json
{
  "id": "product-uuid",
  "status": "publishing",
  "..."
}
```

**Notes:**
- Status transitions to `publishing` immediately, then to `published` asynchronously
- During publishing, product tokens are created (minted or imported based on `draftData`)
- Poll `GET /admin/companies/{companyId}/products/{productId}` to check when status becomes `published`
- If publishing fails, status reverts to `draft` — check product details for error info

---

## Flow: Manage Tags

Tags provide hierarchical categorization. Create tags first, then assign them to products.

### Create a Tag

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/tags` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "Art"
}
```

**Complete Request:**
```json
{
  "name": "Digital Art",
  "parentId": "parent-tag-uuid",
  "hide": false
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Tag name |
| `parentId` | uuid | No | Parent tag for hierarchy |
| `hide` | boolean | No | Hide from public UI |

### Assign Tags to Product

Pass `tags: ["tag-uuid-1", "tag-uuid-2"]` in the product create or update request.

---

## Flow: Cancel a Product

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/products/{productId}/cancel` | Bearer (Admin) | — |

**Notes:**
- Only `published` products can be cancelled
- Cancellation prevents new orders but does not affect existing ones
- Associated product tokens are set to `cancelled` status

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Validation error | Missing required fields or invalid format | Check field types; amounts must be big number strings |
| 400 | Invalid contract address | Contract not found on chain | Verify contract is deployed and address is correct |
| 409 | Slug conflict | Slug already used by another product in this company | Choose a different slug |
| 409 | Product not in draft | Trying to publish a non-draft product | Check current status; only `draft` products can be published |
| 404 | Product not found | Invalid productId or wrong companyId | Verify IDs |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Product stays in `publishing` forever | Publishing is async. If stuck, the backend job may have failed. Check product details for error info and retry |
| 2 | Price amounts look wrong | Amounts are in **wei** (18 decimals for crypto) or smallest fiat unit. `1 MATIC = 1000000000000000000` |
| 3 | Tags not appearing on product | Tags are M:M — pass tag UUIDs in the `tags` array during create/update. They're not auto-inherited from parent tags |
| 4 | Variant extra amounts don't apply | Extra amounts add to base price. If base price is 0, only the extra amount is charged |
| 5 | Order rules not enforced | Rules are checked at order creation time, not at product view. Ensure rules have correct date ranges |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Authentication | Required before any admin operation | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Order & Purchase | Buyers purchase published products | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Promotions | Apply discounts/coupons to products | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Splits & Gateways | Configure payment and revenue for products | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
