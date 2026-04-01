---
id: CONTRACTS_SKILL_INDEX
title: "Contracts & Tokens Skill Index"
module: offpix/contracts
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Contracts & Tokens Skill Index

Documentation for the W3Block Contracts & Tokens module. Covers NFT (ERC721A) contract deployment, token collection and edition management, ERC20 fungible token lifecycle with transfer rules, royalty configuration, gas estimation, RFID tracking, bulk import/export, and public metadata/certificate endpoints.

**Service:** Pixway Registry | **Swagger:** https://api.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [CONTRACTS_API_REFERENCE](./CONTRACTS_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) | 1.0.0 | Create, configure, deploy NFT contracts with royalty | Implemented | Deploy new NFT contracts |
| 3 | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) | 1.0.0 | Collections, editions, templates, publish, bulk XLSX | Implemented | Manage token collections |
| 4 | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) | 1.0.0 | Mint, transfer, burn tokens, RFID, metadata, gas, certificates | Implemented | Operate on individual tokens |
| 5 | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) | 1.0.0 | ERC20 create, deploy, mint, transfer rules (5 handlers), burn, history | Implemented | Build loyalty/fungible token features |

---

## Quick Guide

### For NFT implementation:

```
1. Read: FLOW_CONTRACTS_NFT_LIFECYCLE.md           → Deploy an NFT contract
2. Read: FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md   → Create and publish collections
3. Read: FLOW_CONTRACTS_TOKEN_OPERATIONS.md        → Mint, transfer, burn tokens
4. Consult: CONTRACTS_API_REFERENCE.md             → For enums, DTOs, edge cases
```

### For ERC20/Loyalty implementation:

```
1. Read: FLOW_CONTRACTS_ERC20_LIFECYCLE.md         → Deploy and operate ERC20 tokens
2. Consult: CONTRACTS_API_REFERENCE.md             → For transfer config models, DTOs
```

### Minimal NFT flow (5 calls):

```
1. POST   /{id}/contracts                           → Create NFT contract
2. PATCH  /{id}/contracts/{id}/publish               → Deploy to blockchain
3. POST   /{id}/token-collections                    → Create collection
4. PATCH  /{id}/token-collections/{id}/sync-draft-tokens  → Generate editions
5. PATCH  /{id}/token-collections/publish/{id}       → Publish & mint
```

### Minimal ERC20 flow (3 calls):

```
1. POST   /{id}/erc20-contracts                     → Create ERC20 contract
2. PATCH  /{id}/erc20-contracts/{id}/publish         → Deploy to blockchain
3. PATCH  /{id}/erc20-tokens/{id}/mint               → Mint tokens
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Contract stuck in `publishing` | Blockchain confirmation takes time. Poll status or use webhooks |
| 2 | Can't create collection without contract | Deploy and publish an NFT contract first |
| 3 | Editions not appearing after collection create | Call `sync-draft-tokens` to generate edition records |
| 4 | ERC20 user transfer rejected | Check transfer config rules (FREE policy, MAX_LIMIT, PERIOD) |
| 5 | Gas estimation returns 0 | Ensure contract/collection has valid configuration |
| 6 | RFID already in use | RFIDs are globally unique — check availability before assigning |
| 7 | Published contracts are immutable | Create a new contract if you need different settings |
| 8 | Percentage/fixed fees create 2 transactions | By design — fee goes to company wallet separately |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Deploy a new NFT contract | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Configure royalty splits | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Create a token collection | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Bulk import editions via XLSX | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Publish a collection to blockchain | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Mint tokens on-demand | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Transfer tokens between wallets | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Burn tokens | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Assign RFID to a token | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Generate a PDF certificate | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Estimate gas costs | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Deploy an ERC20 (loyalty) token | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Configure transfer fees/limits | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Mint/transfer/burn ERC20 tokens | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| View ERC20 transaction history | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | NFT Life | Collections | Token Ops | ERC20 |
|----------|:-------:|:--------:|:-----------:|:---------:|:-----:|
| POST /contracts | X | X | | | |
| GET /contracts | X | X | | | |
| GET /contracts/:id | X | X | | | |
| PATCH /contracts/:id | X | X | | | |
| PATCH /contracts/:id/publish | X | X | | | |
| GET /contracts/:id/estimate-gas | X | X | | X | |
| PATCH /contracts/has-role | X | X | | | |
| PATCH /contracts/grant-role | X | X | | | |
| POST /token-collections | X | | X | | |
| GET /token-collections | X | | X | | |
| GET /token-collections/:id | X | | X | | |
| PUT /token-collections/:id | X | | X | | |
| PATCH /token-collections/publish/:id | X | | X | | |
| PATCH /token-collections/:id/sync-draft-tokens | X | | X | | |
| GET/POST /token-collections/:id/export-import/xlsx | X | | X | | |
| GET /token-collections/estimate-gas | X | | X | X | |
| GET /token-editions | X | | X | X | |
| PATCH /token-editions/ready-to-mint | X | | X | X | |
| PATCH /token-editions/mint-on-demand | X | | | X | |
| PATCH /token-editions/transfer-token | X | | | X | |
| DELETE /token-editions/burn | X | | | X | |
| PATCH /token-editions/update-token-metadata | X | | X | X | |
| GET /token-editions/check-rfid | X | | X | X | |
| PATCH /token-editions/:id/rfid | X | | X | X | |
| GET /token-editions/:id/estimate-gas/* | X | | | X | |
| POST /erc20-contracts | X | | | | X |
| GET /erc20-contracts | X | | | | X |
| PATCH /erc20-contracts/:id | X | | | | X |
| PATCH /erc20-contracts/:id/publish | X | | | | X |
| PATCH /erc20-contracts/:id/transfer-config | X | | | | X |
| PATCH /erc20-tokens/:id/mint | X | | | | X |
| PATCH /erc20-tokens/:id/transfer/* | X | | | | X |
| PATCH /erc20-tokens/:id/burn | X | | | | X |
| GET /erc20-tokens/:id/history/* | X | | | | X |
| POST/GET /contracts/royalty-eligible | X | X | | | |
| GET /metadata/* | X | | | X | |
| GET/POST /certification/* | X | | | X | |
| POST/GET /subcategories | X | | X | | |
