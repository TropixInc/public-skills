---
id: FLOW_LOYALTY_BALANCE_OPERATIONS
title: "Loyalty - Balance Operations (Mint, Transfer, Burn)"
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
  - mint
  - transfer
  - burn
  - balance
depends_on:
  - FLOW_LOYALTY_CONTRACT_SETUP
---

# Loyalty - Balance Operations (Mint, Transfer, Burn)

## Overview

This flow covers direct ERC20 token operations: minting new tokens, transferring tokens between wallets, burning tokens, and checking balances. These are the core admin operations for manually managing loyalty point balances, independent of the automated reward/cashback system. The frontend groups these under the "Loyalty" section with dedicated modals for each action.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token (Admin role for mint/admin-transfer) | ID service authentication |
| Published ERC20 contract | Contract in `published` status | [Contract Setup flow](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Loyalty program (for balance queries) | Active loyalty with linked contract | [Contract Setup flow](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Wallet address | Recipient/source ETH address | User wallet from the ID service |

## Entities & Relationships

```
ERC20ContractEntity ──→ Erc20ActionEntity (on-chain actions: mint, transfer, burn)
                              │
LoyaltiesEntity ──→ LoyaltiesDeferredEntity (transaction records)
      │
      └── References ERC20ContractEntity via contractId
```

**Key concepts:**
- **Mint** creates new tokens and sends them to a wallet address. Only Admins can mint.
- **Transfer (Admin)** moves tokens from one wallet to another. Requires Admin role.
- **Transfer (User)** allows users to transfer their own tokens. Subject to `transferConfig` restrictions on the contract.
- **Burn** destroys tokens from a wallet. Available to Admins and the token owner.
- **Balance** is queried through the loyalty users endpoint, which aggregates on-chain balance with loyalty program metadata.

---

## Flow: Mint Tokens

### Step 1: Find User Wallet Address

Before minting, you need the recipient's wallet address. The frontend uses the `useGetUserByWallet` hook to look up users by wallet address, or you can get it from the user profile.

To find a user by wallet address (via the ID service SDK):

```
GET /tenants/{companyId}/users?address={walletAddress}&includeOwnerInfo=true
```

### Step 2: Mint Tokens

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/mint` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "to": "0x1234567890abcdef1234567890abcdef12345678",
  "amount": "100"
}
```

**Complete Request (production example from frontend):**
```json
{
  "to": "0x1234567890abcdef1234567890abcdef12345678",
  "amount": "500",
  "metadata": {
    "description": "Manual bonus points for loyalty program"
  },
  "sendEmail": false,
  "available": "1s"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string (ETH address) | Yes | Recipient wallet address |
| `amount` | string | Yes | Number of tokens to mint (string to handle large numbers) |
| `metadata` | object | No | Arbitrary metadata. Frontend passes `{ description: "..." }`. |
| `sendEmail` | boolean | No | Send notification email. Frontend defaults to `false`. |
| `available` | string (date expression) | No | When tokens become available. Default behavior: immediate. |

**Response (204):** No content.

**Notes:**
- The `contractId` in the URL is the ERC20 contract UUID (not the on-chain address).
- The contract must be in `published` status.
- Minting is an on-chain operation and may take a few seconds to confirm.
- If the contract has a `maxSupply` set and this mint would exceed it, the operation will fail.
- The frontend's `MintModal` component collects the `to` address (via wallet search), `amount`, and optional `description`.

---

## Flow: Transfer Tokens (Admin)

### Step 3: Admin Transfer

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/transfer/admin` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "from": "0xSourceWallet...",
  "to": "0xDestinationWallet...",
  "amount": "50"
}
```

**Complete Request (production example from frontend):**
```json
{
  "from": "0xSourceWallet...",
  "to": "0xDestinationWallet...",
  "amount": "50",
  "metadata": {
    "description": "Transfer between user wallets"
  },
  "sendEmail": false,
  "available": "1s"
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string (ETH address) | Yes | Source wallet address |
| `to` | string (ETH address) | Yes | Destination wallet address |
| `amount` | string | Yes | Amount to transfer |
| `metadata` | object | No | Arbitrary metadata |
| `sendEmail` | boolean | No | Send notification email. Frontend defaults to `false`. |
| `available` | string (date expression) | No | When transferred tokens become available. Frontend sends `"1s"`. |

**Response (204):** No content.

**Notes:**
- Admin transfers bypass transfer config restrictions.
- The source wallet must have sufficient balance.
- The frontend's `TransferModal` component collects `from`, `to` (via wallet search), `amount`, and optional `description`.

---

## Flow: Transfer Tokens (User)

### Step 4: User Transfer

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/transfer/user` | Bearer token (User) | application/json |

The request body is identical to admin transfer. However:
- The `from` address must be the authenticated user's own wallet.
- Transfer config restrictions on the contract are enforced (e.g., `max_limit`, `percentage`, `period`).
- Users can only transfer their own tokens.

---

## Flow: Burn Tokens

### Step 5: Burn Tokens

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{companyId}/erc20-tokens/{contractId}/burn` | Bearer token (Admin or User) | application/json |

**Request:**
```json
{
  "from": "0xWalletAddress...",
  "amount": "25",
  "metadata": {
    "description": "Burn excess tokens"
  },
  "sendEmail": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string (ETH address) | Yes | Wallet to burn tokens from |
| `amount` | string | Yes | Amount to burn |
| `metadata` | object | No | Arbitrary metadata |
| `sendEmail` | boolean | No | Send notification email |
| `available` | string | No | Date expression for when burn takes effect |

**Response (204):** No content.

**Notes:**
- The ERC20 contract must have `isBurnable: true`.
- Users can only burn their own tokens.

---

## Flow: Check Balances

### Step 6a: Get All Loyalty Balances for a User

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/loyalties/users/balance/{userId}` | Bearer token | -- |

**Response (200):**
```json
[
  {
    "balance": "1500",
    "loyaltyId": "loyalty-uuid",
    "contractId": "erc20-contract-uuid",
    "pointsPrecision": "integer",
    "image": "https://cdn.w3block.io/loyalty-logo.png",
    "name": "Loyalty Points",
    "symbol": "LPT"
  }
]
```

**Notes:**
- Returns balances for ALL active loyalty programs the user participates in.
- Cached for 10 seconds. After minting/transferring, wait briefly before checking updated balance.
- Regular users can only query their own balance. Admins and LoyaltyOperators can query any user.

### Step 6b: Get Balance for a Specific Loyalty

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/loyalties/users/balance/{userId}/{loyaltyId}` | Bearer token | -- |

**Response (200):** Single balance object (same shape as above but not wrapped in an array).

**Notes:**
- Cached for 5 seconds (shorter than the all-balances endpoint).
- Useful for real-time balance checks in payment flows.

---

## Flow: View Transaction History

### Step 7a: Contract Transaction History

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/erc20-tokens/{contractId}/history` | Bearer token (Admin, LoyaltyOperator) | -- |

Returns paginated list of all on-chain actions (mints, transfers, burns) for a contract.

### Step 7b: User Transaction History

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/erc20-tokens/{contractId}/history/{userId}` | Bearer token | -- |

Returns paginated list of on-chain actions for a specific user on a contract.

### Step 7c: Get Specific Action Details

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/erc20-tokens/{contractId}/history/action/{actionId}` | Bearer token (Admin) | -- |

Returns details of a single on-chain action.

### Step 7d: Deferred Transactions (Loyalty Layer)

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/{companyId}/loyalties/users/deferred` | Bearer token | -- |

Returns paginated deferred transactions with extensive filtering. See the [API Reference](./LOYALTY_API_REFERENCE.md) for the full list of query parameters.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 403 | ForbiddenException | User trying to access another user's data | Users can only access their own balance/history |
| 400 | Contract not published | Attempting operations on a draft contract | Publish the contract first |
| 400 | Insufficient balance | Transfer/burn amount exceeds balance | Check balance before operating |
| 400 | Transfer config violation | User transfer exceeds configured limits | Check contract's `transferConfig` rules |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Balance not updated immediately after mint | Balances are cached (5-10 seconds). Wait or implement polling on the client. |
| 2 | Using on-chain address instead of contract UUID in URL | API endpoints use the contract UUID, not the deployed blockchain address |
| 3 | Minting to user ID instead of wallet address | The `to` field requires an Ethereum address, not a user UUID. Look up the user's wallet first. |
| 4 | Frontend sends `sendEmail: false` but emails still sent | The `sendEmail` field defaults to `true` in the backend. Always explicitly set it to `false` if you don't want emails. |
| 5 | User transfer fails with no clear error | Check the contract's `transferConfig` array -- restrictions like `max_limit` or `period` may be blocking the transfer |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Contract & Program Setup | Must be done first | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Rewards & Payments | Automated reward distribution | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) |
