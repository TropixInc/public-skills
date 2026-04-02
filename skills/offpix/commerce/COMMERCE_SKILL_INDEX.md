---
id: COMMERCE_SKILL_INDEX
title: "Commerce Module - Skill Index"
module: offpix
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - index
depends_on:
  - AUTH_SKILL_INDEX
---

# Commerce Module - Skill Index

The Commerce module powers the W3Block e-commerce engine: products (NFTs, ERC20, external), multi-currency orders, multi-provider payments, promotions/coupons, revenue splits, tags, refunds, and sales reporting.

**Backend Service:** Commerce Service
**Swagger:** https://commerce.w3block.io/docs
**Base URL:** `https://commerce.w3block.io`

---

## Documents

| # | Document | Type | Version | Status | Description |
|---|----------|------|---------|--------|-------------|
| 1 | [COMMERCE_SKILL_INDEX](./COMMERCE_SKILL_INDEX.md) | Index | 1.0.0 | Implemented | This file — entry point, decision table, endpoint matrix |
| 2 | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) | API Ref | 1.0.0 | Implemented | All endpoints, DTOs, enums, schemas |
| 3 | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) | Flow | 1.0.0 | Implemented | Create, update, publish, cancel products |
| 4 | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) | Flow | 1.0.0 | Implemented | Order preview, creation, payment, delivery |
| 5 | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) | Flow | 1.0.0 | Implemented | Coupons, discounts, whitelists, product scope |
| 6 | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) | Flow | 1.0.0 | Implemented | Revenue splits, payment gateway config |

---

## Quick Start

```
1. Authenticate         → See auth/AUTH_SKILL_INDEX.md
2. Create a product     → FLOW_COMMERCE_PRODUCT_LIFECYCLE.md
3. Configure payments   → FLOW_COMMERCE_SPLITS_GATEWAYS.md
4. Accept orders        → FLOW_COMMERCE_ORDER_PURCHASE.md
5. Manage promotions    → FLOW_COMMERCE_PROMOTIONS.md
6. Review sales         → COMMERCE_API_REFERENCE.md § Orders (Admin)
```

---

## Decision Table

| I want to… | Read this |
|------------|-----------|
| List/create/publish products | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Add variants or order rules to a product | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Preview an order (compute prices, fees, discounts) | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Create an order and process payment | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Cancel/refund an order | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Create coupons or automatic discounts | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Restrict promotions to specific users/products | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Configure Stripe/ASAAS gateways | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
| Set up revenue distribution rules | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
| Manage product categories (tags) | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) § Tags |
| Export sales reports | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) § Exports |
| Look up any endpoint | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) |

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Order creation fails with "insufficient allowance" | Run order preview first — check `currencyAllowanceState` for each payment currency |
| 2 | Product stuck in PUBLISHING status | Publishing is async. Poll `GET /admin/companies/{companyId}/products/{productId}` until status changes to PUBLISHED |
| 3 | Coupon code rejected | Codes are UPPERCASE and unique per company. Check `maxUsages`, `maxUsagesPerUser`, date range, and product/whitelist scope |
| 4 | Split percentages don't add up | Total split % for a given `type` must not exceed 100. Each type (PRODUCT_PRICE, CLIENT_SERVICE_FEE, etc.) is independent |
| 5 | Payment expires before completion | Default expiration is configurable per gateway. Set `checkoutExpireTime` in gateway config |

---

## Matrix: Endpoints × Documents

| Endpoint Pattern | API Ref | Product | Order | Promos | Splits |
|-----------------|---------|---------|-------|--------|--------|
| `/admin/.../products` | X | X | | | |
| `/admin/.../products/{id}/publish` | X | X | | | |
| `/admin/.../products/{id}/variants` | X | X | | | |
| `/admin/.../products/{id}/order-rules` | X | X | | | |
| `/companies/{id}/orders` (user) | X | | X | | |
| `/companies/{id}/orders/preview` | X | | X | | |
| `/admin/.../orders` | X | | X | | |
| `/admin/.../orders/{id}/approve-payment` | X | | X | | |
| `/admin/.../orders/{id}/cancel` | X | | X | | |
| `/admin/.../orders/{id}/refund` | X | | X | | |
| `/admin/.../promotions` | X | | | X | |
| `/admin/.../promotions/{id}/products` | X | | | X | |
| `/admin/.../promotions/{id}/whitelists` | X | | | X | |
| `/admin/.../split-configurations` | X | | | | X |
| `/admin/.../configurations/providers/*` | X | | | | X |
| `/admin/.../tags` | X | | | | |
| `/globals/currencies` | X | | X | | X |
| `/admin/.../exports/*` | X | | | | |
