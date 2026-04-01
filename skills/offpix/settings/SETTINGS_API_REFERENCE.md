---
id: SETTINGS_API_REFERENCE
title: "Settings & Billing API Reference"
module: offpix/settings
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - settings
  - billing
  - tenant
  - api-reference
depends_on:
  - SETTINGS_SKILL_INDEX
---

# Settings & Billing API Reference

Complete API reference for the W3Block Settings & Billing module. Covers 16 endpoints across billing management (12) and tenant settings (4), plus all DTOs, enums, and entity relationships.

## Base URLs

| Environment | Service | URL |
|-------------|---------|-----|
| Production | Pixway ID | https://pixwayid.w3block.io |
| Swagger | Pixway ID | https://pixwayid.w3block.io/docs/ |

## Authentication

All authenticated endpoints require:
```
Authorization: Bearer {accessToken}
```

Role requirements are noted per endpoint. Roles: **SuperAdmin**, **Admin**, **Operator**, **User**, **Integration** (API key).

---

## Entities & Relationships

```
BillingPlanEntity (1) ----> (N) TenantPlanEntity ----> (1) TenantEntity
       |                            |                         |
       |                            |                    (1)--+--(1)
       |                            |                         |
       |                      (N) TenantBillingLogEntity   TenantConfigurationsEntity
       |                            |
       |                      (N) TenantPlanLogEntity
       |
       +-- BillingFeature[] (enabledFeatures)
       +-- BillingPlanLimit (per resource)

BillingUsageEntity (N) ----> (1) TenantEntity
BillingCurrencyExchangeRateEntity (standalone)
TenantClientEntity (N) ----> (1) TenantEntity
```

### BillingPlanEntity

The master plan definition. Contains pricing, limits, features, and display configuration.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `name` | string | Plan display name |
| `basePrice` | number | Monthly base price in USD |
| `isDefault` | boolean | Whether this is the default plan for new tenants |
| `isPublic` | boolean | Whether the plan is visible in public listings |
| `limits.nftContracts` | BillingPlanLimit | NFT contract creation limits |
| `limits.nftCollections` | BillingPlanLimit | NFT collection limits |
| `limits.nftTokens` | BillingPlanLimit | NFT token limits |
| `limits.tokensPerCollection` | BillingPlanLimit | Tokens per collection limits |
| `limits.nftMints` | BillingPlanLimit | NFT minting limits |
| `limits.erc20Contracts` | BillingPlanLimit | ERC20 contract limits |
| `limits.erc20Mints` | BillingPlanLimit | ERC20 minting limits |
| `pricing.extraNftMintPrice` | number | Cost per additional NFT mint beyond plan limit |
| `pricing.nftTransactionPrice` | number | Cost per NFT transaction |
| `pricing.erc20TransactionPrice` | number | Cost per ERC20 transaction |
| `pricing.nftSaleTransactionPrice` | number | Cost per NFT sale transaction |
| `pricing.erc20SaleTransactionPrice` | number | Cost per ERC20 sale transaction |
| `fees.primarySaleFee.minValue` | number | Minimum fee on primary sales |
| `fees.primarySaleFee.percentage` | number | Percentage fee on primary sales |
| `fees.resaleFee` | number | Fee on resales |
| `resourceLimits.activeWallets` | number | Max active wallets |
| `resourceLimits.onSaleProducts` | number | Max products on sale |
| `resourceLimits.storage` | number | Storage limit (bytes) |
| `resourceLimits.bandwidth` | number | Bandwidth limit (bytes) |
| `resourceLimits.maxUploadFileSizes.image` | number | Max image upload size (bytes) |
| `resourceLimits.maxUploadFileSizes.video` | number | Max video upload size (bytes) |
| `resourceLimits.maxUploadFileSizes.others` | number | Max other file upload size (bytes) |
| `userLimits.users` | number | Max users |
| `userLimits.maxClients` | number | Max OAuth clients |
| `userLimits.maxLoyaltyPartners` | number | Max loyalty partners |
| `features.enabledFeatures` | BillingFeature[] | List of enabled features |
| `display.homeSortOrder` | number | Sort order on plans page |
| `display.homeContent` | object | Content displayed on plans page |
| `trial.testDays` | number | Trial period in days |

### BillingPlanLimit

Limit structure used for each metered resource.

| Field | Type | Description |
|-------|------|-------------|
| `daily` | number or null | Daily soft limit |
| `weekly` | number or null | Weekly soft limit |
| `monthly` | number or null | Monthly soft limit |
| `lifetime` | number or null | Lifetime soft limit |
| `hardPerDay` | number or null | Daily hard (enforced) limit |
| `hardPerWeek` | number or null | Weekly hard (enforced) limit |
| `hardPerMonth` | number or null | Monthly hard (enforced) limit |
| `hardLifetime` | number or null | Lifetime hard (enforced) limit |

> **Soft vs. Hard limits:** Soft limits are tracked for billing/reporting. Hard limits are enforced and reject operations when exceeded.

### TenantPlanEntity

Links a tenant to their billing plan with optional custom overrides.

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | UUID | Tenant identifier (unique) |
| `planId` | UUID | Reference to BillingPlanEntity |
| `customLimits` | Partial&lt;BillingPlanLimits&gt; | Optional overrides of plan limits |
| `startCycleDate` | Date | Current billing cycle start |
| `endCycleDate` | Date | Current billing cycle end |
| `primaryCreditCardId` | string | Default credit card for charges |
| `coupon` | string | Applied coupon code |

### BillingUsageEntity

Tracks metered usage per tenant.

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | UUID | Tenant identifier |
| `type` | string | Usage type (see BillingUsageKnownTypes) |
| `isLifeTime` | boolean | Whether this is a lifetime counter |
| `value` | number | Current usage count |
| `metadata` | object | Additional usage metadata |

> **Unique index:** `(tenantId, type, isLifeTime)` — one record per tenant per usage type per lifetime flag.

### TenantBillingLogEntity

Immutable record of each billing cycle charge.

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | UUID | Tenant identifier |
| `startCycleDate` | Date | Cycle start date |
| `endCycleDate` | Date | Cycle end date |
| `paymentDate` | Date | When payment was processed |
| `creditCardId` | string | Card used for payment |
| `cyclePrice` | number | Calculated cycle price |
| `commerceOrderId` | string | Associated commerce order |
| `paidValue` | number | Amount actually charged |
| `paymentTries` | number | Number of charge attempts |
| `planId` | UUID | Plan at time of charge |
| `customLimits` | object | Custom limits at time of charge |
| `finalReport` | BillingEventSummary[] | Detailed breakdown of usage events |

### TenantPlanLogEntity

Audit trail for plan changes.

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | UUID | Tenant identifier |
| `oldPlanId` | UUID | Previous plan |
| `newPlanId` | UUID | New plan |
| `userId` | UUID | User who made the change |

### BillingCurrencyExchangeRateEntity

Currency conversion rates for multi-currency billing.

| Field | Type | Description |
|-------|------|-------------|
| `currencyId` | string | Currency identifier |
| `currencyCode` | string | ISO currency code |
| `usdExchangeRate` | number | Fixed USD exchange rate |
| `dynamicRate` | number | Dynamic (live) exchange rate |
| `dynamicCurrencyCode` | string | Currency code for dynamic rate |

### TenantEntity

Core tenant record.

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `name` | string | Tenant display name |
| `document` | string | Business document (CNPJ, EIN, etc.) |
| `countryCode` | string | ISO country code |
| `wallets` | object[] | Associated blockchain wallets |
| `clientId` | string | OAuth client identifier |
| `info` | object | Additional tenant info |
| `roles` | string[] | Tenant-level roles |
| `operatorAddress` | string | Blockchain operator address |
| `setupDone` | boolean | Whether initial setup is complete |

### TenantConfigurationsEntity

Tenant-specific feature and integration settings.

| Field | Type | Description |
|-------|------|-------------|
| `tenantId` | UUID | Tenant identifier (unique) |
| `kyc` | JSONB | KYC provider configuration |
| `signUp` | JSONB | Sign-up flow configuration |
| `passwordless` | object | `{ enabled: boolean }` |
| `googleOAuth` | object | Google OAuth provider config |
| `appleOAuth` | object | Apple OAuth provider config |
| `oneSignal` | object | OneSignal push notification config |
| `twilioWhatsApp` | object | Twilio WhatsApp messaging config |
| `clearSale` | object | ClearSale fraud detection config |
| `emailService` | object | Email service provider config |
| `automaticAccountExclusion` | boolean | Auto-delete inactive accounts |

---

## Enums

### BillingFeature (23 values)

Features that can be enabled or disabled per plan.

| Value | Description |
|-------|-------------|
| `MINT_ERC_721` | Mint ERC-721 (NFT) tokens |
| `MINT_ERC_20` | Mint ERC-20 (fungible) tokens |
| `MINT_WITH_ROYALTY` | Mint with royalty configuration |
| `MINT_WITH_VARIABLE_ROYALTIES` | Mint with variable royalty percentages |
| `MINT_PERMISSIONED_ERC_20` | Mint permissioned ERC-20 tokens |
| `MINT_CUSTODIAL_ERC_20` | Mint custodial ERC-20 tokens |
| `BATCH_MINT` | Batch minting operations |
| `CUSTOM_NFT_TEMPLATES` | Custom NFT display templates |
| `TRANSFER_NFT` | Transfer NFT tokens between wallets |
| `WHITE_LABEL` | White-label branding |
| `EXTERNAL_ECOMMERCE_INTEGRATION` | External e-commerce platform integration |
| `SMART_CONTRACT_PORTABILITY` | Export/import smart contracts |
| `CUSTODIAL_WALLET` | Custodial wallet management |
| `CUSTODIAL_WALLET_PORTABILITY` | Export custodial wallet keys |
| `BASIC_TOKEN_PASS` | Basic token pass (gate access) |
| `FULL_TOKEN_PASS` | Full token pass (advanced benefits) |
| `LOYALTY` | Loyalty program features |
| `KYC` | Know Your Customer verification |
| `COMMERCE` | Commerce/storefront features |
| `CONSTRUCTOR` | Visual constructor/builder |
| `CUSTOM_DOMAIN` | Custom domain configuration |

### BillingUsageKnownTypes (11 values)

Types of metered usage events.

| Value | Description |
|-------|-------------|
| `COMMERCE_PRODUCT_PURCHASE` | Product purchase in commerce |
| `COMMERCE_PRODUCT_PUBLISHED` | Product published to storefront |
| `KEY_NFT_MINTED` | NFT token minted |
| `KEY_ERC20_MINTED` | ERC20 token minted |
| `KEY_NFT_COLLECTION_CREATED` | NFT collection created |
| `KEY_NFT_CONTRACT_CREATED` | NFT contract deployed |
| `KEY_ERC20_CONTRACT_CREATED` | ERC20 contract deployed |
| `NFT_SALE_TRANSACTION` | NFT sale completed |
| `ERC20_SALE_TRANSACTION` | ERC20 sale completed |
| `NFT_TRANSACTION` | Generic NFT transaction (transfer, etc.) |
| `ERC20_TRANSACTION` | Generic ERC20 transaction (transfer, etc.) |

---

## Endpoints

### Billing Endpoints (12)

#### 1. List Billing Plans

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/plans` | None (Public) | Any |

Returns a paginated list of available billing plans.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 10) |
| `isPublic` | boolean | No | Filter by public visibility |

**Response (200):**
```json
{
  "items": [
    {
      "id": "plan-uuid",
      "name": "Professional",
      "basePrice": 99.00,
      "isDefault": false,
      "isPublic": true,
      "limits": {
        "nftContracts": {
          "daily": null,
          "weekly": null,
          "monthly": 10,
          "lifetime": null,
          "hardPerMonth": 15
        },
        "nftCollections": { "monthly": 50 },
        "nftTokens": { "monthly": 1000 },
        "nftMints": { "monthly": 500 },
        "erc20Contracts": { "monthly": 5 },
        "erc20Mints": { "monthly": 1000 }
      },
      "pricing": {
        "extraNftMintPrice": 0.50,
        "nftTransactionPrice": 0.10,
        "erc20TransactionPrice": 0.05,
        "nftSaleTransactionPrice": 0.25,
        "erc20SaleTransactionPrice": 0.15
      },
      "fees": {
        "primarySaleFee": {
          "minValue": 1.00,
          "percentage": 2.5
        },
        "resaleFee": 5.0
      },
      "resourceLimits": {
        "activeWallets": 100,
        "onSaleProducts": 50,
        "storage": 5368709120,
        "bandwidth": 10737418240,
        "maxUploadFileSizes": {
          "image": 10485760,
          "video": 104857600,
          "others": 52428800
        }
      },
      "userLimits": {
        "users": 1000,
        "maxClients": 5,
        "maxLoyaltyPartners": 10
      },
      "features": {
        "enabledFeatures": [
          "MINT_ERC_721",
          "MINT_ERC_20",
          "COMMERCE",
          "LOYALTY",
          "BASIC_TOKEN_PASS"
        ]
      },
      "display": {
        "homeSortOrder": 2,
        "homeContent": {}
      },
      "trial": {
        "testDays": 14
      }
    }
  ],
  "meta": {
    "totalItems": 3,
    "itemCount": 3,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

---

#### 2. Register Billing Usage

| Method | Path | Auth | Role |
|--------|------|------|------|
| POST | `/billing/{tenantId}/register-billing-usage` | Bearer token | SuperAdmin, Integration |

Registers a metered usage event against the tenant's billing.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Minimal Request:**
```json
{
  "type": "KEY_NFT_MINTED"       // required
}
```

**Complete Request:**
```json
{
  "type": "KEY_NFT_MINTED",      // required
  "value": 5,                     // optional -- number of units (default: 1)
  "metadata": {                   // optional -- additional context
    "contractId": "contract-uuid",
    "collectionId": "collection-uuid"
  }
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | BillingUsageKnownTypes | Yes | Usage event type |
| `value` | number | No | Number of units (default: 1) |
| `metadata` | object | No | Contextual metadata |

**Response (201):** Empty body on success.

---

#### 3. Check Feature Enabled

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/{tenantId}/is-feature-enabled/{feature}` | Bearer token | Admin, User, Operator |

Checks whether a specific billing feature is enabled for the tenant's current plan.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |
| `feature` | BillingFeature | Yes | Feature to check |

**Response (200):**
```json
{
  "enabled": true
}
```

---

#### 4. Get Billing State

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/{tenantId}/state` | Bearer token | Admin |

Returns the complete billing state for a tenant, including current plan, limits, usage, and cycle info.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Response (200):**
```json
{
  "plan": {
    "id": "plan-uuid",
    "name": "Professional",
    "basePrice": 99.00
  },
  "tenantPlan": {
    "tenantId": "tenant-uuid",
    "planId": "plan-uuid",
    "customLimits": null,
    "startCycleDate": "2026-03-01T00:00:00.000Z",
    "endCycleDate": "2026-03-31T23:59:59.000Z",
    "primaryCreditCardId": "card_abc123",
    "coupon": null
  },
  "features": {
    "enabledFeatures": ["MINT_ERC_721", "COMMERCE", "LOYALTY"]
  }
}
```

---

#### 5. Set Credit Card

| Method | Path | Auth | Role |
|--------|------|------|------|
| PATCH | `/billing/{tenantId}/credit-card` | Bearer token | Admin |

Sets or updates the primary credit card for billing.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Minimal Request:**
```json
{
  "creditCardId": "card_abc123"   // required
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `creditCardId` | string | Yes | Credit card identifier from payment provider |

**Response (200):** Updated TenantPlanEntity.

---

#### 6. Change Plan

| Method | Path | Auth | Role |
|--------|------|------|------|
| PATCH | `/billing/{tenantId}/plan` | Bearer token | Admin |

Changes the tenant's billing plan. Resets the billing cycle dates.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Minimal Request:**
```json
{
  "planId": "new-plan-uuid"       // required
}
```

**Complete Request:**
```json
{
  "planId": "new-plan-uuid",      // required
  "coupon": "PROMO2026"           // optional -- coupon code for discount
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `planId` | UUID | Yes | Target plan identifier |
| `coupon` | string | No | Discount coupon code |

**Response (200):** Updated TenantPlanEntity.

**Notes:**
- Creates a TenantPlanLogEntity audit record (oldPlanId, newPlanId, userId)
- Resets `startCycleDate` and `endCycleDate` to the new billing cycle
- A valid credit card must be set for paid plans

---

#### 7. Cancel Subscription

| Method | Path | Auth | Role |
|--------|------|------|------|
| PATCH | `/billing/{tenantId}/cancel` | Bearer token | Admin |

Cancels the tenant's current subscription, reverting to the default plan.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Response (200):** Updated TenantPlanEntity (now on default plan).

**Notes:**
- The tenant reverts to the default (free) plan
- Creates an audit log entry
- Active features from the cancelled plan are disabled immediately

---

#### 8. Get Billing Cycles

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/{tenantId}/cycles` | Bearer token | Admin |

Returns the history of billing cycles with charge details.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number |
| `limit` | number | No | Items per page |

**Response (200):**
```json
{
  "items": [
    {
      "tenantId": "tenant-uuid",
      "startCycleDate": "2026-02-01T00:00:00.000Z",
      "endCycleDate": "2026-02-28T23:59:59.000Z",
      "paymentDate": "2026-03-01T00:05:00.000Z",
      "creditCardId": "card_abc123",
      "cyclePrice": 124.50,
      "paidValue": 124.50,
      "paymentTries": 1,
      "planId": "plan-uuid",
      "customLimits": null,
      "finalReport": [
        {
          "type": "KEY_NFT_MINTED",
          "count": 520,
          "includedInPlan": 500,
          "overage": 20,
          "overageCost": 10.00
        }
      ]
    }
  ],
  "meta": {
    "totalItems": 6,
    "itemCount": 6,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

---

#### 9. Get Billing Summary

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/{tenantId}/billing-summary` | Bearer token | Admin |

Returns a summary of usage and costs for the current billing cycle.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Response (200):**
```json
{
  "currentCycle": {
    "startDate": "2026-03-01T00:00:00.000Z",
    "endDate": "2026-03-31T23:59:59.000Z"
  },
  "basePlanCost": 99.00,
  "usageCosts": [
    {
      "type": "KEY_NFT_MINTED",
      "count": 150,
      "includedInPlan": 500,
      "overage": 0,
      "overageCost": 0.00
    },
    {
      "type": "NFT_TRANSACTION",
      "count": 45,
      "cost": 4.50
    }
  ],
  "totalEstimatedCost": 103.50
}
```

---

#### 10. Simulate Billing Event Usage

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/{tenantId}/simulate-billing-event-usage` | Bearer token | Admin |

Simulates the cost of a billing event without actually registering it. Useful for showing users projected costs before they perform an action.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | BillingUsageKnownTypes | Yes | Usage event type to simulate |
| `value` | number | No | Number of units to simulate (default: 1) |

**Response (200):**
```json
{
  "type": "KEY_NFT_MINTED",
  "currentUsage": 480,
  "planLimit": 500,
  "simulatedAdditional": 25,
  "wouldExceedBy": 5,
  "estimatedOverageCost": 2.50
}
```

---

#### 11. List Billing Usages

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/billing/{tenantId}/billing-usages` | Bearer token | Admin |

Returns all usage records for the tenant.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number |
| `limit` | number | No | Items per page |

**Response (200):**
```json
{
  "items": [
    {
      "tenantId": "tenant-uuid",
      "type": "KEY_NFT_MINTED",
      "isLifeTime": false,
      "value": 150,
      "metadata": {}
    },
    {
      "tenantId": "tenant-uuid",
      "type": "KEY_NFT_MINTED",
      "isLifeTime": true,
      "value": 3200,
      "metadata": {}
    }
  ],
  "meta": {
    "totalItems": 22,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 3,
    "currentPage": 1
  }
}
```

---

#### 12. Invoice and Charge

| Method | Path | Auth | Role |
|--------|------|------|------|
| PATCH | `/billing/{tenantId}/invoice-and-charge` | Bearer token | SuperAdmin |

Manually triggers invoicing and charging for the tenant's current billing cycle. Typically used by platform administrators for manual billing corrections.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Response (200):** TenantBillingLogEntity for the generated invoice.

**Notes:**
- Creates a TenantBillingLogEntity with `finalReport`
- Charges the tenant's primary credit card
- Increments `paymentTries` on failure
- SuperAdmin only -- not available to tenant admins

---

### Tenant Settings Endpoints (4)

#### 13. Create/Update Tenant Configuration

| Method | Path | Auth | Role |
|--------|------|------|------|
| POST | `/tenant/configurations/{tenantId}` | Bearer token | Admin |

Creates or updates the tenant's configuration. Uses upsert behavior (creates if not exists, updates if exists).

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Minimal Request:**
```json
{
  "passwordless": {               // required -- at least one config field
    "enabled": true
  }
}
```

**Complete Request:**
```json
{
  "kyc": {                                    // optional -- KYC provider settings
    "provider": "clear_sale",
    "enabled": true,
    "requiredForPurchase": false
  },
  "signUp": {                                // optional -- sign-up flow config
    "requireEmailVerification": true,
    "allowedDomains": ["example.com"]
  },
  "passwordless": {                          // optional -- passwordless auth
    "enabled": true
  },
  "googleOAuth": {                           // optional -- Google OAuth
    "clientId": "google-client-id",
    "clientSecret": "google-client-secret",
    "enabled": true
  },
  "appleOAuth": {                            // optional -- Apple OAuth
    "clientId": "apple-client-id",
    "teamId": "apple-team-id",
    "keyId": "apple-key-id",
    "enabled": true
  },
  "oneSignal": {                             // optional -- push notifications
    "appId": "onesignal-app-id",
    "apiKey": "onesignal-api-key"
  },
  "twilioWhatsApp": {                        // optional -- WhatsApp messaging
    "accountSid": "twilio-sid",
    "authToken": "twilio-token",
    "fromNumber": "+15551234567"
  },
  "clearSale": {                             // optional -- fraud detection
    "enabled": true,
    "apiKey": "clearsale-api-key"
  },
  "emailService": {                          // optional -- email provider
    "provider": "sendgrid",
    "apiKey": "sendgrid-api-key",
    "fromEmail": "noreply@example.com"
  },
  "automaticAccountExclusion": false         // optional -- auto-delete accounts
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kyc` | JSONB | No | KYC verification provider settings |
| `signUp` | JSONB | No | Sign-up flow configuration |
| `passwordless` | object | No | `{ enabled: boolean }` -- toggle passwordless auth |
| `googleOAuth` | object | No | Google OAuth credentials and toggle |
| `appleOAuth` | object | No | Apple OAuth credentials and toggle |
| `oneSignal` | object | No | OneSignal push notification config |
| `twilioWhatsApp` | object | No | Twilio WhatsApp config |
| `clearSale` | object | No | ClearSale fraud detection config |
| `emailService` | object | No | Email service provider config |
| `automaticAccountExclusion` | boolean | No | Auto-delete inactive accounts |

**Response (201):** Created/updated TenantConfigurationsEntity.

---

#### 14. Get Tenant Configuration

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/tenant/configurations/{tenantId}` | Bearer token | Admin |

Retrieves the tenant's current configuration settings.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Response (200):**
```json
{
  "tenantId": "tenant-uuid",
  "kyc": {
    "provider": "clear_sale",
    "enabled": true,
    "requiredForPurchase": false
  },
  "signUp": {
    "requireEmailVerification": true
  },
  "passwordless": {
    "enabled": true
  },
  "googleOAuth": {
    "clientId": "google-client-id",
    "enabled": true
  },
  "appleOAuth": null,
  "oneSignal": null,
  "twilioWhatsApp": null,
  "clearSale": null,
  "emailService": {
    "provider": "sendgrid",
    "fromEmail": "noreply@example.com"
  },
  "automaticAccountExclusion": false
}
```

---

#### 15. Update Tenant Profile

| Method | Path | Auth | Role |
|--------|------|------|------|
| PUT | `/tenant/profile/{tenantId}` | Bearer token | Admin |

Updates the tenant's profile information (name, document, country).

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Minimal Request:**
```json
{
  "name": "My Company"            // required
}
```

**Complete Request:**
```json
{
  "name": "My Company LLC",       // required
  "document": "12345678000190",   // optional -- business registration number
  "countryCode": "BR",           // optional -- ISO 3166-1 alpha-2
  "info": {                       // optional -- additional metadata
    "website": "https://example.com",
    "industry": "retail"
  }
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Tenant display name |
| `document` | string | No | Business document number (CNPJ, EIN, etc.) |
| `countryCode` | string | No | ISO 3166-1 alpha-2 country code |
| `info` | object | No | Additional tenant metadata |

**Response (200):** Updated TenantEntity.

---

#### 16. Get Tenant Details

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/tenant/{tenantId}` | Bearer token | Admin |

Returns full tenant details including wallets, roles, and setup status.

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenantId` | UUID | Yes | Tenant identifier |

**Response (200):**
```json
{
  "id": "tenant-uuid",
  "name": "My Company LLC",
  "document": "12345678000190",
  "countryCode": "BR",
  "wallets": [
    {
      "address": "0x1234...abcd",
      "type": "operator"
    }
  ],
  "clientId": "client-uuid",
  "info": {
    "website": "https://example.com"
  },
  "roles": ["admin"],
  "operatorAddress": "0x1234...abcd",
  "setupDone": true
}
```

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | VALIDATION_ERROR | Missing or invalid fields in request body | Check required fields and types |
| 401 | UNAUTHORIZED | Missing or expired bearer token | Re-authenticate |
| 403 | FORBIDDEN | Insufficient role for endpoint | Verify user role matches endpoint requirement |
| 404 | NOT_FOUND | Tenant or plan not found | Verify tenantId/planId exist |
| 409 | CONFLICT | Duplicate configuration or plan assignment | Check existing state before creating |
| 422 | UNPROCESSABLE_ENTITY | Business rule violation (e.g., no credit card for paid plan) | Read error message for specific rule |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [SETTINGS_SKILL_INDEX](./SETTINGS_SKILL_INDEX.md) | Module entry point |
| [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) | Billing flow guide |
| [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) | Configuration flow guide |
| [AUTH_API_REFERENCE](../auth/AUTH_API_REFERENCE.md) | Authentication (required for all authenticated endpoints) |
