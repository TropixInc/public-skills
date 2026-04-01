---
id: LOYALTY_SKILL_INDEX
title: "Loyalty Skill Index"
module: offpix/loyalty
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Loyalty Skill Index

Documentation for the W3Block Loyalty module. Covers ERC20 token contracts, loyalty programs, reward rules (add, multiply, split, cashback with multilevel commissions), point-based payments, staking, and webhook-driven reward distribution.

**Service:** KEY (pixway-registry) | **Swagger:** https://api.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [LOYALTY_API_REFERENCE](./LOYALTY_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) | 1.0.0 | Create ERC20 contract, deploy on-chain, create loyalty program, configure rules | Implemented | Initial loyalty system setup |
| 3 | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) | 1.0.0 | Mint, transfer, burn tokens; check balances and transaction history | Implemented | Manual point management |
| 4 | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) | 1.0.0 | Deposits, cashback, multilevel commissions, payments, staking, rollbacks | Implemented | Automated rewards and point spending |

---

## Quick Guide

### For a basic loyalty implementation:

```
1. Read: FLOW_LOYALTY_CONTRACT_SETUP.md       -> Create ERC20 contract and loyalty program
2. Read: FLOW_LOYALTY_BALANCE_OPERATIONS.md   -> Mint tokens to users, check balances
3. Read: FLOW_LOYALTY_REWARDS_PAYMENTS.md     -> Set up automated rewards and payments
4. Consult: LOYALTY_API_REFERENCE.md          -> For enums, DTOs, edge cases
```

### Minimal API-first implementation (5 calls):

```
1. POST /{companyId}/erc20-contracts             -> Create token contract (draft)
2. PATCH /{companyId}/erc20-contracts/{id}/publish -> Deploy on-chain
3. POST /{companyId}/loyalties/admin              -> Create loyalty program
4. POST /{companyId}/loyalties/admin/{id}/rules   -> Add reward rule
5. PATCH /{companyId}/erc20-tokens/{id}/mint      -> Mint points to a user
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Contract still in `draft` when trying to use | Must publish contract first. Poll status until `published`. |
| 2 | Loyalty program without `contractId` | On-chain operations fail without a linked ERC20 contract. Always assign one. |
| 3 | Rule priority collision on creation | Each rule needs a unique priority within the loyalty (except priority 0). |
| 4 | Deposit webhook returns 404 | The `action` field must match a rule's `name` exactly (slugified, lowercase). |
| 5 | Points don't appear after mint/deposit | Balances are cached 5-10 seconds. Points may also have a delayed `available` date. |
| 6 | Payment fails with InsufficientBalanceException | Always preview the payment first and check user balance. |
| 7 | `userCode` mismatch on payment | The `userId` and `userCode` must belong to the same user. |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Create a new loyalty token and program | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) |
| Configure cashback rules with multilevel commissions | [FLOW_LOYALTY_CONTRACT_SETUP](./FLOW_LOYALTY_CONTRACT_SETUP.md) (Step 5) |
| Mint points to a user | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) (Step 2) |
| Transfer points between wallets | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) (Steps 3-4) |
| Check a user's point balance | [FLOW_LOYALTY_BALANCE_OPERATIONS](./FLOW_LOYALTY_BALANCE_OPERATIONS.md) (Step 6) |
| Trigger automated rewards via webhook | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Steps 1-2) |
| Preview and execute point-based payments | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Steps 4-5) |
| Split rewards among multiple payees | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Step 6) |
| Rollback reward transactions | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Steps 7-8) |
| View staking details and redeem | [FLOW_LOYALTY_REWARDS_PAYMENTS](./FLOW_LOYALTY_REWARDS_PAYMENTS.md) (Steps 9-10) |
| Look up all enums and error codes | [LOYALTY_API_REFERENCE](./LOYALTY_API_REFERENCE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Contract Setup | Balance Ops | Rewards & Payments |
|----------|:-------:|:--------------:|:-----------:|:------------------:|
| POST /{companyId}/erc20-contracts | X | X | | |
| GET /{companyId}/erc20-contracts | X | | | |
| GET /{companyId}/erc20-contracts/:id | X | X | | |
| PATCH /{companyId}/erc20-contracts/:id | X | X | | |
| PATCH /{companyId}/erc20-contracts/:id/publish | X | X | | |
| GET /{companyId}/erc20-contracts/:id/estimate-gas | X | X | | |
| PATCH /{companyId}/erc20-contracts/:id/transfer-config | X | | | |
| PATCH /{companyId}/erc20-contracts/has-role | X | | | |
| PATCH /{companyId}/erc20-contracts/grant-role | X | | | |
| POST /{companyId}/loyalties/admin | X | X | | |
| GET /{companyId}/loyalties/admin | X | X | | |
| GET /{companyId}/loyalties/admin/:loyaltyId | X | X | | |
| PATCH /{companyId}/loyalties/admin/:loyaltyId | X | X | | |
| POST /{companyId}/loyalties/admin/:loyaltyId/rules | X | X | | |
| GET /{companyId}/loyalties/admin/:loyaltyId/rules/:ruleId | X | X | | |
| PATCH /{companyId}/loyalties/admin/:loyaltyId/rules/:ruleId | X | X | | |
| DELETE /{companyId}/loyalties/admin/:loyaltyId/rules/:ruleId | X | X | | |
| PATCH /{companyId}/erc20-tokens/:id/mint | X | | X | |
| PATCH /{companyId}/erc20-tokens/:id/transfer/admin | X | | X | |
| PATCH /{companyId}/erc20-tokens/:id/transfer/user | X | | X | |
| PATCH /{companyId}/erc20-tokens/:id/burn | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history/action/:actionId | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history/operator/:operatorId | X | | X | |
| GET /{companyId}/erc20-tokens/:id/history/:userId | X | | X | |
| GET /{companyId}/loyalties/users/balance/:userId | X | | X | |
| GET /{companyId}/loyalties/users/balance/:userId/:loyaltyId | X | | X | |
| GET /{companyId}/loyalties/users/deferred | X | | X | X |
| GET /{companyId}/loyalties/users/deferred/xlsx | X | | X | |
| GET /{companyId}/loyalties/users/deferred/:userId | X | | X | |
| PATCH /{companyId}/loyalties/rewards/cashback/preview | X | | | X |
| PATCH /{companyId}/loyalties/rewards/payment/preview | X | | | X |
| PATCH /{companyId}/loyalties/rewards/payment | X | | | X |
| POST /{companyId}/loyalties/webhook/deposit | X | | | X |
| POST /{companyId}/loyalties/webhook/cashback-multilevel | X | | | X |
| POST /{companyId}/loyalties/webhook/rollback | X | | | X |
| GET /{companyId}/loyalties/webhook/rollback/:requestId | X | | | X |
| POST /{companyId}/loyalties/webhook/split-payees | X | | | X |
| GET /{companyId}/loyalties/users/:userId/cashback-multilevel-report/:loyaltyId | X | | | X |
| GET /{companyId}/loyalties/users/:userId/staking/:loyaltyId | X | | | X |
| PATCH /{companyId}/loyalties/users/:userId/staking/:loyaltyId/redeem | X | | | X |
