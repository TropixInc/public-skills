---
id: FLOW_SETTINGS_BILLING_MANAGEMENT
title: "Settings - Billing Management"
module: offpix/settings
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - settings
  - billing
  - plans
  - credit-card
  - subscription
  - usage
depends_on:
  - SETTINGS_SKILL_INDEX
  - SETTINGS_API_REFERENCE
---

# Billing Management

## Overview

This flow covers the complete billing lifecycle for a W3Block tenant: browsing available plans, setting up a credit card, subscribing to a plan, monitoring usage and costs, simulating future charges, reviewing billing cycle history, and cancelling subscriptions. It also covers administrative operations like registering usage events and triggering manual invoices.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token (Admin role) | ID service authentication |
| Credit card | Payment method registered with payment provider | External payment provider setup |

## Entities & Relationships

```
BillingPlanEntity (1) -----> (N) TenantPlanEntity -----> (1) TenantEntity
                                      |
                                      +-----> (N) TenantBillingLogEntity
                                      |
                                      +-----> (N) TenantPlanLogEntity (audit)

BillingUsageEntity (N) -----> (1) TenantEntity
BillingCurrencyExchangeRateEntity (standalone, for multi-currency)
```

**Key concepts:**
- Each tenant has exactly one active TenantPlanEntity
- Usage is tracked in BillingUsageEntity with both cycle and lifetime counters
- Every billing cycle produces a TenantBillingLogEntity with a detailed `finalReport`
- Plan changes are audited in TenantPlanLogEntity

---

## Flow: Plan Selection and Subscription

### Step 1: List Available Plans

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/plans` | None (Public) | - |

**Request:** No body required. Optionally filter with query parameters.

```
GET /billing/plans?isPublic=true&limit=10
```

**Response (200):**
```json
{
  "items": [
    {
      "id": "free-plan-uuid",
      "name": "Free",
      "basePrice": 0,
      "isDefault": true,
      "isPublic": true,
      "limits": {
        "nftContracts": { "monthly": 1, "hardPerMonth": 1 },
        "nftMints": { "monthly": 10, "hardPerMonth": 15 }
      },
      "features": {
        "enabledFeatures": ["MINT_ERC_721", "BASIC_TOKEN_PASS"]
      },
      "trial": { "testDays": 0 }
    },
    {
      "id": "pro-plan-uuid",
      "name": "Professional",
      "basePrice": 99.00,
      "isDefault": false,
      "isPublic": true,
      "limits": {
        "nftContracts": { "monthly": 10, "hardPerMonth": 15 },
        "nftMints": { "monthly": 500, "hardPerMonth": null }
      },
      "pricing": {
        "extraNftMintPrice": 0.50,
        "nftTransactionPrice": 0.10
      },
      "features": {
        "enabledFeatures": [
          "MINT_ERC_721", "MINT_ERC_20", "COMMERCE",
          "LOYALTY", "BASIC_TOKEN_PASS", "FULL_TOKEN_PASS"
        ]
      },
      "trial": { "testDays": 14 }
    }
  ],
  "meta": { "totalItems": 3, "currentPage": 1 }
}
```

**Notes:**
- Plans with `isDefault: true` are assigned to new tenants automatically
- Compare `enabledFeatures` arrays to understand feature differences between plans
- `trial.testDays` indicates how many free trial days are included
- Limits with `hardPerMonth: null` mean no hard cap (overage is billed per unit)

### Step 2: Set Credit Card

A valid credit card must be configured before subscribing to a paid plan.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/billing/{tenantId}/credit-card` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "creditCardId": "card_abc123"   // required
}
```

**Response (200):**
```json
{
  "tenantId": "tenant-uuid",
  "planId": "free-plan-uuid",
  "primaryCreditCardId": "card_abc123",
  "startCycleDate": "2026-03-01T00:00:00.000Z",
  "endCycleDate": "2026-03-31T23:59:59.000Z"
}
```

**Notes:**
- The `creditCardId` comes from your payment provider integration (e.g., Stripe card token)
- You can update the credit card at any time; the new card will be used for the next charge
- Only one primary card per tenant

### Step 3: Subscribe to a Plan

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/billing/{tenantId}/plan` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "planId": "pro-plan-uuid"       // required
}
```

**Complete Request:**
```json
{
  "planId": "pro-plan-uuid",      // required
  "coupon": "LAUNCH2026"          // optional -- discount coupon
}
```

**Response (200):**
```json
{
  "tenantId": "tenant-uuid",
  "planId": "pro-plan-uuid",
  "customLimits": null,
  "startCycleDate": "2026-03-31T00:00:00.000Z",
  "endCycleDate": "2026-04-30T23:59:59.000Z",
  "primaryCreditCardId": "card_abc123",
  "coupon": "LAUNCH2026"
}
```

**Notes:**
- Billing cycle dates reset to start from today
- A TenantPlanLogEntity is created with `oldPlanId`, `newPlanId`, and `userId`
- If switching from a paid plan, the previous cycle may be invoiced
- A credit card is required for non-free plans

---

## Flow: Monitoring Usage and Costs

### Step 4: Check Current Billing State

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/{tenantId}/state` | Bearer token (Admin) | - |

**Response (200):**
```json
{
  "plan": {
    "id": "pro-plan-uuid",
    "name": "Professional",
    "basePrice": 99.00
  },
  "tenantPlan": {
    "tenantId": "tenant-uuid",
    "planId": "pro-plan-uuid",
    "customLimits": null,
    "startCycleDate": "2026-03-01T00:00:00.000Z",
    "endCycleDate": "2026-03-31T23:59:59.000Z",
    "primaryCreditCardId": "card_abc123",
    "coupon": null
  },
  "features": {
    "enabledFeatures": ["MINT_ERC_721", "MINT_ERC_20", "COMMERCE", "LOYALTY"]
  }
}
```

### Step 5: View Usage Summary

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/{tenantId}/billing-summary` | Bearer token (Admin) | - |

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
      "count": 520,
      "includedInPlan": 500,
      "overage": 20,
      "overageCost": 10.00
    },
    {
      "type": "NFT_TRANSACTION",
      "count": 45,
      "cost": 4.50
    }
  ],
  "totalEstimatedCost": 113.50
}
```

**Notes:**
- `includedInPlan` shows how many units are covered by the base plan price
- `overage` is the count exceeding the plan limit
- `overageCost` = overage * per-unit price from plan pricing

### Step 6: List Detailed Usage Records

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/{tenantId}/billing-usages` | Bearer token (Admin) | - |

**Response (200):**
```json
{
  "items": [
    {
      "tenantId": "tenant-uuid",
      "type": "KEY_NFT_MINTED",
      "isLifeTime": false,
      "value": 520,
      "metadata": {}
    },
    {
      "tenantId": "tenant-uuid",
      "type": "KEY_NFT_MINTED",
      "isLifeTime": true,
      "value": 8400,
      "metadata": {}
    }
  ],
  "meta": { "totalItems": 22, "currentPage": 1 }
}
```

**Notes:**
- Records with `isLifeTime: false` are cycle counters (reset each billing period)
- Records with `isLifeTime: true` are cumulative counters (never reset)
- Unique constraint: `(tenantId, type, isLifeTime)` -- one record per combination

### Step 7: Simulate a Billing Event

Before performing an expensive operation, simulate its cost.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/{tenantId}/simulate-billing-event-usage` | Bearer token (Admin) | - |

**Request:**
```
GET /billing/{tenantId}/simulate-billing-event-usage?type=KEY_NFT_MINTED&value=50
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | BillingUsageKnownTypes | Yes | Usage event type |
| `value` | number | No | Units to simulate (default: 1) |

**Response (200):**
```json
{
  "type": "KEY_NFT_MINTED",
  "currentUsage": 480,
  "planLimit": 500,
  "simulatedAdditional": 50,
  "wouldExceedBy": 30,
  "estimatedOverageCost": 15.00
}
```

**Notes:**
- Does not modify any data -- purely read-only simulation
- Use this to show users projected costs in a confirmation dialog before minting, creating contracts, etc.

---

## Flow: Billing Cycle History

### Step 8: View Past Billing Cycles

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/{tenantId}/cycles` | Bearer token (Admin) | - |

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
      "planId": "pro-plan-uuid",
      "customLimits": null,
      "finalReport": [
        {
          "type": "KEY_NFT_MINTED",
          "count": 550,
          "includedInPlan": 500,
          "overage": 50,
          "overageCost": 25.00
        },
        {
          "type": "NFT_TRANSACTION",
          "count": 5,
          "cost": 0.50
        }
      ]
    }
  ],
  "meta": { "totalItems": 6, "currentPage": 1 }
}
```

**Notes:**
- `finalReport` contains the detailed breakdown per usage type for that cycle
- `paymentTries` > 1 indicates failed payment attempts before success
- `cyclePrice` is the calculated total; `paidValue` is what was actually charged (may differ with coupons)

---

## Flow: Subscription Cancellation

### Step 9: Cancel Subscription

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/billing/{tenantId}/cancel` | Bearer token (Admin) | - |

**Request:** No body required.

**Response (200):**
```json
{
  "tenantId": "tenant-uuid",
  "planId": "free-plan-uuid",
  "customLimits": null,
  "startCycleDate": "2026-03-31T00:00:00.000Z",
  "endCycleDate": "2026-04-30T23:59:59.000Z",
  "primaryCreditCardId": "card_abc123",
  "coupon": null
}
```

**Notes:**
- Reverts the tenant to the default (free) plan immediately
- Features from the paid plan are disabled right away
- The credit card remains on file but will not be charged
- An audit log entry (TenantPlanLogEntity) is created

---

## Flow: Administrative Operations

These endpoints are for platform operators, not tenant admins.

### Step 10: Register Usage Event (Integration/SuperAdmin)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/billing/{tenantId}/register-billing-usage` | Bearer token (SuperAdmin/Integration) | application/json |

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
  "value": 5,                     // optional -- units consumed (default: 1)
  "metadata": {                   // optional -- context for the event
    "contractId": "contract-uuid",
    "collectionId": "collection-uuid"
  }
}
```

**Notes:**
- Typically called by internal services (Integration API key) when billable events occur
- Increments both cycle and lifetime usage counters
- If a hard limit is exceeded, the operation may be rejected upstream

### Step 11: Trigger Manual Invoice (SuperAdmin)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/billing/{tenantId}/invoice-and-charge` | Bearer token (SuperAdmin) | - |

**Request:** No body required.

**Response (200):** TenantBillingLogEntity with the invoice details and `finalReport`.

**Notes:**
- Forces immediate cycle close, invoice generation, and credit card charge
- Creates a TenantBillingLogEntity record
- Increments `paymentTries` if the charge fails
- SuperAdmin only -- not accessible to tenant admins

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | VALIDATION_ERROR | Invalid `planId`, missing `creditCardId`, or bad usage type | Verify field values against enums and existing records |
| 401 | UNAUTHORIZED | Expired or missing token | Re-authenticate via ID service |
| 403 | FORBIDDEN | Non-admin trying to change plan or cancel | Ensure user has Admin role |
| 404 | PLAN_NOT_FOUND | `planId` does not exist | List plans first to get valid IDs |
| 404 | TENANT_NOT_FOUND | `tenantId` does not exist | Verify tenant ID |
| 422 | NO_CREDIT_CARD | Attempting paid plan without credit card | Set credit card first (Step 2) |
| 422 | HARD_LIMIT_EXCEEDED | Usage exceeds hard limit | Upgrade plan or wait for cycle reset |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Plan change fails with no credit card | Always set a credit card (Step 2) before changing to a paid plan (Step 3) |
| 2 | Usage appears zero after plan change | Cycle counters reset when the billing cycle resets; lifetime counters persist |
| 3 | Soft limit exceeded but operation succeeds | Soft limits are for billing, not enforcement. Only `hard*` limits block operations |
| 4 | Billing summary shows unexpected costs | Check overage pricing in the plan definition -- costs accumulate per unit beyond the included amount |
| 5 | Simulation shows zero cost but invoice is high | Simulation is point-in-time; usage between simulation and invoice end can add up |
| 6 | Cancellation still shows credit card | The card remains on file after cancellation; it simply won't be charged on the free plan |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Authentication | Required for all authenticated endpoints | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Tenant Configuration | Configure features checked via billing | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Contracts & Tokens | Operations that generate billing usage events | [CONTRACTS_SKILL_INDEX](../contracts/CONTRACTS_SKILL_INDEX.md) |
| Commerce | Product purchases generate billing events | [COMMERCE_SKILL_INDEX](../commerce/COMMERCE_SKILL_INDEX.md) |
