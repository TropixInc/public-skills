---
id: FLOW_CONTRACTS_ERC20_LIFECYCLE
title: "Contracts - ERC20 Token Lifecycle"
module: offpix/contracts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - erc20
  - fungible
  - loyalty
  - transfer-rules
  - mint
  - burn
depends_on:
  - CONTRACTS_API_REFERENCE
---

# ERC20 Token Lifecycle

## Overview

ERC20 contracts in W3Block represent fungible tokens — used for loyalty points, in-app currencies, and custom tokens. This flow covers creating, deploying, and operating ERC20 contracts, including the sophisticated transfer handler system for permissioned contracts. In the frontend, ERC20 is managed under **Loyalty > Contracts** (`/dash/loyalty/contracts`).

W3Block supports three ERC20 types:
- **Classic** — standard ERC20, direct blockchain transfers
- **Permissioned** — transfers go through a rule chain (fees, limits, periods)
- **In Custody** — company holds tokens on behalf of users

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Company wallet | Default owner address | Company settings |

## State Machine

Same as NFT contracts:

```
DRAFT ──[publish]──→ PUBLISHING ──[confirms]──→ PUBLISHED
  ↑                                    │
  └──────────[failed]──────────────────┘
              FAILED ──[retry]──→ PUBLISHING
```

---

## Flow: Create and Deploy ERC20 Contract

### Step 1: Create Draft

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/erc20-contracts` | Bearer (Admin) | application/json |

**Minimal Request (classic):**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "classic"
}
```

**Complete Request (permissioned with transfer rules):**
```json
{
  "name": "Platform Token",
  "symbol": "PTK",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "1000000",
  "initialOwner": "0xCompanyWallet...",
  "isBurnable": true,
  "isWithdrawable": false,
  "maxSupply": "10000000",
  "transferConfig": [
    {
      "model": "PERCENTAGE",
      "value": "0.05"
    },
    {
      "model": "MAX_LIMIT",
      "value": "100"
    },
    {
      "model": "PERIOD",
      "value": "10",
      "period": "7d"
    }
  ]
}
```

**Response (201):** Full ERC20 contract object with `status: "draft"`.

### Step 2: Estimate Gas

```
GET /{companyId}/erc20-contracts/{contractId}/estimate-gas
```

### Step 3: Publish

```
PATCH /{companyId}/erc20-contracts/{contractId}/publish
```

**Response:** 204 No Content. Contract deploys asynchronously.

---

## Transfer Rules (Permissioned & In Custody)

When a user calls the `/transfer/user` endpoint on a permissioned or in-custody contract, the transfer goes through a **handler chain** that evaluates rules in order:

```
Request → FreeTransferHandler → MaxLimitTransferHandler → PeriodTransferHandler
            → PercentageTransferHandler → FixedTransferHandler → Execute
```

Each handler either processes the transfer (if its rule matches) or passes to the next. If no handler accepts, the transfer is rejected.

### Rule: FREE

**Config:** `{ "model": "FREE", "value": "free" }`

| Policy | Behavior |
|--------|----------|
| `free` | Transfer allowed with no restrictions |
| `forbidden` | All user transfers blocked |
| `erc20ReceiverOnly` | Only addresses with `keyErc20Receiver` role can receive |

### Rule: MAX_LIMIT

**Config:** `{ "model": "MAX_LIMIT", "value": "100" }`

Limits the total number of transfers a user can make. Counts all successful (non-failed) transfers for the user.

### Rule: PERIOD

**Config:** `{ "model": "PERIOD", "value": "10", "period": "7d" }`

Limits transfers within a time window. Value = max transfers, period = time window (e.g., "7d", "24h", "30d").

### Rule: PERCENTAGE

**Config:** `{ "model": "PERCENTAGE", "value": "0.05" }`

Deducts a percentage fee from each transfer. Creates **two blockchain transactions**:

1. Recipient receives: `amount * (1 - percentage)`
2. Company receives: `amount * percentage` (sent to `company.defaultOwnerAddress`)

**Example:** Transfer 1000 tokens with 5% fee → recipient gets 950, company gets 50.

### Rule: FIXED

**Config:** `{ "model": "FIXED", "value": "10" }`

Deducts a fixed amount from each transfer. Only applies if transfer amount > fixed fee. Creates **two blockchain transactions**:

1. Recipient receives: `amount - fixed`
2. Company receives: `fixed` amount

**Example:** Transfer 1000 tokens with fixed fee of 10 → recipient gets 990, company gets 10.

### Multiple Rules

Rules can be combined. They are evaluated in chain order — the first matching rule processes the transfer:

```json
"transferConfig": [
  { "model": "MAX_LIMIT", "value": "100" },
  { "model": "PERIOD", "value": "10", "period": "7d" },
  { "model": "PERCENTAGE", "value": "0.05" }
]
```

This means: max 100 total transfers, max 10 per week, with 5% fee on each.

### Update Transfer Rules (Post-Deploy)

```
PATCH /{companyId}/erc20-contracts/{contractId}/transfer-config
```

```json
{
  "transferConfig": [
    { "model": "PERCENTAGE", "value": "0.03" }
  ]
}
```

Transfer rules can be updated even on published contracts.

---

## Flow: Mint ERC20 Tokens

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/mint` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "to": "0xRecipientWallet...",
  "amount": "5000",
  "metadata": {
    "reason": "Welcome bonus",
    "campaign": "onboarding-q1"
  },
  "sendEmail": true,
  "available": "1s"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | Yes | Recipient wallet address |
| `amount` | string | Yes | Amount to mint |
| `metadata` | object | No | Custom transaction metadata |
| `sendEmail` | boolean | No | Send notification (default: true) |
| `available` | string | No | Availability delay (e.g., "1s", "24h") |

---

## Flow: Transfer ERC20 Tokens

### Admin Transfer (Bypasses Rules)

```
PATCH /{companyId}/erc20-tokens/{contractId}/transfer/admin
```

```json
{
  "from": "0xSenderWallet...",
  "to": "0xRecipientWallet...",
  "amount": "1000",
  "metadata": { "reason": "Manual adjustment" },
  "sendEmail": true
}
```

Admin transfers bypass all transfer config rules — they execute directly on the blockchain.

### User Transfer (Rules Applied)

```
PATCH /{companyId}/erc20-tokens/{contractId}/transfer/user
```

Same body. For `permissioned` and `in_custody` contracts, the transfer handler chain is evaluated. For `classic` contracts, it executes directly.

---

## Flow: Burn ERC20 Tokens

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/burn` | Bearer (Admin, User) | application/json |

**Request:**
```json
{
  "from": "0xBurnerWallet...",
  "amount": "500",
  "metadata": { "reason": "Redemption for reward" },
  "sendEmail": true
}
```

Requires `isBurnable: true` on the contract.

---

## Flow: View Transaction History

### All Transactions

```
GET /{companyId}/erc20-tokens/{contractId}/history
```

Paginated history of all actions (mint, transfer, burn). Cached for 10 seconds.

### Specific Action

```
GET /{companyId}/erc20-tokens/{contractId}/history/action/{actionId}
```

### User's Transactions

```
GET /{companyId}/erc20-tokens/{contractId}/history/{userId}
```

Users can only see their own history. Admins can see any user's history.

### Operator Activity

```
GET /{companyId}/erc20-tokens/{contractId}/history/operator/{operatorId}
```

For LoyaltyOperator role — see all operations performed by a specific operator.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | YouCannotTransferThisTokenException | Transfer rules prevent operation | Check FREE policy, limits, period |
| 400 | Contract not published | Trying to operate on draft/failed contract | Publish the contract first |
| 400 | Not burnable | Trying to burn on non-burnable contract | Set `isBurnable: true` at creation |
| 403 | Insufficient permissions | User can't perform this operation | Check roles and transfer policies |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | User transfer rejected | Check which transfer config rules are active. The chain evaluates in order |
| 2 | Percentage fee creates extra transaction | This is by design — fee goes to company wallet as a separate blockchain action |
| 3 | Transfer limit reached | MAX_LIMIT counts all successful transfers. Period-based limits reset after the window |
| 4 | `available` delay confusing | Tokens are minted immediately but may not be spendable until the delay expires |
| 5 | Admin vs user transfer | Admin transfer always bypasses rules. Use `/transfer/user` to enforce transfer config |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| NFT Contracts | Non-fungible token lifecycle | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Token Operations | NFT mint/transfer/burn | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| API Reference | Full endpoint details | [CONTRACTS_API_REFERENCE](./CONTRACTS_API_REFERENCE.md) |
