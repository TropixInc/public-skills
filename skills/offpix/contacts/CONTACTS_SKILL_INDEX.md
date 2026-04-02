---
id: CONTACTS_SKILL_INDEX
title: "Contacts Skill Index"
module: offpix/contacts
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Contacts Skill Index

Documentation for the W3Block Contacts module. Covers user management (invite, edit, roles), KYC document submission and approval workflows, royalty eligibility, external contacts, and payment provider configuration.

**Services:** PixwayID (users, KYC), Registry (royalty, external contacts), Commerce (payment providers)
**Swagger:** https://pixwayid.w3block.io/docs/ | https://api.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [CONTACTS_API_REFERENCE](./CONTACTS_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) | 1.0.0 | List, invite, edit contacts, roles, royalty, external contacts | Implemented | Manage users/contacts |
| 3 | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) | 1.0.0 | Fetch form fields, submit documents, multi-step, resubmission | Implemented | Build KYC submission UI |
| 4 | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) | 1.0.0 | List, review, approve/reject/request-review KYC submissions | Implemented | Build KYC review/admin UI |

---

## Quick Guide

### For a basic contacts implementation:

```
1. Read: FLOW_CONTACTS_USER_MANAGEMENT.md     → List/invite/edit users
2. Read: FLOW_CONTACTS_KYC_SUBMISSION.md      → Build document submission flow
3. Read: FLOW_CONTACTS_KYC_APPROVAL.md        → Build admin review flow
4. Consult: CONTACTS_API_REFERENCE.md         → For enums, DTOs, edge cases
```

### Prerequisites from other modules:

```
1. Auth module    → Sign-in (get JWT)
2. Configurations → Set up KYC contexts and form fields
3. Contacts       → Manage users and KYC flows
```

### Minimal API-first implementation:

```
1. POST /users/invite                                     → Create user
2. GET  /tenants/{id}/tenant-input/slug/{slug}            → Get form fields
3. POST /{id}/users/documents/{userId}/context/{ctxId}    → Submit KYC docs
4. GET  /customer-infos/{id}/search                       → List submissions
5. PATCH /{id}/users/contexts/{userId}/{ctxId}/approve    → Approve KYC
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | CPF must be unique per tenant | Each CPF can only be used once per tenant |
| 2 | Rejection is terminal | Use "require review" instead of "reject" if resubmission is desired |
| 3 | Multiface selfie requires CPF + birthdate | Submit CPF and birthdate documents BEFORE the selfie |
| 4 | Royalty eligibility is a separate API call | Inviting a user doesn't create royalty records — requires Registry backend call |
| 5 | File uploads need asset service first | Upload files to get `assetId`, then reference in document submission |
| 6 | KycApprover whitelist filtering | If `approverWhitelistIds` is set, only whitelisted approvers can review |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| List all clients/team/partners | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Invite a new user | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Edit user profile/roles | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Submit KYC documents | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Build a multi-step KYC form | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Handle document resubmission | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| List pending KYC reviews | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Approve/reject KYC submissions | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Request document resubmission | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Manage royalty eligibility | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Add external wallet contacts | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Set up KYC forms first | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | User Mgmt | KYC Submit | KYC Approve |
|----------|:-------:|:---------:|:----------:|:-----------:|
| GET /{id}/users | X | X | | |
| GET /{id}/users/{userId} | X | X | | |
| POST /users/invite | X | X | | |
| PATCH /{id}/users/{userId} | X | X | | |
| GET /users/documents/find-user-by-any | X | X | | |
| GET /users/wallets/by-address/{addr} | X | X | | |
| GET /tenant-input/{id}/slug/{slug} | | | X | |
| POST /users/documents/{userId}/context/{ctxId} | X | | X | |
| GET /users/documents/{userId} | X | | X | X |
| GET /users/documents/{userId}/context/{ctxId} | X | | X | X |
| GET /users/contexts/find | X | | | X |
| GET /users/contexts/{userId} | X | | | X |
| GET /users/contexts/{userId}/{ucId} | X | | | X |
| PATCH /users/contexts/{userId}/{ctxId}/approve | X | | | X |
| PATCH /users/contexts/{userId}/{ctxId}/reject | X | | | X |
| PATCH /users/contexts/{userId}/{ctxId}/require-review | X | | | X |
| GET /customer-infos/{id}/search | X | | | X |
| POST /{id}/contracts/royalty-eligible/create | X | X | | |
| PATCH /{id}/contracts/royalty-eligible | X | X | | |
| GET /{id}/external-contacts | X | X | | |
| POST /{id}/external-contacts/import | X | X | | |
| PATCH /users/{id}/{userId}/referrer | X | X | | |
