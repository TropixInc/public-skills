---
id: KYC_SKILL_INDEX
title: "KYC Skill Index"
module: offpix/kyc
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# KYC Skill Index

Documentation for the W3Block KYC (Know Your Customer) module. Covers document verification workflows including submission, approval, rejection, review requests, input type validation, and tenant-level KYC configuration.

**Service:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

> **Note:** Basic KYC flows are also referenced in the [Contacts module](../contacts/CONTACTS_SKILL_INDEX.md). This dedicated module provides deeper coverage of validation rules, input types, approval workflows, and configuration options.

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) | 1.0.0 | Fetch form, upload files, submit documents, multi-step, resubmission | Implemented | Build KYC submission UI |
| 3 | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) | 1.0.0 | List pending, review, approve/reject/require-review, audit trail | Implemented | Build KYC admin review UI |
| 4 | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) | 1.0.0 | Tenant context setup, auto-approve, approver whitelists, notifications | Implemented | Configure KYC for a tenant |

---

## Quick Guide

### For a complete KYC implementation:

```
1. Read: FLOW_KYC_CONFIGURATION.md      → Set up tenant contexts, inputs, approval rules
2. Read: FLOW_KYC_SUBMISSION.md         → Build the user-facing document submission flow
3. Read: FLOW_KYC_APPROVAL.md           → Build the admin review and approval flow
4. Consult: KYC_API_REFERENCE.md        → For enums, DTOs, validation rules, edge cases
```

### Prerequisites from other modules:

```
1. Auth module          → Sign-in (get JWT)
2. Configurations       → Set up KYC contexts and form fields (tenant-input, tenant-context)
3. Contacts (optional)  → User invite / management
```

### Minimal API-first implementation:

```
1. GET  /tenant-input/{tenantId}/slug/{slug}                  → Get form definition
2. POST /cloudinary/get-signature                              → Upload files (if needed)
3. POST /{tenantId}/users/documents/{userId}/context/{ctxId}   → Submit KYC documents
4. GET  /{tenantId}/users/contexts/find?status[]=created       → List pending submissions
5. PATCH /{tenantId}/users/contexts/{userId}/{ctxId}/approve   → Approve submission
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | CPF must be unique per tenant | Each CPF value can only be used by one user per tenant. Duplicate CPF submission returns `CPfDuplicatedException` |
| 2 | Email verification required | Users must have a verified email before submitting KYC documents (unless passwordless is enabled) |
| 3 | Multiface selfie requires CPF + birthdate | The biometric validation service requires CPF and birthdate documents to be submitted BEFORE the selfie |
| 4 | `maxSubmissions` limit | Each context can define a maximum number of submissions. Once reached, `MaxSubmissionReachedException` is thrown |
| 5 | DENIED is a terminal state | Rejection sets the context to `DENIED` permanently. Use "require review" instead if resubmission is desired |
| 6 | File uploads need asset service first | Upload files to Cloudinary via the asset service to get an `assetId`, then reference it in document submission |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Understand all KYC endpoints and DTOs | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
| Submit KYC documents for a user | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Build a multi-step KYC form | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Handle document resubmission after review | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| List pending KYC submissions | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Approve, reject, or request review | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Understand the audit trail | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Configure auto-approve or specific approver | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Set up approver whitelists | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Configure KYC email notifications | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Understand all 19 input types | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Validate a CPF number | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
| Block commerce orders until KYC approved | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Set up KYC form fields (inputs) | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Create a KYC context | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Submission | Approval | Configuration |
|----------|:-------:|:----------:|:--------:|:-------------:|
| GET /tenant-input/{id}/slug/{slug} | | X | | X |
| POST /cloudinary/get-signature | | X | | |
| POST /{id}/users/documents/{userId}/context/{ctxId} | X | X | | |
| GET /{id}/users/documents/{userId} | X | X | X | |
| GET /{id}/users/documents/{userId}/context/{ctxId} | X | X | X | |
| GET /documents/find-user-by-any | X | | X | |
| GET /{id}/users/contexts/find | X | | X | |
| GET /{id}/users/contexts/{userId} | X | | X | |
| GET /{id}/users/contexts/{userId}/{ucId} | X | | X | |
| PATCH /{id}/users/contexts/{userId}/{ctxId}/approve | X | | X | |
| PATCH /{id}/users/contexts/{userId}/{ctxId}/reject | X | | X | |
| PATCH /{id}/users/contexts/{userId}/{ctxId}/require-review | X | | X | |
| GET /tenant-context/{id} | X | | | X |
