---
id: FLOW_LOYALTY_REWARDS_PAYMENTS
title: "Loyalty - Rewards, Cashback & Payments"
module: offpix/loyalty
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - loyalty
  - rewards
  - cashback
  - payments
  - staking
  - webhooks
depends_on:
  - FLOW_LOYALTY_CONTRACT_SETUP
  - FLOW_LOYALTY_BALANCE_OPERATIONS
---

# Loyalty - Rewards, Cashback & Payments

## Overview

This flow covers the automated reward system and payment operations: triggering deposits and cashback (including multilevel commissions), previewing and executing point-based payments, splitting rewards among multiple payees, staking management, and rolling back transactions. These operations build on the loyalty program and rules configured in the setup flow, and they power the point economy for in-store and online transactions.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token | ID service authentication |
| Active loyalty program | With ERC20 contract and at least one rule | [Contract Setup flow](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| User wallet | User must have a wallet address | User creation / wallet provisioning |
| Rule name | The `name` field of the rule to trigger | Loyalty rules configuration |

## Entities & Relationships

```
LoyaltiesEntity (1) ──→ (N) LoyaltiesRulesEntity
      │                          │
      │                          └── Matched by action name during deposit/cashback
      │
      └──→ (N) LoyaltiesDeferredEntity
                    │
                    ├── status: pending → started → success (or failed)
                    ├── status: deferred (scheduled for future)
                    ├── status: pool (grouped for batch)
                    ├── status: waiting_for_rollback → rollback
                    │
                    ├── parentMinterId (link to original mint deferred)
                    ├── parentTransferId (link to original transfer deferred)
                    └── stakingRuleId (link to staking rule)
```

**Key concepts:**
- **Deposit webhook** triggers reward distribution. The system finds the matching rule by `action` name, calculates the reward amount, and creates deferred transaction(s).
- **Cashback multilevel** processes a purchase amount through the cashback rule, distributing direct cashback to the buyer and indirect commissions up the referral chain.
- **Payment** is the reverse: users spend points to get a discount on a purchase. Points are burned/transferred from the user's wallet.
- **Deferred transactions** are the atomic records of all point movements. They track status, execution timing, metadata, and can be rolled back.
- **Staking** automatically distributes periodic rewards to users meeting holder requirements.

---

## Flow: Deposit Rewards (Webhook)

Use this flow to trigger point distribution based on an external event (e.g., a purchase, a referral, or any custom action).

### Step 1: Trigger Deposit

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/loyalties/webhook/deposit` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "user-uuid",
  "action": "welcome-bonus"
}
```

**Complete Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "user-uuid",
  "action": "cashback_multilevel",
  "amount": "100.00",
  "description": "Cashback from order #1234",
  "metadata": {
    "orderId": "order-uuid",
    "totalAmount": "100.00",
    "buyerName": "John Doe"
  }
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program to use |
| `userId` | UUID | Yes | User receiving the reward |
| `action` | string | Yes | Must match a rule's `name` field in the loyalty program |
| `amount` | string | No | Base amount for reward calculation. Required for `multiply`/`cashback` rules. |
| `description` | string | No | Human-readable transaction description |
| `metadata` | object | No | Passed to Handlebars templates in `descriptionTemplate`. Include any variables your templates reference (e.g., `{{orderId}}`, `{{buyerName}}`). |

**Response (200):**
```json
{
  "requestId": "request-uuid"
}
```

**Notes:**
- The `action` field is matched against the rule's `name` to determine which rule to apply. If no matching rule is found, a `ThereIsNoRuleForThisActionException` is thrown.
- The `requestId` in the response can be used later for rollback operations.
- For `add` type rules, the `value` field of the rule is used as the fixed amount regardless of the `amount` in the request.
- For `multiply` type rules, the request `amount` is multiplied by the rule's `value`.
- For `cashback` type rules, the request `amount` is multiplied by the rule's `value` (percentage).
- Deferred transactions are created with the `available` date expression from the rule, controlling when points actually become usable.

---

## Flow: Cashback Multilevel (Webhook)

Use this flow for purchase-based cashback with multilevel commissions.

### Step 2: Trigger Multilevel Cashback

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/loyalties/webhook/cashback-multilevel` | Bearer token (Integration) | application/json |

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "buyer-user-uuid",
  "amount": "500.00",
  "description": "Purchase cashback from order #5678",
  "operatorUserId": "operator-uuid",
  "metadata": {
    "orderId": "order-uuid",
    "productName": "Premium Subscription"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program |
| `userId` | UUID | Yes | The buyer (direct cashback recipient) |
| `amount` | string | Yes | Purchase amount |
| `description` | string | No | Transaction description |
| `operatorUserId` | UUID | No | The operator/store that processed the sale |
| `metadata` | object | No | Custom metadata |

**Response (200):**
```json
{
  "requestId": "request-uuid"
}
```

**How multilevel cashback works:**

Given a rule with `value: "0.05"` (5% cashback) and `cashbackConfigurations.indirectCashback: [0.03, 0.02, 0.01]`:

1. **Direct cashback** to the buyer: `amount * value` = 500 * 0.05 = 25 points
2. **L1 commission** to buyer's referrer: `amount * indirectCashback[0]` = 500 * 0.03 = 15 points
3. **L2 commission** to L1's referrer: `amount * indirectCashback[1]` = 500 * 0.02 = 10 points
4. **L3 commission** to L2's referrer: `amount * indirectCashback[2]` = 500 * 0.01 = 5 points
5. **Final recipient** (if configured): receives `finalRecipientRate` percentage
6. **Remaining points** (if chain is incomplete): go to `indirectCashbackUserIdToReceiveRemainingPoints`

Each level's points become available according to their respective `available` date expression.

**Override webhooks:** If `cashbackOverride`, `rateOverride`, or `overrideMultilevelCommission` are enabled in the cashback configuration, the system calls the configured external webhook to get custom values before calculating rewards.

---

## Flow: Cashback Preview

Preview cashback before triggering it.

### Step 3: Preview Cashback

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/loyalties/rewards/cashback/preview` | Bearer token | application/json |

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "100.00",
  "userId": "user-uuid",
  "action": "cashback_multilevel"
}
```

**Response (200):**
```json
{
  "amount": "100.00",
  "pointsCashback": "5",
  "currency": "BRL",
  "contractName": "Loyalty Points",
  "contractSymbol": "LPT",
  "currencyEquivalent": "0.50"
}
```

| Response Field | Type | Description |
|----------------|------|-------------|
| `amount` | string | Original purchase amount |
| `pointsCashback` | string | Points the user would earn |
| `currency` | string | Currency from loyalty payment settings |
| `contractName` | string | ERC20 token name |
| `contractSymbol` | string | ERC20 token symbol |
| `currencyEquivalent` | string | Currency value of the cashback points |

---

## Flow: Point-Based Payment

Use this flow for in-store or online payments where users spend loyalty points for a discount.

### Step 4: Preview Payment

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/loyalties/rewards/payment/preview` | Bearer token | application/json |

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "50.00",
  "points": "200",
  "userId": "user-uuid",
  "userCode": "ABC123"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program |
| `amount` | string | Yes | Total purchase amount (must be a positive number string) |
| `points` | string | Yes | Points the user wants to spend |
| `userId` | UUID | Yes | User making the payment |
| `userCode` | string | Yes | Verification code (used for in-store payments to confirm identity) |

**Response (200):**
```json
{
  "total": "50.00",
  "amount": "30.00",
  "points": "200",
  "pointsCashback": "0",
  "discount": "20.00",
  "currency": "BRL",
  "contractName": "Loyalty Points",
  "contractSymbol": "LPT"
}
```

| Response Field | Type | Description |
|----------------|------|-------------|
| `total` | string | Original purchase amount |
| `amount` | string | Amount remaining after discount |
| `points` | string | Points being spent |
| `pointsCashback` | string | Any cashback points earned from this payment |
| `discount` | string | Discount value in currency |
| `currency` | string | Currency code |
| `contractName` | string | Token name |
| `contractSymbol` | string | Token symbol |

### Step 5: Execute Payment

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/loyalties/rewards/payment` | Bearer token (Admin, LoyaltyOperator) | application/json |

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "50.00",
  "points": "200",
  "userId": "user-uuid",
  "userCode": "ABC123",
  "description": "In-store purchase at Location X"
}
```

**Response (200):**
```json
{
  "requestId": "request-uuid"
}
```

**Notes:**
- The `userCode` and `userId` must match the same user. A mismatch throws `UserNotMatchException`.
- The user must have sufficient balance. Otherwise, `InsufficientBalanceException` is thrown.
- The `operatorUserId` is automatically set from the authenticated user (the cashier/operator).
- Points are burned or transferred based on the loyalty program's `tokenTransferabilityMethod`.
- The `amount` field is validated to be a positive number string.

---

## Flow: Split Payees (Webhook)

Distribute rewards across multiple wallet addresses with percentage-based splits.

### Step 6: Split Rewards

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/loyalties/webhook/split-payees` | Bearer token (Integration) | application/json |

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "total": "1000",
  "payees": [
    {
      "walletAddress": "0xAddress1...",
      "percent": "0.60",
      "metadata": { "role": "owner" },
      "descriptionTemplate": "Owner share: {{amount}} points",
      "available": "1s",
      "sendEmail": true
    },
    {
      "walletAddress": "0xAddress2...",
      "percent": "0.25",
      "metadata": { "role": "partner" },
      "available": "7d",
      "sendEmail": false
    },
    {
      "walletAddress": "0xAddress3...",
      "percent": "0.15",
      "metadata": { "role": "affiliate" },
      "available": "30d",
      "sendEmail": false
    }
  ]
}
```

**Notes:**
- Each payee's amount is calculated as `total * percent`.
- Each payee can have its own `available` date expression (different vesting periods).
- The `percent` values do not need to sum to exactly 1.0 -- excess or remainder is silently ignored.
- Useful for revenue sharing models where different stakeholders receive different portions of reward points.

---

## Flow: Rollback Transactions

Roll back previously created deferred transactions (e.g., when an order is cancelled).

### Step 7: Request Rollback

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/loyalties/webhook/rollback` | Bearer token (Integration) | application/json |

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "requestId": "original-request-uuid",
  "metadata": {
    "reason": "Order #1234 cancelled by customer"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program |
| `requestId` | string | No | The `requestId` from the original deposit/cashback response |
| `metadata` | object | No | Rollback reason and context |

**Response (200):** Returns rollback processing information.

### Step 8: Check Rollback Status

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/loyalties/webhook/rollback/{requestId}` | Bearer token (Integration) | -- |

**Response (200):**
```json
{
  "completed": true
}
```

**Notes:**
- Rollback is asynchronous. Use the status endpoint to poll for completion.
- Rolled-back transactions change status from their current state to `waiting_for_rollback` then `rollback`.
- If the `requestId` matches an already-processed rollback, a `DuplicateRequestException` is thrown.

---

## Flow: Staking Rewards

Staking automatically distributes periodic rewards to users who meet holding requirements.

### Step 9: View Staking Summary

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/loyalties/users/{userId}/staking/{loyaltyId}` | Bearer token | -- |

**Response (200):**
```json
{
  "items": [
    {
      "id": "deferred-uuid",
      "amount": "50",
      "status": "deferred",
      "executeAt": "2026-04-08T00:00:00Z",
      "expiresAt": "2026-05-08T00:00:00Z",
      "withdrawableAt": "2026-04-15T00:00:00Z",
      "isExpired": false,
      "isWithdrawal": false
    }
  ],
  "meta": {
    "totalItems": 1,
    "totalPages": 1,
    "currentPage": 1,
    "itemsPerPage": 10
  },
  "summary": {
    "availableToRedeem": "150",
    "delivering": "50",
    "expired": "0",
    "received": "200"
  }
}
```

| Summary Field | Description |
|---------------|-------------|
| `availableToRedeem` | Points ready to be redeemed now |
| `delivering` | Points in transit (pending execution) |
| `expired` | Points that have expired |
| `received` | Total points received from staking |

### Step 10: Redeem Staking Rewards

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem` | Bearer token | -- |

**Request body:** None.

**Response (204):** No content.

**Notes:**
- Redeems all available staking rewards for the user.
- Staking rules can have `requireRedeem: true`, meaning rewards remain in `deferred` status until the user explicitly redeems them.
- If `redeemExpirationDays` is set on the staking rule, unredeemed rewards expire after that many days.

---

## Flow: Cashback Multilevel Report

View aggregated statistics about a user's multilevel cashback earnings.

### Step 11: Get Report

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/loyalties/users/{userId}/cashback-multilevel-report/{loyaltyId}` | Bearer token (User) | -- |

**Response (200):**
```json
{
  "reportPerLevel": [
    { "level": "1", "totalAmount": 500, "transactionsCount": 25 },
    { "level": "2", "totalAmount": 200, "transactionsCount": 15 },
    { "level": "3", "totalAmount": 50, "transactionsCount": 5 },
    { "level": "businessReferral", "totalAmount": 100, "transactionsCount": 5 },
    { "level": "areaReferral", "totalAmount": 30, "transactionsCount": 2 }
  ]
}
```

**Notes:**
- Data comes from a materialized view for performance. Cached for 5 minutes.
- Numeric levels correspond to `indirectCashback` array indices (1-based).
- Named levels (`businessReferral`, `areaReferral`) correspond to zone ambassador and business referrer rewards.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 404 | ThereIsNoRuleForThisActionException | `action` doesn't match any rule name | Verify rule names in the loyalty program |
| 404 | UserByCodeNotFoundException | User code not found | Verify the user code |
| 400 | UserNotMatchException | userId and userCode don't match | Ensure both identify the same user |
| 400 | InsufficientBalanceException | Not enough points for payment | Check balance before payment |
| 400 | LoyaltiesNotHaveContractException | Loyalty has no linked ERC20 contract | Link a published contract to the loyalty |
| 400 | DuplicateRequestException | Same rollback request sent twice | Use the original requestId to check status instead |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Deposit webhook returns 404 | The `action` field must exactly match a rule's `name` (lowercased, slugified). Check spelling and casing. |
| 2 | Points don't appear in balance after deposit | Points may have a delayed `available` date from the rule. Check the deferred transaction's `executeAt` date. |
| 3 | Payment preview succeeds but execution fails | Balance may have changed between preview and execution. Always re-check or execute immediately after preview. |
| 4 | Multilevel cashback doesn't reach all levels | Referral chain must be established (user relationships). Missing referrers cause remaining points to go to the fallback user. |
| 5 | Rollback not completing | Rollback is async. Poll `/rollback/{requestId}` instead of assuming immediate completion. |
| 6 | `amount` validation fails on payment | Amount must be a positive number string (e.g., `"50.00"`, not `"0"` or negative). |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Contract & Program Setup | Must be done first | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Balance Operations | Direct mint/transfer | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) |
| Commerce Orders | Triggers cashback on purchase | Commerce module |
