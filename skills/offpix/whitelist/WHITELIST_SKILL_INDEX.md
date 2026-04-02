---
id: WHITELIST_SKILL_INDEX
title: "Whitelist Skill Index"
module: offpix/whitelist
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Whitelist Skill Index

Documentation for the W3Block Whitelist module. Covers creation and management of whitelists (user groups) and their entries, used to control access to products, promotions, and blockchain operations. Entry types include email, wallet address, user ID, collection holder, KYC-approved context, KEY collection holder, and KEY ERC-20 holder.

**Service:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

> **Frontend naming:** The Offpix frontend labels whitelists as **"User Groups"** in the navigation and breadcrumbs.

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [WHITELIST_API_REFERENCE](./WHITELIST_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) | 1.0.0 | Create, edit, delete whitelists; add/remove entries; check user access; promote on-chain | Implemented | Build whitelist (user group) management features |

---

## Quick Guide

### For a basic whitelist implementation:

```
1. Read: FLOW_WHITELIST_MANAGEMENT.md      -> Create a whitelist, add entries, check access
2. Consult: WHITELIST_API_REFERENCE.md     -> For enums, DTOs, pagination, edge cases
```

### Minimal API-first implementation (3 calls):

```
1. POST /whitelists/{tenantId}                    -> Create a whitelist
2. POST /whitelists/{tenantId}/{id}/entries        -> Add entries (repeat per entry)
3. GET  /whitelists/{tenantId}/{id}/check-user     -> Verify user access
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Duplicate entry error on add | The combination of `whitelistId` + `type` + `value` is unique (except for `collection_holder` and `key_collection_holder`). Check before adding |
| 2 | `value` is lowercased silently | All entry values are lowercased on save via a database transformer. Store and compare in lowercase |
| 3 | User ID entry fails with "User not found" | The user must belong to the same tenant. Cross-tenant user IDs are rejected |
| 4 | `additionalData` required for holder types | `collection_holder` requires `chainId`; `key_collection_holder` and `key_erc20_holder` accept `amount`. Omitting `additionalData` for these types causes validation errors |
| 5 | Promote on-chain fails with "already on chain" | A whitelist can only have one wallet group per `chainId`. Check existing wallet groups before promoting |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Create a new whitelist (user group) | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Add entries (emails, wallets, users) to a whitelist | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Check if a user has access via whitelist | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Promote a whitelist on-chain | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Understand entry types and additionalData schemas | [WHITELIST_API_REFERENCE](./WHITELIST_API_REFERENCE.md) |
| Filter and paginate whitelist entries | [WHITELIST_API_REFERENCE](./WHITELIST_API_REFERENCE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Whitelist Management |
|----------|:-------:|:--------------------:|
| POST /whitelists/{tenantId} | X | X |
| GET /whitelists/{tenantId} | X | X |
| GET /whitelists/{tenantId}/{id} | X | X |
| PATCH /whitelists/{tenantId}/{id} | X | X |
| DELETE /whitelists/{tenantId}/{id} | X | X |
| POST /whitelists/{tenantId}/{id}/entries | X | X |
| GET /whitelists/{tenantId}/{id}/entries | X | X |
| DELETE /whitelists/{tenantId}/{id}/entries/{entryId} | X | X |
| GET /whitelists/{tenantId}/{id}/check-user | X | X |
| GET /whitelists/{tenantId}/check-user | X | X |
| PATCH /whitelists/{tenantId}/{id}/promote-on-chain | X | X |
