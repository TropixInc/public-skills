---
id: OFFPIX_SKILL_INDEX
title: "Offpix Skill Index"
module: offpix
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Offpix Skill Index

Master index for all Offpix backend API documentation. These documents are aimed at developers and AI-assisted users building custom applications on the W3Block platform.

**Before writing any new documentation, read:** [OFFPIX_DOCUMENTATION_SPEC.md](./OFFPIX_DOCUMENTATION_SPEC.md)

---

## Modules

| # | Module | Version | Documents | Status | Description |
|---|--------|---------|-----------|--------|-------------|
| 0 | [Documentation Spec](./OFFPIX_DOCUMENTATION_SPEC.md) | 1.0.0 | 1 | Setup | Process spec for generating documentation |
| 1 | [Auth](./auth/AUTH_SKILL_INDEX.md) | 1.0.0 | 6 | Implemented | Sign-in (5 methods), sign-up, password reset, OAuth, token management |
| 2 | [Commerce](./commerce/COMMERCE_SKILL_INDEX.md) | 1.0.0 | 6 | Implemented | Products, orders, payments, promotions, splits, gateways, tags |
| 3 | [Configurations](./configurations/CONFIGURATIONS_SKILL_INDEX.md) | 1.0.0 | 5 | Implemented | Contexts, tenant contexts, form fields, signup activation, KYC config |
| 4 | [Contacts](./contacts/CONTACTS_SKILL_INDEX.md) | 1.0.0 | 5 | Implemented | User management, KYC submission & approval, royalty, external contacts |
| 5 | [Contracts & Tokens](./contracts/CONTRACTS_SKILL_INDEX.md) | 1.0.0 | 6 | Implemented | NFT contracts, collections, editions, ERC20, royalty, gas, RFID, XLSX |
| 6 | [Loyalty](./loyalty/LOYALTY_SKILL_INDEX.md) | 1.0.0 | 4 | Implemented | ERC20 contracts, loyalty programs, rewards, cashback, payments, staking |
| 7 | [Whitelist](./whitelist/WHITELIST_SKILL_INDEX.md) | 1.0.0 | 3 | Implemented | Whitelists (user groups), entries, access check, on-chain promotion |
| 8 | [Pass & Benefits](./pass/PASS_SKILL_INDEX.md) | 1.0.0 | 3 | Implemented | Token passes, digital/physical benefits, QR check-in, operators, share codes |
| 9 | [Settings & Billing](./settings/SETTINGS_SKILL_INDEX.md) | 1.0.0 | 4 | Implemented | Plans, credit card, billing cycles, usage, feature-gating, tenant config |

---

## Auth Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [AUTH_SKILL_INDEX](./auth/AUTH_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [AUTH_API_REFERENCE](./auth/AUTH_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, schemas |
| 3 | [FLOW_AUTH_SIGNIN](./auth/FLOW_AUTH_SIGNIN.md) | Flow | Password, OTP, magic token, tenant API key sign-in |
| 4 | [FLOW_AUTH_SIGNUP](./auth/FLOW_AUTH_SIGNUP.md) | Flow | User registration + tenant onboarding |
| 5 | [FLOW_AUTH_PASSWORD_RESET](./auth/FLOW_AUTH_PASSWORD_RESET.md) | Flow | Request reset + set new password |
| 6 | [FLOW_AUTH_OAUTH](./auth/FLOW_AUTH_OAUTH.md) | Flow | Google & Apple OAuth (redirect + direct) |

## Commerce Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [COMMERCE_SKILL_INDEX](./commerce/COMMERCE_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [COMMERCE_API_REFERENCE](./commerce/COMMERCE_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, schemas |
| 3 | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./commerce/FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) | Flow | Create, update, publish, cancel products; variants, order rules, tags |
| 4 | [FLOW_COMMERCE_ORDER_PURCHASE](./commerce/FLOW_COMMERCE_ORDER_PURCHASE.md) | Flow | Order preview, creation, multi-payment, delivery, refunds |
| 5 | [FLOW_COMMERCE_PROMOTIONS](./commerce/FLOW_COMMERCE_PROMOTIONS.md) | Flow | Coupons, discounts, product/user scoping, requirements |
| 6 | [FLOW_COMMERCE_SPLITS_GATEWAYS](./commerce/FLOW_COMMERCE_SPLITS_GATEWAYS.md) | Flow | Revenue splits, Stripe/ASAAS config, provider selections |

---

## Available Domains (Not Yet Documented)

| Domain | Backend Service | Swagger | Priority |
|--------|----------------|---------|----------|
| ~~Commerce~~ | commerce-backend | https://commerce.w3block.io/docs | **Documented** |
| ~~Configuration~~ | pixwayid-backend | https://pixwayid.w3block.io/docs/ | **Documented** |
| ~~Tokens & NFTs~~ | pixway-registry-backend | https://api.w3block.io/docs/ | **Documented** (in Contracts module) |
| ~~Tokenization~~ | pixway-registry-backend | https://api.w3block.io/docs/ | **Documented** (in Contracts module) |
| ~~Pass & Benefits~~ | pass-backend | https://pass.w3block.io/docs | **Documented** |
| ~~Loyalty~~ | pixway-registry-backend | https://api.w3block.io/docs/ | **Documented** |
| ~~Contacts~~ | pixwayid-backend | https://pixwayid.w3block.io/docs/ | **Documented** |
| ~~KYC~~ | pixwayid-backend | https://pixwayid.w3block.io/docs/ | **Documented** (in Contacts module) |
| ~~Settings & Billing~~ | pixwayid-backend | https://pixwayid.w3block.io/docs/ | **Documented** |
| ~~Whitelist~~ | pixwayid-backend | https://pixwayid.w3block.io/docs/ | **Documented** |
| PDF | w3block-pdf | https://pdf.w3block.io/docs | Low (internal only) |

---

## Configurations Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [CONFIGURATIONS_SKILL_INDEX](./configurations/CONFIGURATIONS_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [CONFIGURATIONS_API_REFERENCE](./configurations/CONFIGURATIONS_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, entity relationships |
| 3 | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) | Flow | Create, update, list forms (contexts + tenant contexts) |
| 4 | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) | Flow | Add, update, retrieve form fields (19 input types) |
| 5 | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./configurations/FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) | Flow | Toggle signup registration, configure signup form |

## Contacts Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [CONTACTS_SKILL_INDEX](./contacts/CONTACTS_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [CONTACTS_API_REFERENCE](./contacts/CONTACTS_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, entity relationships |
| 3 | [FLOW_CONTACTS_USER_MANAGEMENT](./contacts/FLOW_CONTACTS_USER_MANAGEMENT.md) | Flow | List, invite, edit users, roles, royalty, external contacts |
| 4 | [FLOW_CONTACTS_KYC_SUBMISSION](./contacts/FLOW_CONTACTS_KYC_SUBMISSION.md) | Flow | Fetch form fields, submit documents, multi-step, resubmission |
| 5 | [FLOW_CONTACTS_KYC_APPROVAL](./contacts/FLOW_CONTACTS_KYC_APPROVAL.md) | Flow | List, review, approve/reject/request-review KYC submissions |

## Contracts & Tokens Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [CONTRACTS_SKILL_INDEX](./contracts/CONTRACTS_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [CONTRACTS_API_REFERENCE](./contracts/CONTRACTS_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, entity relationships |
| 3 | [FLOW_CONTRACTS_NFT_LIFECYCLE](./contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) | Flow | Create, configure, deploy NFT contracts with royalty |
| 4 | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) | Flow | Collections, editions, templates, publish, bulk XLSX |
| 5 | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./contracts/FLOW_CONTRACTS_TOKEN_OPERATIONS.md) | Flow | Mint, transfer, burn tokens, RFID, metadata, gas, certificates |
| 6 | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./contracts/FLOW_CONTRACTS_ERC20_LIFECYCLE.md) | Flow | ERC20 create, deploy, mint, transfer rules (5 handlers), burn |

## Loyalty Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [LOYALTY_SKILL_INDEX](./loyalty/LOYALTY_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [LOYALTY_API_REFERENCE](./loyalty/LOYALTY_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, entity relationships |
| 3 | [FLOW_LOYALTY_CONTRACT_SETUP](./loyalty/FLOW_LOYALTY_CONTRACT_SETUP.md) | Flow | Create ERC20 contract, deploy, create loyalty program, configure rules |
| 4 | [FLOW_LOYALTY_BALANCE_OPERATIONS](./loyalty/FLOW_LOYALTY_BALANCE_OPERATIONS.md) | Flow | Mint, transfer, burn tokens; check balances and history |
| 5 | [FLOW_LOYALTY_REWARDS_PAYMENTS](./loyalty/FLOW_LOYALTY_REWARDS_PAYMENTS.md) | Flow | Deposits, cashback, multilevel commissions, payments, staking, rollbacks |

## Pass & Benefits Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [PASS_SKILL_INDEX](./pass/PASS_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [PASS_API_REFERENCE](./pass/PASS_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, error codes, check-in schemas |
| 3 | [FLOW_PASS_LIFECYCLE](./pass/FLOW_PASS_LIFECYCLE.md) | Flow | Create pass, add benefits, check-in config, operators, share codes |
| 4 | [FLOW_PASS_BENEFIT_REDEMPTION](./pass/FLOW_PASS_BENEFIT_REDEMPTION.md) | Flow | QR generation, 4 redemption methods, validation chain, usage tracking |

## Settings & Billing Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [SETTINGS_SKILL_INDEX](./settings/SETTINGS_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [SETTINGS_API_REFERENCE](./settings/SETTINGS_API_REFERENCE.md) | API Reference | All 16 endpoints, DTOs, enums, entity relationships |
| 3 | [FLOW_SETTINGS_BILLING_MANAGEMENT](./settings/FLOW_SETTINGS_BILLING_MANAGEMENT.md) | Flow | Plan selection, credit card, billing cycles, usage, simulation, cancellation |
| 4 | [FLOW_SETTINGS_TENANT_CONFIGURATION](./settings/FLOW_SETTINGS_TENANT_CONFIGURATION.md) | Flow | Tenant profile, auth providers, KYC, passwordless, email, feature checks |

## Whitelist Module Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [WHITELIST_SKILL_INDEX](./whitelist/WHITELIST_SKILL_INDEX.md) | Index | Entry point, decision table, endpoint matrix |
| 2 | [WHITELIST_API_REFERENCE](./whitelist/WHITELIST_API_REFERENCE.md) | API Reference | All endpoints, DTOs, enums, entity relationships |
| 3 | [FLOW_WHITELIST_MANAGEMENT](./whitelist/FLOW_WHITELIST_MANAGEMENT.md) | Flow | Create, edit, delete whitelists; add/remove entries; check access; on-chain |

---

## Quick Start

```
1. Authenticate    → Read auth/AUTH_SKILL_INDEX.md
2. Pick your flow  → Check the "Available Domains" table above
3. Implement       → Follow the module's SKILL_INDEX → FLOW docs → API_REFERENCE
```
