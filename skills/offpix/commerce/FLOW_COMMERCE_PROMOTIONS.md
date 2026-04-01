---
id: FLOW_COMMERCE_PROMOTIONS
title: "Commerce - Promotions & Coupons"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - promotions
  - coupons
  - discounts
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
---

# Commerce — Promotions & Coupons

## Overview

The promotion system supports two types: **coupons** (code-based, user enters a code) and **discounts** (automatic, applied without code). Both can be scoped to specific products, users, date ranges, and payment methods. Promotions track usage per user and globally, and can be combined or restricted.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT with Admin role | ID service authentication |
| Published products | Products to apply promotions to | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |

## Entities & Relationships

```
Promotion (1) ──→ (N) PromotionProduct     ← which products qualify
Promotion (1) ──→ (N) PromotionWhitelist   ← which users qualify
Promotion (1) ──→ (N) PromotionUsageLog    ← audit trail
Promotion (N) ──→ (1) User (owner)         ← creator (for referrals)
```

### Promotion Types

| Type | Trigger | Example |
|------|---------|---------|
| `coupon` | User enters a code at checkout | Code "SUMMER20" for 20% off |
| `discount` | Applied automatically if conditions met | 10% off for PIX payments |

---

## Flow: Create a Coupon

### Step 1: Create the Promotion

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions` | Bearer (Admin) | application/json |

**Minimal Request (simple coupon):**
```json
{
  "publicDescription": "20% off summer sale",
  "type": "coupon",
  "amountType": "percentage",
  "amount": "20",
  "code": "SUMMER20",
  "amountByCart": true,
  "applyToAllProducts": true,
  "applyToAllUsers": true,
  "isCombinable": false
}
```

**Complete Request (production example):**
```json
{
  "description": "Internal: Summer 2026 campaign - Instagram influencer codes",
  "publicDescription": "20% off your purchase!",
  "type": "coupon",
  "amountType": "percentage",
  "amount": "20",
  "amountByCart": true,
  "code": "SUMMER20",
  "applyToAllProducts": false,
  "applyToAllUsers": false,
  "isCombinable": false,
  "startAt": "2026-04-01T00:00:00.000Z",
  "endAt": "2026-06-30T23:59:59.000Z",
  "maxUsages": 1000,
  "maxUsagesPerUser": 3,
  "requirements": {
    "minCartPrice": "10000000000000000000",
    "maxCartItems": 5,
    "paymentMethods": ["credit_card", "pix"],
    "currencyIds": ["brl-currency-uuid"],
    "notApplicableProductIds": ["excluded-product-uuid"]
  }
}
```

**Response (201):**
```json
{
  "id": "promotion-uuid",
  "companyId": "company-uuid",
  "type": "coupon",
  "code": "SUMMER20",
  "amountType": "percentage",
  "amount": "20",
  "usages": 0,
  "maxUsages": 1000,
  "createdAt": "2026-03-31T12:00:00.000Z"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `publicDescription` | string | Yes | Customer-facing description |
| `type` | enum | Yes | `coupon` or `discount` |
| `amountType` | enum | Yes | `fixed` or `percentage` |
| `amount` | string | Yes | Discount value (big number for fixed, 0-100 for percentage) |
| `amountByCart` | boolean | Yes | `true`: apply once per order. `false`: apply per item |
| `applyToAllProducts` | boolean | Yes | Scope: all products or specific ones |
| `applyToAllUsers` | boolean | Yes | Scope: all users or whitelisted only |
| `isCombinable` | boolean | Yes | Can stack with other promotions |
| `code` | string | Yes (coupon) | UPPERCASE code, unique per company |
| `description` | string | No | Internal admin notes |
| `startAt` | ISO 8601 | No | Promotion start date |
| `endAt` | ISO 8601 | No | Promotion end date |
| `maxUsages` | number | No | Global usage limit |
| `maxUsagesPerUser` | number | No | Per-user usage limit |
| `requirements` | object | No | Additional conditions (see below) |

**Requirements Object:**

| Field | Type | Description |
|-------|------|-------------|
| `minCartPrice` | string | Minimum cart total (big number) |
| `maxCartPrice` | string | Maximum cart total |
| `minCartItems` | number | Minimum items in cart |
| `maxCartItems` | number | Maximum items in cart |
| `minSameCartItems` | number | Min quantity of the same product |
| `maxSameCartItems` | number | Max quantity of the same product |
| `paymentMethods` | string[] | Restrict to these payment methods |
| `currencyIds` | uuid[] | Restrict to these currencies |
| `destinationWalletAddresses` | string[] | Restrict to these destination wallets |
| `notApplicableProductIds` | uuid[] | Exclude these products |

**Notes:**
- Coupon codes are automatically uppercased
- Codes must be unique per company — duplicates return 409
- The `amount` field uses big numbers for `fixed` type but regular numbers (0-100) for `percentage`
- `amountByCart: true` means "20% off the total cart". `false` means "20% off each qualifying item"

---

### Step 2: Scope to Specific Products (if applyToAllProducts = false)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/products` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "productId": "product-uuid",
  "maxUsages": 500,
  "maxUsagesPerUser": 2
}
```

**Notes:**
- Each product in a promotion can have its own usage limits
- Use the SDK method `setAndOverridePromotionProducts` to replace all products at once

**To replace all products at once:**
```json
{
  "products": [
    { "productId": "product-1-uuid", "maxUsages": 100 },
    { "productId": "product-2-uuid", "maxUsages": 200 }
  ]
}
```

---

### Step 3: Restrict to Specific Users (if applyToAllUsers = false)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists` | Bearer (Admin) | application/json |

**By email:**
```json
{
  "type": "email",
  "value": "user@example.com",
  "maxUsages": 5,
  "maxUsagesPerUser": 1
}
```

**By W3Block whitelist:**
```json
{
  "type": "w3block_id_whitelist",
  "value": "whitelist-uuid",
  "maxUsages": 100
}
```

**Notes:**
- `email` type restricts to a specific email address
- `w3block_id_whitelist` type restricts to all users in a W3Block whitelist
- Each whitelist entry can have its own usage limits

---

## Flow: Create an Automatic Discount

Same endpoint as coupons, but with `type: "discount"` and no `code`.

**Request:**
```json
{
  "publicDescription": "10% off for PIX payments",
  "type": "discount",
  "amountType": "percentage",
  "amount": "10",
  "amountByCart": true,
  "applyToAllProducts": true,
  "applyToAllUsers": true,
  "isCombinable": true,
  "requirements": {
    "paymentMethods": ["pix"]
  }
}
```

**Notes:**
- Automatic discounts are evaluated during order preview without user input
- If `isCombinable: true`, this discount can stack with coupons
- The preview response shows the applied discount in `originalCartPrice` vs `cartPrice`

---

## Flow: Update a Promotion

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/promotions/{promotionId}` | Bearer (Admin) | application/json |

**Request (partial update):**
```json
{
  "maxUsages": 2000,
  "endAt": "2026-09-30T23:59:59.000Z"
}
```

---

## Flow: Delete a Promotion

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}` | Bearer (Admin) | — |

**Notes:**
- This is a **soft delete** (`deletedAt` is set)
- Existing orders with this promotion are not affected
- The coupon code becomes available for reuse after deletion

---

## Flow: Apply a Coupon to an Order

Coupons are applied during order creation or via a dedicated endpoint on an existing order.

### During Order Creation

Pass `couponCode` in the create order request:
```json
{
  "orderProducts": [...],
  "payments": [...],
  "couponCode": "SUMMER20"
}
```

### On an Existing Order

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/companies/{companyId}/orders/{orderId}/coupon` | Bearer (Owner) | application/json |

**Request:**
```json
{
  "code": "SUMMER20"
}
```

---

## How Promotions Are Evaluated

1. During order preview / creation, the system checks all active promotions
2. For `discount` type: automatically applied if all conditions match
3. For `coupon` type: only applied if the user provides the correct code
4. Validation checks in order:
   - Is the promotion active? (dates, soft-delete)
   - Has the global usage limit been reached?
   - Has the per-user limit been reached?
   - Does the user match the whitelist/email restrictions?
   - Does the cart match the requirements? (min/max price, items, payment method)
   - Does the product match? (applyToAllProducts or in PromotionProduct list)
5. If `isCombinable: false`, only one non-combinable promotion applies per order
6. Usage is logged in `PromotionUsageLog` on order creation
7. If the order is cancelled, usage is reverted (`reverted: true`)

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Invalid coupon code | Code not found or expired | Verify code, check date range |
| 400 | Usage limit exceeded | Global or per-user limit reached | Increase limits or create new promotion |
| 400 | Requirements not met | Cart doesn't meet requirements | Check minCartPrice, paymentMethods, etc. |
| 400 | Product not eligible | Product not in promotion scope | Add product to promotion or set applyToAllProducts |
| 400 | User not eligible | User not in whitelist | Add user email or whitelist entry |
| 409 | Code already exists | Duplicate coupon code for this company | Use a different code |
| 409 | Non-combinable conflict | Multiple non-combinable promotions | Set `isCombinable: true` or remove conflicting promotion |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Coupon code rejected as "not found" | Codes are **UPPERCASE**. Send `"SUMMER20"` not `"summer20"` |
| 2 | Discount not appearing in preview | Check: is the discount type `discount` (not `coupon`)? Are dates active? Does the cart meet requirements? |
| 3 | Promotion applied to wrong products | If `applyToAllProducts: false`, you must add products via the promotion-products endpoint |
| 4 | Usage count not decrementing on cancel | Cancellation sets `reverted: true` on usage logs but doesn't decrement the counter. The system checks `reverted` when evaluating limits |
| 5 | Percentage discount higher than expected | `amountByCart: false` applies the percentage to **each item**, which can exceed expectations on multi-item carts |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Authentication | Admin token required | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Product Lifecycle | Products must exist before scoping | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Order & Purchase | Promotions applied during order creation | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
