---
id: FLOW_COMMERCE_SPLITS_GATEWAYS
title: "Commerce - Revenue Splits & Payment Gateways"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - splits
  - gateways
  - payments
  - stripe
  - asaas
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
---

# Commerce — Revenue Splits & Payment Gateways

## Overview

This flow covers two related configurations: **payment gateways** (connecting Stripe, ASAAS, or other providers) and **revenue splits** (distributing order revenue among stakeholders). Both must be configured before accepting orders.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT with Admin role | ID service authentication |
| Provider credentials | Stripe keys or ASAAS wallet ID | Provider dashboards |
| User UUIDs | Revenue recipients must be registered users | Contacts module |

## Entities & Relationships

```
CompanySplitConfiguration ──→ User (recipient)
                          ──→ CompanyConfiguration (parent)
                          ──→ Product (optional scope)

DeferredSplit ──→ Payment (source)
              ──→ User (recipient)
              ──→ Currency
```

### Split Types

Each order generates multiple fee components. Splits define who receives what percentage of each component:

```
Order Total
  ├── Product Price ────── split among sellers / partners
  ├── Client Service Fee ─ split among platform / partners
  ├── Company Service Fee  split among company / W3Block
  ├── Gas Fee ──────────── split for gas cost coverage
  └── Resale Fee ───────── split on secondary sales
```

---

## Flow: Configure Payment Gateways

### Step 1: Check Available Currencies

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/globals/currencies` | Public | — |

**Response (200):**
```json
{
  "items": [
    {
      "id": "brl-uuid",
      "name": "Brazilian Real",
      "code": "BRL",
      "symbol": "R$",
      "crypto": false,
      "erc20contractAddress": null
    },
    {
      "id": "matic-uuid",
      "name": "MATIC",
      "code": "MATIC",
      "symbol": "MATIC",
      "crypto": true,
      "erc20contractAddress": "0x..."
    }
  ],
  "meta": { "totalItems": 15 }
}
```

---

### Step 2: Configure Stripe

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/configurations/providers/stripe` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "secret": "sk_live_...",
  "publicKey": "pk_live_..."
}
```

**Complete Request:**
```json
{
  "secret": "sk_live_...",
  "publicKey": "pk_live_...",
  "minPaymentPrice": "500",
  "checkoutExpireTime": 1800
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `secret` | string | Yes | Stripe secret key |
| `publicKey` | string | Yes | Stripe publishable key |
| `minPaymentPrice` | string | No | Minimum payment amount (smallest unit) |
| `checkoutExpireTime` | number | No | Seconds until payment expires (default: 1800 = 30 min) |

---

### Step 3: Configure ASAAS

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/configurations/providers/asaas` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "apiKey": "aact_..."
}
```

**Complete Request:**
```json
{
  "apiKey": "aact_...",
  "checkoutExpireTime": 3600,
  "minPaymentPrice": "1000",
  "maxInstallments": 12,
  "minInstallmentPrice": "5000",
  "canSaveCreditCard": true,
  "percentageSplit": "5"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiKey` | string | Yes | ASAAS API key |
| `checkoutExpireTime` | number | No | Seconds until payment expires |
| `minPaymentPrice` | string | No | Minimum payment amount |
| `maxInstallments` | number | No | Max credit card installments |
| `minInstallmentPrice` | string | No | Min amount per installment |
| `canSaveCreditCard` | boolean | No | Allow users to save cards |
| `percentageSplit` | string | No | ASAAS split percentage |

---

### Step 4: Select Provider-Currency-Method Combinations

Define which payment methods are available for each currency.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/configurations/providers-selections` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "currencyId": "brl-uuid",
  "paymentProvider": "stripe",
  "paymentMethod": "credit_card"
}
```

**Notes:**
- Call this endpoint once per combination you want to enable
- Example setup for BRL:
  - Stripe + credit_card
  - Stripe + debit_card
  - ASAAS + pix
  - ASAAS + billet
- The order preview response will show `providersForSelection` based on these configurations
- Invalid combinations return a "Invalid payment provider" error

---

### Step 5: Get Current Configuration

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/configurations` | Bearer (Admin) | — |

Returns the full company commerce configuration including all configured providers and selections.

---

## Flow: Configure User Payment Provider

For platforms where individual users (sellers) need their own payment accounts.

### Check User's Configured Providers

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/companies/{companyId}/users/{userId}/providers/check-configured-providers` | Bearer | — |

### Configure Provider for User

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/companies/{companyId}/users/{userId}/providers/{provider}` | Bearer | application/json |

**Request (Stripe):**
```json
{
  "accountId": "acct_..."
}
```

**Request (ASAAS):**
```json
{
  "walletId": "..."
}
```

**Notes:**
- User provider accounts are needed for revenue splits — the system needs to know where to send each user's share
- Stripe uses Connect account IDs (`acct_...`)
- ASAAS uses wallet IDs

---

## Flow: Configure Revenue Splits

### Step 1: Create a Split Configuration

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/split-configurations` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "type": "product_price",
  "userId": "seller-user-uuid",
  "percentage": 80
}
```

**Complete Request (production example):**
```json
{
  "type": "product_price",
  "userId": "seller-user-uuid",
  "percentage": 80,
  "description": "Seller receives 80% of product price on Stripe credit card sales",
  "paymentProvider": "stripe",
  "paymentMethod": "credit_card",
  "productId": "specific-product-uuid",
  "contractAddress": "0x1234...abcd",
  "chainId": 137
}
```

**Response (201):**
```json
{
  "id": "split-config-uuid",
  "companyId": "company-uuid",
  "type": "product_price",
  "userId": "seller-user-uuid",
  "percentage": 80,
  "createdAt": "2026-03-31T12:00:00.000Z"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Split type: `product_price`, `client_service_fee`, `company_service_fee`, `gas_fee`, `resale_fee` |
| `userId` | uuid | Yes | Revenue recipient user |
| `percentage` | number | Yes | 0-100, percentage of the fee type |
| `description` | string | No | Admin description |
| `paymentProvider` | enum | No | Apply only to this provider |
| `paymentMethod` | enum | No | Apply only to this method |
| `productId` | uuid | No | Apply only to this product |
| `contractAddress` | string | No | Apply only to this contract |
| `chainId` | integer | No | Apply only to this blockchain |

**Notes:**
- Splits without filters (no `paymentProvider`, `productId`, etc.) apply to **all** orders
- Filters narrow the scope: a split with `paymentProvider: "stripe"` only applies to Stripe payments
- Multiple splits can exist for the same type — their percentages should sum to ≤ 100%
- If total split % < 100%, the remaining percentage goes to the company treasury by default

---

### Step 2: List Split Configurations

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/split-configurations` | Bearer (Admin) | — |

---

### Step 3: Update a Split

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/admin/companies/{companyId}/split-configurations/{configId}` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "percentage": 85,
  "description": "Updated seller share to 85%"
}
```

---

### Step 4: Delete a Split

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| DELETE | `/admin/companies/{companyId}/split-configurations/{configId}` | Bearer (Admin) | — |

---

## How Splits Work at Order Time

1. When an order is completed, the system calculates each fee component:
   - `currencyAmount` → product price
   - `clientServiceFee` → buyer fee
   - `companyServiceFee` → company fee
   - `gasFee` → blockchain gas
   - `resaleFee` → secondary sale fee

2. For each component, matching split configurations are found (filtered by provider, method, product, chain)

3. Each recipient gets their percentage as a `DeferredSplit`

4. Deferred splits execute asynchronously:
   ```
   PENDING → PENDING_USER_ACCOUNT → PROCESSING_SPLIT → UNDER_USER_ACCOUNT → PROCESSING_WITHDRAW → WITHDRAWN
   ```

5. If the recipient doesn't have a payment provider account, the split waits in `PENDING_USER_ACCOUNT`

---

## Example: Complete Gateway + Split Setup

```
Scenario: Marketplace with Stripe (credit card) and ASAAS (PIX) for BRL

1. Configure Stripe:
   PATCH /admin/.../configurations/providers/stripe
   { "secret": "sk_live_...", "publicKey": "pk_live_..." }

2. Configure ASAAS:
   PATCH /admin/.../configurations/providers/asaas
   { "apiKey": "aact_..." }

3. Enable payment methods:
   POST /admin/.../configurations/providers-selections
   { "currencyId": "brl-uuid", "paymentProvider": "stripe", "paymentMethod": "credit_card" }

   POST /admin/.../configurations/providers-selections
   { "currencyId": "brl-uuid", "paymentProvider": "asaas", "paymentMethod": "pix" }

4. Configure splits:
   POST /admin/.../split-configurations
   { "type": "product_price", "userId": "seller-uuid", "percentage": 80 }

   POST /admin/.../split-configurations
   { "type": "product_price", "userId": "platform-uuid", "percentage": 15 }

   // Remaining 5% stays in company treasury

   POST /admin/.../split-configurations
   { "type": "client_service_fee", "userId": "platform-uuid", "percentage": 100 }
```

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Invalid payment provider | Provider not recognized or not supported | Check valid providers: `stripe`, `asaas`, `pagar_me`, `paypal`, `transfer`, `crypto`, `free`, `braza` |
| 400 | Invalid credentials | Stripe keys or ASAAS API key invalid | Verify keys in provider dashboard |
| 400 | Percentage exceeds 100 | Total splits for a type exceed 100% | Reduce percentages; total per type must be ≤ 100% |
| 400 | User not found | Split recipient userId doesn't exist | Create user in contacts first |
| 404 | Configuration not found | Company has no commerce configuration | Ensure commerce is enabled: `GET /admin/companies/{companyId}/is-enabled` |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Payment method not appearing in order preview | You must create a provider-selection for each currency + provider + method combination |
| 2 | Splits not executing | Recipient user needs a configured payment provider account (Stripe Connect or ASAAS wallet) |
| 3 | Split stuck in `PENDING_USER_ACCOUNT` | The recipient hasn't set up their payment provider. Use the user provider configuration endpoints |
| 4 | Wrong provider gets the split | Splits without `paymentProvider` filter apply to all providers. Add a provider filter to restrict |
| 5 | Splits for a specific product not working | Pass `productId` in the split configuration to scope it. Without it, the split applies to all products |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Authentication | Admin token required | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Product Lifecycle | Products must exist for product-scoped splits | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Order & Purchase | Splits execute after order payment | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
