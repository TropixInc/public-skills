---
id: FLOW_LOYALTY_CONTRACT_SETUP
title: "Loyalty - Contract & Program Setup"
module: offpix/loyalty
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - loyalty
  - erc20
  - contract
  - setup
depends_on: []
---

# Loyalty - Contract & Program Setup

## Overview

This flow covers the end-to-end setup of a loyalty system: creating an ERC20 token contract, deploying it on-chain, creating a loyalty program linked to that contract, and configuring reward rules. This is the foundational flow -- all other loyalty operations (minting, transfers, payments, cashback) depend on this setup being complete.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token (Admin or SuperAdmin role) | ID service authentication |
| Blockchain chain ID | Target chain for contract deployment (e.g., `137` for Polygon) | Platform configuration |

## Entities & Relationships

```
ERC20ContractEntity (1) ←── (N) LoyaltiesEntity (1) ──→ (N) LoyaltiesRulesEntity
      │                            │
      │                            └── Loyalty program config: issuance method,
      │                                point precision, payment view settings
      │
      └── Token contract: name, symbol, chainId, status (draft → published)
```

**Key concepts:**
- An **ERC20 Contract** is the on-chain token. It starts in `draft` status and must be `published` (deployed) before tokens can be minted or transferred.
- A **Loyalty Program** defines how the token is used: how points are issued (mint vs transfer), how they are spent (transfer vs burn), and their currency equivalence.
- **Rules** define automated reward logic. Each rule has a type (`add`, `multiply`, `split`, `cashback`), a priority, and optional whitelist scoping.

---

## Flow: Create & Deploy ERC20 Contract

### Step 1: Create ERC20 Contract (Draft)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/erc20-contracts` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned"
}
```

**Complete Request (production example):**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "0",
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": []
}
```

**Response (201):**
```json
{
  "id": "erc20-contract-uuid",
  "companyId": "company-uuid",
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned",
  "status": "draft",
  "address": null,
  "operators": [],
  "roles": [],
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": [],
  "createdAt": "2026-04-01T10:00:00Z",
  "updatedAt": "2026-04-01T10:00:00Z"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Token name (sanitized to alphanumeric + spaces) |
| `symbol` | string | Yes | Token symbol (sanitized to alphanumeric) |
| `chainId` | integer | Yes | Target blockchain (137 = Polygon, 80001 = Mumbai) |
| `type` | enum | Yes | `classic`, `permissioned`, or `in_custody`. Frontend uses enum `TypeERC20Contract`. |
| `initialAmount` | string | No | Tokens to mint on deployment |
| `initialOwner` | ETH address | No | Address receiving initial supply |
| `isBurnable` | boolean | No | Allow burning tokens. Default: `false`. |
| `isWithdrawable` | boolean | No | Allow withdrawal to external wallets. Default: `false`. |
| `transferConfig` | array | No | Transfer restriction rules |

**Notes:**
- The `name` and `symbol` fields are automatically sanitized: diacritical marks removed, non-alphanumeric characters stripped.
- The contract is created in `draft` status. No on-chain interaction happens at this step.
- Most loyalty use cases use `permissioned` type, which allows role-based access control on the contract.

### Step 2: Estimate Gas (Optional)

Before publishing, estimate the deployment gas cost.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/erc20-contracts/{contractId}/estimate-gas` | Bearer token (Admin) | -- |

**Response (200):**
```json
{
  "totalGas": 2500000,
  "totalGasPrice": {
    "fast": "50000000000",
    "proposed": "30000000000",
    "safe": "20000000000"
  }
}
```

**Notes:**
- Gas prices are in wei. The `fast`, `proposed`, and `safe` tiers represent different confirmation speed priorities.
- Results are cached for 1 minute.
- The frontend displays this in the `NewERC20ContractGasController` component before the user confirms deployment.

### Step 3: Publish Contract (Deploy On-Chain)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-contracts/{contractId}/publish` | Bearer token (Admin) | -- |

**Request body:** None.

**Response (204):** No content.

**Notes:**
- This is an asynchronous operation. The contract status transitions: `draft` -> `publishing` -> `published` (or `failed`).
- After publishing, the contract's `address` field is populated with the deployed on-chain address.
- Poll `GET /{companyId}/erc20-contracts/{contractId}` to check when `status` becomes `published`.
- Once published, the contract cannot be updated (except for `transferConfig`).

---

## Flow: Create Loyalty Program

### Step 4: Create Loyalty Program

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/loyalties/admin` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "My Loyalty Program",
  "priority": 1,
  "tokenIssuanceMethod": "mint",
  "tokenTransferabilityMethod": "transfer",
  "pointPrecision": "integer",
  "paymentViewSettings": {
    "pointsEquivalent": {
      "currency": "BRL",
      "currencyValue": 10,
      "pointsValue": 1
    }
  },
  "active": true
}
```

**Complete Request (with ERC20 contract):**
```json
{
  "name": "My Loyalty Program",
  "contractId": "erc20-contract-uuid",
  "priority": 1,
  "tokenIssuanceMethod": "mint",
  "tokenTransferabilityMethod": "transfer",
  "pointPrecision": "integer",
  "paymentViewSettings": {
    "pointsEquivalent": {
      "currency": "BRL",
      "currencyValue": 10,
      "pointsValue": 1
    }
  },
  "image": "https://cdn.example.com/logo.png",
  "active": true
}
```

**Response (201):** Full loyalty entity with generated `id`.

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Program display name |
| `contractId` | UUID | No | Published ERC20 contract. Without this, on-chain operations won't work. |
| `priority` | integer | Yes | Program ordering priority (higher = first) |
| `tokenIssuanceMethod` | enum | Yes | `mint` (new tokens) or `transfer` (from source address) |
| `tokenIssuanceAddress` | ETH address | Conditional | Required when `tokenIssuanceMethod = "transfer"` |
| `tokenTransferabilityMethod` | enum | Yes | `transfer` (move tokens) or `burn` (destroy tokens) |
| `tokenTransferabilityAddress` | ETH address | Conditional | Required when `tokenTransferabilityMethod = "transfer"` |
| `pointPrecision` | enum | Yes | `integer` or `decimal` |
| `paymentViewSettings.pointsEquivalent.currency` | string | Yes | Currency code (e.g., `"BRL"`) |
| `paymentViewSettings.pointsEquivalent.currencyValue` | number | Yes | Points per 1 unit of currency (e.g., 10 = 10 points per 1 BRL) |
| `paymentViewSettings.pointsEquivalent.pointsValue` | number | Yes | Currency value per 1 point (e.g., 0.1 = 1 point worth 0.1 BRL) |
| `image` | URL string | No | Program logo URL |
| `active` | boolean | Yes | Enable/disable the program |

**Notes:**
- The `contractId` must reference an ERC20 contract belonging to the same `companyId`. If you attempt to use a contract from another company, it will throw an error.
- You can create a loyalty program without a contract and attach one later via PATCH, but no on-chain operations will work until a contract is linked.
- `paymentViewSettings` is critical for the payment flow -- it determines how points convert to/from currency values.

---

## Flow: Configure Reward Rules

### Step 5: Add Rule to Loyalty Program

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/loyalties/admin/{loyaltyId}/rules` | Bearer token (Admin) | application/json |

**Minimal Request (simple add rule):**
```json
{
  "value": "100",
  "available": "1s",
  "type": "add",
  "name": "welcome-bonus",
  "priority": 1
}
```

**Complete Request (cashback with multilevel commissions):**
```json
{
  "value": "0.05",
  "available": "7d",
  "type": "cashback",
  "name": "cashback_multilevel",
  "description": "5% cashback with 3-level commission",
  "priority": 1,
  "descriptionTemplate": "Cashback: {{value}} points from order {{orderId}}",
  "cashbackConfigurations": {
    "indirectCashback": [0.03, 0.02, 0.01],
    "indirectCashbackAvailable": "7d",
    "indirectCashbackDescriptionTemplate": "L{{level}} commission from {{buyerName}}",
    "indirectCashbackSendEmail": true,
    "finalRecipientRate": 0.01,
    "finalRecipientAvailable": "1s",
    "finalRecipientSendEmail": false,
    "remainingPointsAvailable": "1s",
    "remainingPointsSendEmail": false
  }
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `value` | string | Yes | Rule value. Meaning depends on type: fixed amount for `add`, multiplier for `multiply`, percentage for `cashback` (e.g., `"0.05"` = 5%). |
| `available` | string | Yes | Date expression for when points become available. `"1s"` = immediate, `"7d"` = 7 days. |
| `type` | enum | Yes | `add`, `multiply`, `split`, or `cashback` |
| `name` | string | Yes | Rule identifier (slugified). Used to match webhook actions. |
| `priority` | integer | Yes | Unique within the loyalty (except 0). Determines processing order. |
| `description` | string | No | Human-readable description |
| `descriptionTemplate` | string | No | Handlebars template for dynamic descriptions |
| `whitelistId` | UUID | No | Scope rule to a whitelist |
| `poolAddress` | ETH address | No | Pool address for grouped transactions |
| `cashbackConfigurations` | object | No | Required for `cashback` type. See API Reference for full schema. |

**Notes:**
- The `name` field is automatically slugified (lowercased, special chars replaced with hyphens).
- Priority must be unique per loyalty program. Priority 0 is exempt from the uniqueness constraint.
- When using a `whitelistId`, the whitelist must exist in the same tenant. The combination of `loyaltyId + whitelistId + name` must also be unique.
- `cashbackConfigurations.indirectCashback` is an array of rates per referral level. `[0.03, 0.02, 0.01]` means 3% for L1, 2% for L2, 1% for L3.
- Description templates use Handlebars syntax. Available variables depend on the webhook metadata passed during reward distribution.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 404 | LoyaltyNotFoundException | Loyalty ID not found in company | Verify loyalty ID and company |
| 400 | AlreadyExistRuleWithSamePriorityException | Duplicate priority in rules | Use a unique priority value |
| 400 | AlreadyExistRuleWithSameWhitelistException | Duplicate whitelist+name | Change whitelist or rule name |
| 404 | WhitelistNotFoundException | Whitelist not found | Create whitelist first |
| 400 | Validation error on date expression | Invalid `available` format | Use `ms`-compatible string like `1s`, `7d` |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Loyalty created without `contractId`, then mint/transfer fails | Always create and publish an ERC20 contract first, then link it to the loyalty program |
| 2 | Contract still in `draft` or `publishing` status when trying to use it | Poll the contract status until it reaches `published` before attempting operations |
| 3 | Rule priority collision causing 400 error | Check existing rules before adding. Priority 0 is exempt from uniqueness checks. |
| 4 | `name` and `symbol` containing accented characters get silently modified | The API strips diacritical marks and non-alphanumeric characters. Use ASCII-only names. |
| 5 | `cashbackConfigurations` ignored for non-cashback rule types | Only `cashback` type rules process cashback configurations. Other types ignore the field. |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Mint & Transfer Tokens | Next step after setup | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) |
| Rewards & Payments | Requires rules configured | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) |
| Auth (Sign-in) | Required before any API call | Auth module |
