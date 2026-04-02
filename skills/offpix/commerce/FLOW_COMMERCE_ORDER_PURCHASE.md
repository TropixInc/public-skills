---
id: FLOW_COMMERCE_ORDER_PURCHASE
title: "Commerce - Order & Purchase Flow"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - orders
  - payments
  - refunds
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
  - FLOW_COMMERCE_PRODUCT_LIFECYCLE
---

# Commerce — Order & Purchase Flow

## Overview

This flow covers the complete purchase lifecycle: previewing an order (computing prices, fees, and discounts), creating the order, processing payment, and handling cancellations/refunds. W3Block supports multi-currency orders with multiple payment methods in a single transaction.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token | ID service authentication |
| Published product | Product in `published` status | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Wallet address | Destination wallet for NFT delivery | User's connected wallet |
| Configured gateway | At least one payment provider configured | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |

## Entities & Relationships

```
Order (1) ──→ (N) OrderProduct   ← items in the order
Order (1) ──→ (N) Payment        ← multi-payment support
Order (1) ──→ (N) OrderDocument  ← attached files (invoices, etc.)
Order (1) ──→ (N) Refund         ← refund requests
OrderProduct ──→ ProductToken    ← specific token being purchased
Payment ──→ Currency             ← payment currency
```

### Order Status Lifecycle

```
PENDING ──→ CONFIRMING_PAYMENT ──→ WAITING_DELIVERY ──→ DELIVERING ──→ CONCLUDED
                                                                          │
                                                                        FAILED
                                        │
                          CANCELLING ──→ CANCELLED / PARTIALLY_CANCELLED
                                        │
                                      EXPIRED
```

---

## Flow: Purchase a Product

### Step 1: Preview the Order

Always preview before creating. This computes prices, fees, available payment providers, and validates the cart.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/companies/{companyId}/orders/preview` | Public | application/json |

**Minimal Request:**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "quantity": 1,
      "expectedPrices": [
        { "currencyId": "currency-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "currency-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "all_remaining"
    }
  ]
}
```

**Complete Request:**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "selectBestPrice": true,
      "quantity": 1,
      "variantIds": ["variant-value-uuid"],
      "expectedPrices": [
        { "currencyId": "brl-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "percentage",
      "amount": "50"
    },
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "asaas",
      "paymentMethod": "pix",
      "amountType": "all_remaining"
    }
  ],
  "destinationWalletAddress": "0xabc...123",
  "couponCode": "SUMMER20",
  "signedGasFees": [
    { "currencyId": "matic-uuid", "signedGasFee": "1000000000000000" }
  ]
}
```

**Response (200):**
```json
{
  "products": [
    {
      "id": "product-uuid",
      "name": "My NFT",
      "prices": [...],
      "promotions": [...]
    }
  ],
  "productsErrors": [],
  "appliedCoupon": "SUMMER20",
  "payments": [
    {
      "currencyId": "brl-uuid",
      "cartPrice": "50000000000000000000",
      "clientServiceFee": "2500000000000000000",
      "gasFee": "0",
      "totalPrice": "52500000000000000000",
      "originalCartPrice": "60000000000000000000",
      "originalClientServiceFee": "3000000000000000000",
      "originalTotalPrice": "63000000000000000000",
      "currencyAllowanceState": "AVAILABLE",
      "providersForSelection": [
        { "paymentProvider": "stripe", "paymentMethod": "credit_card", "available": true },
        { "paymentProvider": "asaas", "paymentMethod": "pix", "available": true },
        { "paymentProvider": "crypto", "paymentMethod": "crypto", "available": false }
      ]
    }
  ],
  "cashback": {
    "currencyId": "loyalty-uuid",
    "amount": "50000000000000000000",
    "cashbackAmount": "500000000000000000"
  }
}
```

**Notes:**
- `productsErrors` lists issues like "out of stock", "whitelist required", "purchase limit reached"
- `currencyAllowanceState` values: `AVAILABLE`, `INSUFFICIENT_ALLOWANCE` — check this before creating order
- `providersForSelection` tells you which payment methods are available for each currency
- `originalCartPrice` vs `cartPrice` shows the discount amount
- The preview is **not persisted** — it's a computation only

---

### Step 2: Create the Order

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/companies/{companyId}/orders` | Bearer | application/json |

**Minimal Request:**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "quantity": 1,
      "expectedPrices": [
        { "currencyId": "currency-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "currency-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "all_remaining",
      "providerInputs": {
        "credit_card_id": "card-uuid"
      }
    }
  ],
  "destinationWalletAddress": "0xabc...123",
  "signedGasFees": [
    { "currencyId": "currency-uuid" }
  ]
}
```

**Complete Request (production example):**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "selectBestPrice": true,
      "quantity": 1,
      "variantIds": ["variant-value-uuid"],
      "expectedPrices": [
        { "currencyId": "brl-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "all_remaining",
      "providerInputs": {
        "credit_card_id": "card-uuid",
        "save_credit_card": true,
        "installments": 3
      }
    }
  ],
  "destinationWalletAddress": "0xabc...123",
  "destinationUserId": "user-uuid",
  "addressId": "address-uuid",
  "couponCode": "SUMMER20",
  "successUrl": "https://myapp.com/order/success",
  "utmParams": {
    "utm_source": "instagram",
    "utm_campaign": "summer_sale"
  },
  "signedGasFees": [
    { "currencyId": "matic-uuid", "signedGasFee": "1000000000000000" }
  ],
  "acceptSimilarOrderInShortPeriod": false,
  "passShareCodeData": {}
}
```

**Response (201):**
```json
{
  "id": "order-uuid",
  "companyId": "company-uuid",
  "userId": "user-uuid",
  "status": "pending",
  "deliverId": "ABC123",
  "destinationWalletAddress": "0xabc...123",
  "currencyAmount": [
    { "currencyId": "brl-uuid", "amount": "50000000000000000000" }
  ],
  "clientServiceFee": [
    { "currencyId": "brl-uuid", "amount": "2500000000000000000" }
  ],
  "products": [...],
  "payments": [...],
  "expiresIn": "2026-03-31T12:30:00.000Z",
  "createdAt": "2026-03-31T12:00:00.000Z"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `orderProducts` | array | Yes | 1-100 items to purchase |
| `orderProducts[].productTokenId` | uuid | Yes | Specific token to buy |
| `orderProducts[].quantity` | number | Yes | Quantity |
| `orderProducts[].expectedPrices` | array | Yes | Expected price per currency (prevents price changes) |
| `orderProducts[].selectBestPrice` | boolean | No | Auto-select cheapest price |
| `orderProducts[].variantIds` | string[] | No | Selected variant values |
| `orderProducts[].resellerId` | uuid | No | Reseller user (secondary sale) |
| `payments` | array | Yes | At least 1 payment method |
| `payments[].currencyId` | uuid | Yes | Payment currency |
| `payments[].paymentProvider` | enum | Yes | Provider to use |
| `payments[].paymentMethod` | enum | Yes | Method to use |
| `payments[].amountType` | string | Yes | `percentage`, `fixed`, or `all_remaining` |
| `payments[].amount` | string | Conditional | Required if amountType is `percentage` or `fixed` |
| `payments[].providerInputs` | object | No | Provider-specific data (card ID, installments, SSN) |
| `destinationWalletAddress` | string | Yes (NFT) | Ethereum address to receive tokens |
| `destinationUserId` | uuid | No | Send to another user's wallet |
| `addressId` | uuid | No | Shipping address |
| `couponCode` | string | No | Coupon code to apply |
| `successUrl` | string | No | Redirect URL after payment |
| `signedGasFees` | array | Yes | Gas fee acknowledgment per currency |
| `utmParams` | object | No | Campaign tracking |
| `acceptSimilarOrderInShortPeriod` | boolean | No | Allow duplicate orders (default: false) |

**Notes:**
- `expectedPrices` acts as a price lock — order fails if the price changed since preview
- `amountType: "all_remaining"` should be used on the last payment to cover the remainder
- The order starts in `pending` status and has an expiration time
- `deliverId` is generated automatically — used to track delivery externally
- `acceptSimilarOrderInShortPeriod: false` prevents accidental double purchases

---

### Step 3: Process Payment (if not auto-processed)

Some payment flows (e.g., PIX) require a second step after order creation.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/companies/{companyId}/orders/{orderId}/pay` | Bearer (Owner) | application/json |

**Request:**
```json
{
  "payments": [
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "asaas",
      "paymentMethod": "pix",
      "amountType": "all_remaining"
    }
  ],
  "successUrl": "https://myapp.com/order/success"
}
```

**Response (200):**
```json
{
  "id": "order-uuid",
  "status": "confirming_payment",
  "payments": [
    {
      "id": "payment-uuid",
      "status": "processing",
      "publicData": {
        "qrCode": "00020126...",
        "qrCodeUrl": "https://..."
      }
    }
  ]
}
```

**Notes:**
- For PIX: response includes `publicData.qrCode` and `qrCodeUrl` for the buyer
- For credit card: payment is usually auto-processed during order creation
- For manual transfer: admin must approve via `approve-payment` endpoint
- Payment status transitions: `pending → processing → concluded` (or `failed`)

---

### Step 4: Check Order Status

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/companies/{companyId}/orders/{orderId}` | Bearer (Owner) | — |

**Response (200):**
```json
{
  "id": "order-uuid",
  "status": "concluded",
  "deliverId": "ABC123",
  "deliverDate": "2026-03-31T12:15:00.000Z",
  "products": [
    {
      "orderId": "order-uuid",
      "productTokenId": "token-uuid",
      "status": "concluded",
      "deliveredAt": "2026-03-31T12:15:00.000Z",
      "currencyAmount": [...],
      "productToken": {
        "id": "token-uuid",
        "metadata": { "name": "NFT #1", "image": "https://..." }
      }
    }
  ],
  "payments": [
    {
      "id": "payment-uuid",
      "status": "concluded",
      "amount": "50000000000000000000",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card"
    }
  ]
}
```

---

## Flow: Admin Order Management

### Approve a Pending Payment

For manual transfer orders that require admin approval.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/approve-payment` | Bearer (Admin) | — |

**Notes:**
- Transitions order from `confirming_payment` to `waiting_delivery`
- Only applies to orders with `transfer` payment provider

### Cancel an Order (Admin)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/cancel` | Bearer (Admin) | — |

**Notes:**
- Triggers cancellation of all pending payments
- Product tokens are released back to `for_sale` status
- If some items already delivered: status becomes `partially_cancelled`
- Promotion usage logs are reverted (`reverted: true`)

### Create a Refund

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/orders/{orderId}/refund` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "currencyId": "currency-uuid",
  "amount": "50000000000000000000"
}
```

**Response (201):**
```json
{
  "id": "refund-uuid",
  "status": "pending",
  "amount": "50000000000000000000",
  "orderId": "order-uuid"
}
```

**Refund lifecycle:**
```
PENDING ──→ WAITING_APPROVAL ──→ PROCESSING ──→ CONCLUDED
                                             ↓
                                          FAILED
```

Approve with: `POST /admin/companies/{companyId}/refunds/{refundId}/approve`

---

## Flow: List & Filter Orders (Admin)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/orders` | Bearer (Admin) | — |

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `status` | string[] | Filter by order status(es) |
| `startDate` | ISO 8601 | Orders created after this date |
| `endDate` | ISO 8601 | Orders created before this date |
| `paymentMethod` | string | Filter by payment method |
| `userId` | uuid | Filter by buyer |
| `productId` | uuid | Filter by product |
| `utmCampaign` | string | Filter by UTM campaign |
| `page` | number | Page number |
| `limit` | number | Items per page |
| `sortBy` | string | Sort field |
| `orderBy` | string | `ASC` or `DESC` |

**Response includes `meta.ordersSummary`:**
```json
{
  "items": [...],
  "meta": {
    "totalItems": 500,
    "currentPage": 1,
    "ordersSummary": [
      {
        "currencyId": "brl-uuid",
        "total": "1500000000000000000000",
        "totalClientServiceFee": "75000000000000000000",
        "totalCompanyServiceFee": "30000000000000000000",
        "totalCurrencyAmount": "1395000000000000000000",
        "totalGasFee": "0",
        "totalOrders": 120
      }
    ]
  }
}
```

---

## Flow: Export Sales Report

### Step 1: Request Export

| Method | Path | Auth |
|--------|------|------|
| POST | `/admin/companies/{companyId}/exports/generate/orders` | Admin |

### Step 2: Poll for Status

| Method | Path | Auth |
|--------|------|------|
| GET | `/admin/companies/{companyId}/exports/{exportId}` | Admin |

Poll every 2 seconds until status indicates completion. Response includes a download URL.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Product unavailable | Token not in `for_sale` status | Check product/token availability |
| 400 | Purchase limit exceeded | User exceeded order rule limit | Check `purchaseLimit` in product order rules |
| 400 | Whitelist restriction | User not in required whitelist | Add user to whitelist or remove restriction |
| 400 | Price mismatch | `expectedPrice` doesn't match current price | Re-run preview and use updated prices |
| 400 | Invalid wallet | Destination wallet address invalid | Verify ethereum address format |
| 400 | Insufficient allowance | Payment currency not available | Check `currencyAllowanceState` in preview |
| 409 | Similar order exists | Duplicate order in short period | Set `acceptSimilarOrderInShortPeriod: true` or wait |
| 422 | Account verification required | Email/phone not verified | Complete account verification first |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Order created but no payment processing | Credit card: pass `providerInputs.credit_card_id`. PIX: call `/pay` endpoint after creation |
| 2 | "Price mismatch" error on order creation | Always run preview first, then use the exact prices from the preview response in `expectedPrices` |
| 3 | Order expires before payment | Check `expiresIn` field. Configure `checkoutExpireTime` in gateway settings for longer windows |
| 4 | Refund stuck in `pending` | Refunds require admin approval. Call the approve endpoint |
| 5 | Multi-payment total doesn't match | Use `amountType: "all_remaining"` on the last payment to automatically cover the remainder |
| 6 | Delivery tracking not found | `deliverId` is auto-generated uppercase. Use `GET /companies/{companyId}/orders/get-by-deliver-id/{deliverId}` |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Authentication | Bearer token required | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Product Lifecycle | Products must be published first | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Promotions | Coupons applied during order | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Splits & Gateways | Payment config must exist | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
