---
id: PASS_SKILL_INDEX
title: "Pass & Benefits Skill Index"
module: offpix/pass
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Pass & Benefits Skill Index

Documentation for the W3Block Pass & Benefits module. Covers NFT-based token passes, digital and physical benefits, operator management, QR-based check-in, usage tracking with temporal rules, share codes, and metadata-based eligibility requirements.

**Service:** Pass | **Swagger:** https://pass.w3block.io/docs

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships, error codes | Implemented | API reference at any time |
| 2 | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) | 1.0.0 | Create pass, add benefits (digital/physical), check-in config, operators, share codes | Implemented | Build pass management features |
| 3 | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) | 1.0.0 | View benefits, QR generation, 4 redemption methods, usage tracking, verification | Implemented | Build benefit redemption flows |

---

## Quick Guide

### For admin (pass management):

```
1. Read: FLOW_PASS_LIFECYCLE.md            → Create passes and configure benefits
2. Consult: PASS_API_REFERENCE.md          → For check-in schemas, usage rules, requirements
```

### For user/operator (redemption):

```
1. Read: FLOW_PASS_BENEFIT_REDEMPTION.md   → All redemption flows and validation chain
2. Consult: PASS_API_REFERENCE.md          → For error codes and edge cases
```

### Minimal API-first implementation:

```
1. POST /token-passes/tenants/{id}                              → Create pass
2. POST /token-pass-benefits/tenants/{id}                       → Add benefit
3. POST /token-pass-benefit-operators/tenants/{id}              → Assign operator
4. GET  /token-pass-benefits/tenants/{id}/{bid}/{ed}/secret     → Generate QR
5. POST /token-pass-benefits/tenants/{id}/register-use-by-qrcode → Scan & register
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Collection must be published | Pass creation requires a published collection with contract address |
| 2 | Dynamic QR expires in 45s | Regenerate before scanning — don't cache |
| 3 | `allowSelfUse` defaults to `false` | Enable for digital benefits, keep off for physical (operator required) |
| 4 | Usage doesn't reset | Configure `usageRule` (day/week/month/timestamp) for periodic resets |
| 5 | Check-in timezone mismatch | Set `timezoneOrUtcOffset` to match the physical location |
| 6 | Requirements not matching | Token metadata comparison is case-insensitive for string equality |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Create a new token pass | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Add digital benefits (links) | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Add physical benefits (locations) | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Configure check-in schedules | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) + [API Reference (check-in schema)](./PASS_API_REFERENCE.md) |
| Set up usage limits with resets | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) + [API Reference (usage rules)](./PASS_API_REFERENCE.md) |
| Assign operators for check-in | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Create public share codes | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Generate QR codes for users | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Register benefit use (operator) | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| User self-redemption | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| View/export usage reports | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Understand validation chain | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Token metadata requirements | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Lifecycle | Redemption |
|----------|:-------:|:---------:|:----------:|
| POST /token-passes | X | X | |
| GET /token-passes | X | X | |
| GET /token-passes/:id | X | X | |
| GET /token-passes/:id/token-editions/:ed/benefits | X | | X |
| GET /token-passes/users/:userId | X | X | |
| PATCH /token-passes/:id | X | X | |
| DELETE /token-passes/:id | X | X | |
| POST /token-pass-benefits | X | X | |
| GET /token-pass-benefits | X | X | |
| GET /token-pass-benefits/:id | X | | X |
| GET /token-pass-benefits/:id/:ed/secret | X | | X |
| GET /token-pass-benefits/:id/verify | X | | X |
| PATCH /token-pass-benefits/:id | X | X | |
| DELETE /token-pass-benefits/:id | X | X | |
| POST /token-pass-benefits/:id/register-use | X | | X |
| POST /token-pass-benefits/:id/register-use-by-user | X | | X |
| POST /token-pass-benefits/register-use-by-qrcode | X | | X |
| POST /token-pass-benefits/:id/use | X | | X |
| GET /token-pass-benefits/usages | X | | X |
| GET /token-pass-benefits/usages/xls | X | | X |
| POST /token-pass-benefit-addresses | X | X | |
| GET /token-pass-benefit-addresses | X | X | |
| POST /token-pass-benefit-operators | X | X | |
| GET /token-pass-benefit-operators | X | X | |
| GET /token-pass-benefit-operators/benefits/:id | X | X | |
| DELETE /token-pass-benefit-operators/:id | X | X | |
| POST /token-pass-share-codes | X | X | |
| GET /token-pass-share-codes/:code | X | | X |
