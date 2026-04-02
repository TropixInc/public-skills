---
id: TOKENIZATION_SKILL_INDEX
title: "Tokenization Skill Index"
module: offpix/tokenization
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# Tokenization Skill Index

Documentation for the W3Block Tokenization module. Covers the full token collection lifecycle (DRAFT to PUBLISHED) and edition operations including minting, transferring, burning, metadata updates, RFID assignment, and bulk XLSX import/export. This module handles everything **after** contract deployment -- the Contracts module covers deployment itself.

**Service:** KEY | **Swagger:** https://api.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [TOKENIZATION_API_REFERENCE](./TOKENIZATION_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) | 1.0.0 | Create draft, sync editions, configure, publish to blockchain | Implemented | Build collection creation and publishing flows |
| 3 | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) | 1.0.0 | Mint, transfer, burn, metadata update, RFID, gas estimation | Implemented | Manage individual token editions post-publish |
| 4 | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) | 1.0.0 | XLSX template export, bulk edition import, async job tracking | Implemented | Bulk manage editions via Excel spreadsheets |

---

## Quick Guide

### Create and publish a token collection:

```
1. Read: FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md  -> Create draft, sync editions, configure, publish
2. Consult: TOKENIZATION_API_REFERENCE.md           -> For enums, DTOs, validation rules
```

### Manage editions (mint/transfer/burn):

```
1. Read: FLOW_TOKENIZATION_EDITION_OPERATIONS.md    -> Mint, transfer, burn, RFID, metadata
2. Consult: TOKENIZATION_API_REFERENCE.md           -> For status enums, error codes
```

### Bulk import via Excel:

```
1. Read: FLOW_TOKENIZATION_BULK_IMPORT.md           -> Export template, import, handle errors
2. Consult: TOKENIZATION_API_REFERENCE.md           -> For import/export endpoint details
```

### Consult API details:

```
1. Read: TOKENIZATION_API_REFERENCE.md              -> Full endpoint reference, DTOs, enums
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Publishing without syncing editions first | Must call `sync-draft-tokens` and wait for the async job to complete before calling publish |
| 2 | Contract not published | The contract must be in PUBLISHED status with the ADMIN_MINTER feature enabled before collection publish |
| 3 | Edition status not DRAFT at publish | All editions must be in DRAFT status (no error states like DRAFT_ERROR or IMPORT_ERROR) |
| 4 | RFID uniqueness violation | RFIDs are globally unique across all active editions (excluding deleted/burned). Check availability with `check-rfid` before assigning |
| 5 | Invalid rangeInitialToMint format | Must be a valid range like `"1-50"` or comma-separated like `"1,5,10"` |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Create a token collection | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Sync/generate editions | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Publish a collection to blockchain | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Mint tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Transfer tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Burn tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Update token metadata | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Assign RFID to a token | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Estimate gas costs | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Import editions from Excel | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |
| Export edition template | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Collection Lifecycle | Edition Operations | Bulk Import |
|----------|:-------:|:--------------------:|:------------------:|:-----------:|
| POST /{companyId}/token-collections | X | X | | |
| GET /{companyId}/token-collections | X | X | | |
| GET /{companyId}/token-collections/{id} | X | X | | |
| PUT /{companyId}/token-collections/{id} | X | X | | |
| PATCH /{companyId}/token-collections/publish/{id} | X | X | | |
| DELETE /{companyId}/token-collections/{id}/draft | X | X | | |
| DELETE /{companyId}/token-collections/{id}/burn | X | X | | |
| PATCH /{companyId}/token-collections/{id}/sync-draft-tokens | X | X | | |
| PATCH /{companyId}/token-collections/{id}/increase-editions | X | X | | |
| PATCH /{companyId}/token-collections/{id}/pass/enable | X | | | |
| PATCH /{companyId}/token-collections/{id}/pass/disable | X | | | |
| GET /{companyId}/token-collections/estimate-gas | X | X | | |
| GET /{companyId}/token-collections/{id}/export/xlsx | X | | | X |
| POST /{companyId}/token-collections/{id}/import/xlsx | X | | | X |
| GET /{companyId}/token-editions | X | | X | X |
| GET /{companyId}/token-editions/{id} | X | | X | |
| PATCH /{companyId}/token-editions/{id} | X | | X | |
| PATCH /{companyId}/token-editions/ready-to-mint | X | | X | |
| PATCH /{companyId}/token-editions/{id}/ready-to-mint | X | | X | |
| PATCH /{companyId}/token-editions/mint-on-demand | X | | X | |
| PATCH /{companyId}/token-editions/locked-for-buy | X | | X | |
| PATCH /{companyId}/token-editions/unlocked-for-buy | X | | X | |
| PATCH /{companyId}/token-editions/update-token-metadata | X | | X | |
| PATCH /{companyId}/token-editions/notify-externally-minted | X | | X | |
| GET /{companyId}/token-editions/check-rfid | X | | X | |
| PATCH /{companyId}/token-editions/{id}/rfid | X | | X | |
| DELETE /{companyId}/token-editions/burn | X | | X | |
| PATCH /{companyId}/token-editions/transfer-token | X | | X | |
| PATCH /{companyId}/token-editions/{id}/transfer-token | X | | X | |
| PATCH /{companyId}/token-editions/{id}/transfer-token/email | X | | X | |
| GET /{companyId}/token-editions/{id}/estimate-gas/mint | X | | X | |
| GET /{companyId}/token-editions/{id}/estimate-gas/transfer | X | | X | |
| GET /{companyId}/token-editions/{id}/estimate-gas/burn | X | | X | |
| GET /{companyId}/token-editions/{id}/get-last/{type} | X | | X | |
| GET /{companyId}/token-editions/xls | X | | | X |
| PATCH /{companyId}/token-editions/retry-bulk-by-collection | X | | | X |
