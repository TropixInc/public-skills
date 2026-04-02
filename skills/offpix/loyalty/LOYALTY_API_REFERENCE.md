---
id: LOYALTY_API_REFERENCE
title: "Loyalty - API Reference"
module: offpix/loyalty
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - loyalty
  - erc20
  - cashback
  - staking
  - rewards
  - api-reference
---

# Loyalty API Reference

Complete endpoint reference for the W3Block Loyalty module. This module spans four resource groups -- Loyalty Programs (admin CRUD), ERC20 Contracts (token creation and deployment), ERC20 Tokens (mint/transfer/burn operations), and Loyalty Users (balances, transactions, staking, rewards) -- all served by the KEY (pixway-registry) backend.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://api.w3block.io` |
| Staging | *(staging environment available — use staging base URL)* |
| Swagger | https://api.w3block.io/docs/ |

## Authentication

Endpoints marked **Auth: Bearer** require:

```
Authorization: Bearer {accessToken}
```

Loyalty endpoints use these role levels:
- **SuperAdmin / Admin** -- full access to all loyalty and ERC20 operations
- **LoyaltyOperator** -- can list loyalties, view balances, preview/execute payments
- **User** -- can view own balance and transaction history
- **Integration** -- service-to-service webhook endpoints

---

## Enums

### LoyaltiesRuleType

| Value | Description |
|-------|-------------|
| `add` | Adds a fixed amount of points |
| `multiply` | Multiplies the base amount by a factor |
| `split` | Splits reward across multiple payees |
| `cashback` | Cashback with multilevel commission support |

### LoyaltiesDeferredStatus

| Value | Description |
|-------|-------------|
| `pending` | Awaiting execution |
| `started` | Execution in progress |
| `success` | Successfully executed on-chain |
| `failed` | Execution failed |
| `deferred` | Scheduled for future execution |
| `pool` | Grouped in a pool for batch processing |
| `waiting_for_rollback` | Rollback requested, awaiting processing |
| `rollback` | Successfully rolled back |

### TokenIssuanceMethod

| Value | Description |
|-------|-------------|
| `mint` | New tokens are minted to the recipient |
| `transfer` | Tokens are transferred from a source address (`tokenIssuanceAddress`) |

### TokenTransferabilityMethod

| Value | Description |
|-------|-------------|
| `transfer` | Tokens are transferred between wallets |
| `burn` | Tokens are burned from the source wallet |

### PointPrecisionOptions

| Value | Description |
|-------|-------------|
| `integer` | Points are whole numbers only |
| `decimal` | Points support decimal precision |

### LoyaltiesAction

| Value | Description |
|-------|-------------|
| `multiply` | Multiply base amount |
| `sum` | Add to base amount |
| `percentage` | Percentage of base amount |

### LoyaltiesProfile

| Value | Description |
|-------|-------------|
| `cashback` | Cashback-type rewards |
| `commission` | Commission-type rewards |

### ActionSubtype

| Value | Description |
|-------|-------------|
| `cashback` | Direct cashback |
| `commission_l1` | Level 1 commission |
| `commission_l2` | Level 2 commission |
| `commission_l3` | Level 3 commission |
| `commission_l4` | Level 4 commission |
| `commission_lx` | Generic level commission (any level beyond L4) |
| `finalRecipient` | Final recipient of remaining points |
| `businessReferral` | Business referral reward |
| `areaReferral` | Area/zone referral reward |

### LoyaltyStakingAmountType

| Value | Description |
|-------|-------------|
| `fixed` | Fixed amount per staking cycle |
| `holder_factor` | Amount calculated based on holder's balance multiplied by a factor |

### ERC20ContractType

Frontend name: `TypeERC20Contract`

| Value | Description |
|-------|-------------|
| `classic` | Standard ERC20 token |
| `permissioned` | Permissioned ERC20 with role-based access |
| `in_custody` | Custodial ERC20 managed by the platform |

### ContractStatus

| Value | Description |
|-------|-------------|
| `draft` | Contract created but not deployed |
| `publishing` | Deployment in progress |
| `published` | Successfully deployed on-chain |
| `failed` | Deployment failed |

### Erc20TransferModel

| Value | Description |
|-------|-------------|
| `max_limit` | Maximum transfer limit |
| `percentage` | Percentage-based transfer limit |
| `fixed` | Fixed transfer amount |
| `period` | Period-based transfer restriction |
| `free` | No transfer restrictions |

---

## Entities & Relationships

```
ERC20ContractEntity (1) ←── (N) LoyaltiesEntity (1) ──→ (N) LoyaltiesRulesEntity
                                     │
                                     ├──→ (N) LoyaltiesDeferredEntity (transactions)
                                     │           │
                                     │           └──→ Erc20ActionEntity (on-chain action)
                                     │
                                     └──→ (N) LoyaltiesStakingRulesEntity
```

**Key concepts:**
- An **ERC20 Contract** is the on-chain token contract (name, symbol, chainId). It must be deployed (published) before loyalty operations can execute on-chain.
- A **Loyalty** (program) links a tenant (`companyId`) to an ERC20 contract and defines how tokens are issued and transferred, plus payment view settings (points-to-currency equivalence).
- **Loyalty Rules** define reward logic for a loyalty program. Each rule has a type (add, multiply, split, cashback), a priority, and an optional whitelist scope.
- **Deferred Transactions** (`LoyaltiesDeferredEntity`) track all point movements (pending, executed, rolled back). They record from/to addresses, amounts, execution times, and metadata.
- **Staking Rules** define periodic reward distributions based on token holdings, with configurable requirements (collection holders, ERC20 holders) and amounts (fixed or holder-factor).

---

## Date Expression Format

Several fields use a "date expression" format for scheduling when points become available. This supports:

- **ms-style duration:** `1s`, `30m`, `1h`, `7d`, `30d` (millisecond string parsed by the `ms` library)
- **Duration with hour targeting:** `7d:hour:10` (7 days from now, snapped to 10:00 AM)
- **Cron expression:** Standard cron for recurring schedules (used in staking rules)

Examples: `1s` (immediate), `24h`, `7d`, `30d:hour:0` (30 days, midnight).

---

## Endpoints: Loyalty Admin (Programs & Rules)

Base path: `/{companyId}/loyalties/admin`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | SuperAdmin, Admin | Create a new loyalty program |
| 2 | GET | `/` | Bearer | SuperAdmin, Admin, LoyaltyOperator | List all loyalty programs (paginated) |
| 3 | GET | `/:loyaltyId` | Bearer | SuperAdmin, Admin | Get loyalty program by ID |
| 4 | PATCH | `/:loyaltyId` | Bearer | SuperAdmin, Admin | Update a loyalty program |
| 5 | POST | `/:loyaltyId/rules` | Bearer | SuperAdmin, Admin | Add a rule to a loyalty program |
| 6 | GET | `/:loyaltyId/rules/:ruleId` | Bearer | SuperAdmin, Admin | Get a specific rule |
| 7 | PATCH | `/:loyaltyId/rules/:ruleId` | Bearer | SuperAdmin, Admin | Update a rule |
| 8 | DELETE | `/:loyaltyId/rules/:ruleId` | Bearer | SuperAdmin, Admin | Delete a rule (soft-delete) |

### POST /{companyId}/loyalties/admin

Create a new loyalty program.

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

**Complete Request:**
```json
{
  "name": "My Loyalty Program",
  "contractId": "erc20-contract-uuid",
  "priority": 1,
  "tokenIssuanceMethod": "mint",
  "tokenIssuanceAddress": null,
  "tokenTransferabilityMethod": "transfer",
  "tokenTransferabilityAddress": null,
  "pointPrecision": "integer",
  "paymentViewSettings": {
    "pointsEquivalent": {
      "currency": "BRL",
      "currencyValue": 10,
      "pointsValue": 1
    }
  },
  "image": "https://cdn.example.com/loyalty-logo.png",
  "active": true
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Program display name |
| `contractId` | UUID | No | ERC20 contract ID. Must be a valid published contract in the same company. If null, loyalty operates without on-chain backing. |
| `priority` | integer | Yes | Ordering priority (higher = processed first) |
| `tokenIssuanceMethod` | TokenIssuanceMethod | Yes | How tokens are created for rewards |
| `tokenIssuanceAddress` | string (ETH address) | Conditional | Required when `tokenIssuanceMethod` is `transfer`. Source wallet address. |
| `tokenTransferabilityMethod` | TokenTransferabilityMethod | Yes | How tokens move during payments/spending |
| `tokenTransferabilityAddress` | string (ETH address) | Conditional | Required when `tokenTransferabilityMethod` is `transfer`. |
| `pointPrecision` | PointPrecisionOptions | Yes | Whether points are integer or decimal |
| `paymentViewSettings` | object | Yes | Defines points-to-currency equivalence |
| `paymentViewSettings.pointsEquivalent.currency` | string | Yes | Currency code (e.g., `"BRL"`, `"USD"`) |
| `paymentViewSettings.pointsEquivalent.currencyValue` | number | Yes | How many points equal 1 unit of currency (e.g., 10 points = 1 BRL) |
| `paymentViewSettings.pointsEquivalent.pointsValue` | number | Yes | Value of 1 point in the currency (e.g., 1 point = 0.1 BRL) |
| `image` | string (URL) | No | Logo/image URL for the loyalty program |
| `active` | boolean | Yes | Whether the program is active |

**Response (201):** Full loyalty entity with generated `id`, `companyId`, `createdAt`, `updatedAt`.

**Notes:**
- If `contractId` is provided, the ERC20 contract must exist and belong to the same `companyId`.
- Image URLs from Cloudinary are automatically converted to W3Block CDN URLs.

### GET /{companyId}/loyalties/admin

List all loyalty programs with pagination.

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 10 | Items per page |
| `search` | string | -- | Search in name |
| `sortBy` | string | `createdAt` | Sort column |
| `orderBy` | `ASC` \| `DESC` | `DESC` | Sort direction |

**Response (200):** Paginated list of loyalty entities. Each item includes nested `rules` array.

### POST /{companyId}/loyalties/admin/:loyaltyId/rules

Add a rule to a loyalty program.

**Minimal Request (add type):**
```json
{
  "value": "100",
  "available": "1s",
  "type": "add",
  "name": "welcome-bonus",
  "priority": 1
}
```

**Complete Request (cashback type with multilevel):**
```json
{
  "value": "0.05",
  "available": "7d",
  "type": "cashback",
  "name": "cashback_multilevel",
  "description": "5% cashback on purchases with multilevel commissions",
  "priority": 1,
  "whitelistId": "whitelist-uuid",
  "descriptionTemplate": "Cashback of {{value}} points from purchase by {{buyerId}}",
  "poolAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "cashbackConfigurations": {
    "indirectCashback": [0.03, 0.02, 0.01],
    "indirectCashbackAvailable": "7d",
    "indirectCashbackDescriptionTemplate": "Commission L{{level}} from {{buyerId}}",
    "indirectCashbackUserIdToReceiveRemainingPoints": "fallback-user-uuid",
    "indirectCashbackRemainingPointsSendEmail": true,
    "indirectCashbackSendEmail": true,
    "finalRecipientRate": 0.01,
    "finalRecipientAvailable": "1s",
    "finalRecipientDescriptionTemplate": "Final recipient points from {{buyerId}}",
    "finalRecipientSendEmail": true,
    "zoneAmbassadorRate": "0.025",
    "zoneAmbassadorAvailable": "7d",
    "zoneAmbassadorSendEmail": true,
    "zoneAmbassadorDescriptionTemplate": "Zone ambassador reward",
    "businessReferrerRate": "0.025",
    "businessReferrerAvailable": "7d",
    "businessReferrerSendEmail": true,
    "businessReferrerDescriptionTemplate": "Business referrer reward",
    "remainingPointsAvailable": "1s",
    "remainingPointsSendEmail": false,
    "sendToIdWebhookNotification": false,
    "cashbackOverride": {
      "enabled": false,
      "url": "https://example.com/webhook",
      "path": "data.cashback"
    },
    "rateOverride": {
      "enabled": false,
      "url": "https://example.com/webhook",
      "path": "data.rate"
    }
  }
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `value` | string | Yes | Reward value. Interpretation depends on `type`: fixed amount for `add`, multiplier for `multiply`, percentage for `cashback` (e.g., `"0.05"` = 5%). |
| `available` | string (date expression) | Yes | When points become available after earning (e.g., `"1s"` = immediate, `"7d"` = 7 days) |
| `type` | LoyaltiesRuleType | Yes | Rule type: `add`, `multiply`, `split`, `cashback` |
| `name` | string | Yes | Rule identifier (slugified, lowercase). Used to match webhook actions. Must be unique per loyalty+priority combination. |
| `description` | string | No | Human-readable description |
| `priority` | integer | Yes | Execution priority. Must be unique within the same loyalty program (except priority 0). |
| `whitelistId` | UUID | No | Restrict rule to users in a specific whitelist. Whitelist must exist on the tenant. |
| `descriptionTemplate` | string | No | Handlebars template for transaction descriptions. Supports variables like `{{buyerId}}`, `{{totalAmount}}`, `{{value}}`. |
| `poolAddress` | string (ETH address) | No | Pool wallet address for grouped transactions |
| `cashbackConfigurations` | object | No | Required for `cashback` type rules. See below. |

**CashbackConfigurations Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `indirectCashback` | number[] | Array of commission rates per level (e.g., `[0.03, 0.02, 0.01]` = 3% L1, 2% L2, 1% L3) |
| `indirectCashbackAvailable` | string | Date expression for when indirect cashback becomes available |
| `indirectCashbackDescriptionTemplate` | string | Template for indirect cashback descriptions |
| `indirectCashbackUserIdToReceiveRemainingPoints` | UUID | Fallback user to receive remaining points when referral chain is incomplete |
| `indirectCashbackSendEmail` | boolean | Send email notifications for indirect cashback |
| `finalRecipientRate` | number | Rate for final recipient (e.g., `0.01` = 1%) |
| `finalRecipientAvailable` | string | Date expression for final recipient availability |
| `finalRecipientSendEmail` | boolean | Send email for final recipient |
| `zoneAmbassadorRate` | string | Rate for zone/area ambassador rewards |
| `zoneAmbassadorAvailable` | string | Date expression for zone ambassador availability |
| `businessReferrerRate` | string | Rate for business referrer rewards |
| `businessReferrerAvailable` | string | Date expression for business referrer availability |
| `cashbackOverride` | WebhookConfig | External webhook to override cashback amount |
| `rateOverride` | WebhookConfig | External webhook to override rate |
| `overrideMultilevelCommission` | WebhookConfig | External webhook to override multilevel commissions |
| `sendToIdWebhookNotification` | boolean | Forward notification to ID service webhook |

**WebhookConfig shape:**
```json
{
  "enabled": true,
  "url": "https://your-service.com/webhook",
  "path": "data.field"
}
```

**Errors:**

| Status | Error | Cause |
|--------|-------|-------|
| 400 | AlreadyExistRuleWithSamePriorityException | Another rule with the same priority exists in this loyalty |
| 400 | AlreadyExistRuleWithSameWhitelistException | Another rule with the same whitelist and name exists |
| 404 | WhitelistNotFoundException | Provided `whitelistId` not found on the tenant |

---

## Endpoints: ERC20 Contracts

Base path: `/{companyId}/erc20-contracts`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | SuperAdmin, Admin | Create a new ERC20 contract (draft) |
| 2 | GET | `/` | Bearer | SuperAdmin, Admin | List all ERC20 contracts (paginated) |
| 3 | GET | `/:id` | Bearer | SuperAdmin, Admin | Get contract by ID |
| 4 | PATCH | `/:id` | Bearer | SuperAdmin, Admin | Update a draft contract |
| 5 | PATCH | `/:id/publish` | Bearer | SuperAdmin, Admin | Deploy contract on-chain |
| 6 | GET | `/:id/estimate-gas` | Bearer | SuperAdmin, Admin | Estimate gas cost for deployment |
| 7 | PATCH | `/:id/transfer-config` | Bearer | SuperAdmin, Admin | Update transfer configuration |
| 8 | PATCH | `/has-role` | Bearer | SuperAdmin, Admin, Integration | Check if address has a role on a contract |
| 9 | PATCH | `/grant-role` | Bearer | SuperAdmin, Admin, Integration | Grant a role to an address |

### POST /{companyId}/erc20-contracts

Create a new ERC20 contract in draft status.

**Minimal Request:**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned"
}
```

**Complete Request (production example from frontend):**
```json
{
  "name": "Loyalty Points",
  "symbol": "LPT",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "1000000",
  "initialOwner": "0x0000000000000000000000000000000000000000",
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": [
    {
      "model": "max_limit",
      "value": "1000"
    }
  ]
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Contract name. Auto-sanitized (latin characters, alphanumeric + spaces only). |
| `symbol` | string | Yes | Token symbol (e.g., `"LPT"`). Auto-sanitized (alphanumeric only). |
| `chainId` | integer | Yes | Blockchain chain ID (e.g., `137` for Polygon, `80001` for Mumbai testnet) |
| `type` | ERC20ContractType | Yes | Contract type: `classic`, `permissioned`, or `in_custody` |
| `initialAmount` | string | No | Initial token supply to mint on deployment |
| `initialOwner` | string (ETH address) | No | Address to receive initial supply |
| `isBurnable` | boolean | No | Whether tokens can be burned. Default: `false`. |
| `isWithdrawable` | boolean | No | Whether tokens can be withdrawn to external wallets. Default: `false`. |
| `transferConfig` | array | No | Transfer restriction rules. Default: `[]` (no restrictions). |

**TransferConfig item:**

| Field | Type | Description |
|-------|------|-------------|
| `model` | Erc20TransferModel | Restriction model: `max_limit`, `percentage`, `fixed`, `period`, `free` |
| `value` | string | Restriction value (interpretation depends on model) |
| `period` | string | Time period for `period` model (e.g., `"7d"`) |

**Response (201):** Full ERC20 contract entity with `status: "draft"`.

### PATCH /{companyId}/erc20-contracts/:id/publish

Deploy the contract on-chain. The contract must be in `draft` status.

**Request body:** None required.

**Response (204):** No content. Contract status changes to `publishing`, then `published` on success or `failed` on error.

**Notes:**
- This is an asynchronous operation. The contract transitions through `publishing` -> `published`.
- Use the gas estimate endpoint first to preview costs.
- Once published, the contract `address` field is populated with the on-chain address.

### GET /{companyId}/erc20-contracts/:id/estimate-gas

Estimate gas cost to deploy the contract.

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

---

## Endpoints: ERC20 Tokens (Mint / Transfer / Burn)

Base path: `/{companyId}/erc20-tokens`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | PATCH | `/:id/mint` | Bearer | SuperAdmin, Admin | Mint tokens to an address |
| 2 | PATCH | `/:id/transfer/admin` | Bearer | SuperAdmin, Admin | Admin transfer between addresses |
| 3 | PATCH | `/:id/transfer/user` | Bearer | SuperAdmin, Admin, User | User-initiated transfer |
| 4 | PATCH | `/:id/burn` | Bearer | SuperAdmin, Admin, User | Burn tokens from an address |
| 5 | GET | `/:id/history` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Transaction history for a contract |
| 6 | GET | `/:id/history/action/:actionId` | Bearer | SuperAdmin, Admin | Get specific action details |
| 7 | GET | `/:id/history/operator/:operatorId` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Transaction history by operator |
| 8 | GET | `/:id/history/:userId` | Bearer | SuperAdmin, Admin, User | Transaction history by user |

### PATCH /{companyId}/erc20-tokens/:id/mint

Mint new tokens to a wallet address. The `:id` parameter is the ERC20 contract ID.

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
  "amount": "100",
  "metadata": {
    "description": "Manual point issuance for loyalty program"
  },
  "sendEmail": false,
  "available": "1s"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string (ETH address) | Yes | Recipient wallet address |
| `amount` | string | Yes | Amount of tokens to mint |
| `metadata` | object | No | Arbitrary metadata attached to the action |
| `sendEmail` | boolean | No | Send email notification to recipient. Default: `true`. |
| `available` | string (date expression) | No | When minted tokens become available (e.g., `"1s"` = immediate) |

**Response (204):** No content.

### PATCH /{companyId}/erc20-tokens/:id/transfer/admin

Admin-initiated transfer between two wallet addresses.

**Request (production example from frontend):**
```json
{
  "from": "0xSourceAddress...",
  "to": "0xDestinationAddress...",
  "amount": "50",
  "metadata": {
    "description": "Transfer between loyalty wallets"
  },
  "sendEmail": false,
  "available": "1s"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string (ETH address) | Yes | Source wallet address |
| `to` | string (ETH address) | Yes | Destination wallet address |
| `amount` | string | Yes | Amount to transfer |
| `metadata` | object | No | Arbitrary metadata |
| `sendEmail` | boolean | No | Send email notification. Default: `true`. |
| `available` | string (date expression) | No | When transferred tokens become available |

**Response (204):** No content.

### PATCH /{companyId}/erc20-tokens/:id/burn

Burn tokens from a wallet address.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string (ETH address) | Yes | Wallet address to burn from |
| `amount` | string | Yes | Amount to burn |
| `metadata` | object | No | Arbitrary metadata |
| `sendEmail` | boolean | No | Send email notification. Default: `true`. |
| `available` | string (date expression) | No | When the burn takes effect |

**Response (204):** No content.

---

## Endpoints: Loyalty Users (Balances, Transactions, Staking)

Base path: `/{companyId}/loyalties/users`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | GET | `/balance/:userId` | Bearer | SuperAdmin, Admin, User | Get all loyalty balances for a user |
| 2 | GET | `/balance/:userId/:loyaltyId` | Bearer | SuperAdmin, Admin, User | Get balance for a specific loyalty |
| 3 | GET | `/deferred` | Bearer | SuperAdmin, Admin, User | List deferred transactions (paginated) |
| 4 | GET | `/deferred/xlsx` | Bearer | SuperAdmin, Admin, User | Export deferred transactions as Excel |
| 5 | GET | `/deferred/:userId` | Bearer | SuperAdmin, Admin, User | List deferred transactions for a user |
| 6 | GET | `/:userId/cashback-multilevel-report/:loyaltyId` | Bearer | User | Get cashback multilevel report |
| 7 | GET | `/:userId/staking/:loyaltyId` | Bearer | SuperAdmin, Admin, User | List staking details with summary |
| 8 | PATCH | `/:userId/staking/:loyaltyId/redeem` | Bearer | SuperAdmin, Admin, User | Redeem available staking rewards |

### GET /{companyId}/loyalties/users/balance/:userId

Get all loyalty program balances for a user.

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

| Field | Type | Description |
|-------|------|-------------|
| `balance` | string | Current token balance |
| `loyaltyId` | UUID | Loyalty program ID |
| `contractId` | UUID | ERC20 contract ID |
| `pointsPrecision` | PointPrecisionOptions | Integer or decimal precision |
| `image` | string | Loyalty program logo |
| `name` | string | Contract/token name |
| `symbol` | string | Token symbol |

**Notes:**
- Regular users can only view their own balance. Admins and LoyaltyOperators can view any user's balance.
- Results are cached for 10 seconds.

### GET /{companyId}/loyalties/users/deferred

List all deferred transactions with extensive filtering.

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Items per page (default: 10) |
| `loyaltyId` | UUID | Filter by loyalty program |
| `actionId` | UUID | Filter by on-chain action ID |
| `ruleId` | UUID | Filter by rule ID |
| `action` | string | Filter by action name (matches rule name) |
| `status` | LoyaltiesDeferredStatus[] | Filter by status (accepts multiple values) |
| `sortBy` | `createdAt` \| `updatedAt` \| `executeAt` \| `withdrawableAt` | Sort column |
| `orderBy` | `ASC` \| `DESC` | Sort direction |
| `startDate` | ISO 8601 datetime | Range start (used with `rangeDateBy`) |
| `endDate` | ISO 8601 datetime | Range end (used with `rangeDateBy`) |
| `rangeDateBy` | `createdAt` \| `executeAt` \| `withdrawableAt` | Which date field to filter by range |
| `createdAt_gte` | string | Created at >= (alternative to startDate/endDate) |
| `createdAt_lte` | string | Created at <= |
| `executeAt_gte` | string | Execute at >= |
| `executeAt_lte` | string | Execute at <= |
| `withdrawableAt_gte` | string | Withdrawable at >= |
| `withdrawableAt_lte` | string | Withdrawable at <= |
| `walletAddress` | ETH address | Filter by wallet (either from or to) |
| `walletAddressTo` | ETH address | Filter by destination address |
| `walletAddressFrom` | ETH address | Filter by source address |
| `withdrawBlockedOnly` | boolean | Filter only transactions with blocked withdrawals |

**Response (200):** Paginated list of `LoyaltiesDeferredEntity` items.

**Notes:**
- Regular users are automatically scoped to their own transactions (the API ignores wallet address filters for non-admin users).
- The `startDate`/`endDate` range filters take precedence over individual `_gte`/`_lte` filters.

### GET /{companyId}/loyalties/users/:userId/cashback-multilevel-report/:loyaltyId

Get aggregated cashback multilevel report for a user across all commission levels.

**Response (200):**
```json
{
  "reportPerLevel": [
    {
      "level": "1",
      "totalAmount": 500,
      "transactionsCount": 25
    },
    {
      "level": "2",
      "totalAmount": 200,
      "transactionsCount": 15
    },
    {
      "level": "businessReferral",
      "totalAmount": 100,
      "transactionsCount": 5
    }
  ]
}
```

**Notes:**
- This data comes from a materialized view and is cached for 5 minutes.
- Levels include numeric indirect cashback levels and named subtypes (`businessReferral`, `areaReferral`).

### GET /{companyId}/loyalties/users/:userId/staking/:loyaltyId

List user staking details with a summary.

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number |
| `limit` | integer | Items per page |
| `status` | LoyaltiesDeferredStatus[] | Filter by status |
| `sortBy` | JSON sort items | Sort by `createdAt`, `updatedAt`, `executeAt`, or `expiresAt` |
| `executeAt` | Date or range | Filter by execution date |
| `expiresAt` | Date or range | Filter by expiration date |

**Response (200):**
```json
{
  "items": [],
  "meta": {
    "totalItems": 0,
    "totalPages": 0,
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

### PATCH /{companyId}/loyalties/users/:userId/staking/:loyaltyId/redeem

Redeem available staking rewards. Marks pending staking deferred entries as ready for execution.

**Request body:** None required.

**Response (204):** No content.

---

## Endpoints: Loyalty Rewards (Previews & Payments)

Base path: `/{companyId}/loyalties/rewards`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | PATCH | `/cashback/preview` | Bearer | SuperAdmin, Admin, User, LoyaltyOperator | Preview cashback for a purchase amount |
| 2 | PATCH | `/payment/preview` | Bearer | SuperAdmin, Admin, User, LoyaltyOperator | Preview point payment (discount) |
| 3 | PATCH | `/payment` | Bearer | SuperAdmin, Admin, LoyaltyOperator | Execute point payment |

### PATCH /{companyId}/loyalties/rewards/cashback/preview

Preview how much cashback a user would earn for a given amount.

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "100.00",
  "userId": "user-uuid",
  "action": "cashback_multilevel",
  "destinationWalletAddress": "0x..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `amount` | string | Yes | Purchase amount |
| `userId` | UUID | Yes | User receiving cashback |
| `action` | string | Yes | Rule name to match (e.g., `"cashback_multilevel"`) |
| `destinationWalletAddress` | ETH address | No | Override destination wallet |

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

### PATCH /{companyId}/loyalties/rewards/payment/preview

Preview how a point-based payment would work (points spent as discount).

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
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `amount` | string | Yes | Total purchase amount (positive number string) |
| `points` | string | Yes | Points the user wants to spend |
| `userId` | UUID | Yes | User making the payment |
| `userCode` | string | Yes | User verification code (for in-store payments) |

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

### PATCH /{companyId}/loyalties/rewards/payment

Execute a point-based payment. Burns/transfers points from the user and records the transaction.

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "amount": "50.00",
  "points": "200",
  "userId": "user-uuid",
  "userCode": "ABC123",
  "description": "In-store purchase at location X"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `amount` | string | Yes | Total purchase amount (positive number string) |
| `points` | string | Yes | Points to spend |
| `userId` | UUID | Yes | User making the payment |
| `userCode` | string | Yes | User verification code |
| `description` | string | No | Transaction description |

**Response (200):**
```json
{
  "requestId": "request-uuid"
}
```

**Notes:**
- The `amount` field must be a positive number string (validated by `IsValidAmount` decorator).
- The `operatorUserId` is automatically set from the authenticated user making the request.
- The user is validated against `userCode` -- mismatches throw `UserNotMatchException`.

---

## Endpoints: Loyalty Webhooks (Integration)

Base path: `/{companyId}/loyalties/webhook`

These endpoints are designed for service-to-service integration. Most require the `Integration` tenant role.

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/deposit` | Bearer | SuperAdmin, Admin | Trigger a deposit (reward distribution) |
| 2 | POST | `/cashback-multilevel` | Bearer | Integration | Trigger multilevel cashback distribution |
| 3 | POST | `/rollback` | Bearer | Integration | Rollback deferred transactions |
| 4 | GET | `/rollback/:requestId` | Bearer | Integration | Check rollback status |
| 5 | POST | `/split-payees` | Bearer | Integration | Distribute rewards to multiple payees |

### POST /{companyId}/loyalties/webhook/deposit

Trigger a reward deposit for a user based on loyalty rules.

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "user-uuid",
  "action": "cashback_multilevel",
  "amount": "100.00",
  "description": "Purchase cashback",
  "metadata": {
    "orderId": "order-uuid",
    "totalAmount": "100.00"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `userId` | UUID | Yes | User receiving the deposit |
| `action` | string | Yes | Rule name to match |
| `amount` | string | No | Base amount for calculation |
| `description` | string | No | Transaction description |
| `metadata` | object | No | Arbitrary metadata (passed to description templates via Handlebars) |

**Response (200):**
```json
{
  "requestId": "request-uuid"
}
```

### POST /{companyId}/loyalties/webhook/cashback-multilevel

Trigger multilevel cashback distribution. Processes direct cashback plus indirect commissions across referral levels.

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "userId": "user-uuid",
  "amount": "100.00",
  "description": "Multilevel cashback from order",
  "operatorUserId": "operator-uuid",
  "metadata": {
    "orderId": "order-uuid"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `userId` | UUID | Yes | User who triggered the cashback (buyer) |
| `amount` | string | Yes | Base amount for cashback calculation |
| `description` | string | No | Transaction description |
| `operatorUserId` | UUID | No | Operator who initiated the action |
| `metadata` | object | No | Arbitrary metadata |

### POST /{companyId}/loyalties/webhook/rollback

Rollback deferred transactions matching the specified criteria.

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "requestId": "original-request-uuid",
  "metadata": {
    "reason": "Order cancelled"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `requestId` | string | No | Original request ID to rollback |
| `metadata` | object | No | Rollback metadata |

### GET /{companyId}/loyalties/webhook/rollback/:requestId

Check if a rollback has completed.

**Response (200):**
```json
{
  "completed": true
}
```

### POST /{companyId}/loyalties/webhook/split-payees

Distribute rewards across multiple payees with percentage-based splits.

**Request:**
```json
{
  "loyaltyId": "loyalty-uuid",
  "total": "1000",
  "payees": [
    {
      "walletAddress": "0xAddress1...",
      "percent": "0.60",
      "metadata": { "role": "primary" },
      "descriptionTemplate": "Primary share: {{amount}} points",
      "available": "1s",
      "sendEmail": true
    },
    {
      "walletAddress": "0xAddress2...",
      "percent": "0.40",
      "metadata": { "role": "secondary" },
      "descriptionTemplate": "Secondary share: {{amount}} points",
      "available": "7d",
      "sendEmail": false
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loyaltyId` | UUID | Yes | Loyalty program ID |
| `total` | string | Yes | Total amount to split |
| `payees` | array | Yes | List of payees with their share |
| `payees[].walletAddress` | ETH address | Yes | Recipient wallet |
| `payees[].percent` | string | Yes | Share percentage (e.g., `"0.60"` = 60%) |
| `payees[].metadata` | object | Yes | Payee metadata |
| `payees[].descriptionTemplate` | string | No | Handlebars template for description |
| `payees[].available` | string | No | Date expression for availability |
| `payees[].sendEmail` | boolean | No | Send email notification |

**Response (200):**
```json
{
  "requestId": "request-uuid"
}
```

---

## Error Reference

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 404 | LoyaltyNotFoundException | Loyalty ID not found in company | Verify loyalty ID and company ID |
| 400 | AlreadyExistRuleWithSamePriorityException | Duplicate rule priority | Use a different priority value |
| 400 | AlreadyExistRuleWithSameWhitelistException | Duplicate whitelist + name combination | Use a different whitelist or name |
| 404 | LoyaltyRuleNotFound | Rule ID not found | Verify rule ID, loyalty ID, and company ID |
| 404 | WhitelistNotFoundException | Whitelist not found on tenant | Create the whitelist first or verify the ID |
| 404 | ThereIsNoRuleForThisActionException | No rule matches the action name | Verify the action name matches a rule's `name` field |
| 404 | UserByCodeNotFoundException | User code not found | Verify user code in the company |
| 400 | UserNotMatchException | User ID doesn't match the code | Verify userId and userCode belong to the same user |
| 400 | LoyaltiesNotHaveContractException | Loyalty has no ERC20 contract | Assign an ERC20 contract to the loyalty program |
| 400 | InsufficientBalanceException | User doesn't have enough points | Check balance before attempting payment |
| 400 | DuplicateRequestException | Duplicate request detected | Avoid sending the same request twice |
