---
id: FLOW_CONTACTS_KYC_SUBMISSION
title: "Contacts - KYC Document Submission"
module: offpix/contacts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contacts
  - kyc
  - documents
  - submission
depends_on:
  - CONTACTS_API_REFERENCE
  - CONFIGURATIONS_API_REFERENCE
  - FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
  - FLOW_CONFIGURATIONS_INPUT_MANAGEMENT
---

# KYC Document Submission

## Overview

This flow covers how users submit KYC documents against a context form. The process is: fetch the form definition (inputs), collect user data, submit documents, and track the submission status through the approval pipeline. Submissions follow the status lifecycle: `DRAFT → CREATED → APPROVED / DENIED / REQUIRED_REVIEW`.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT for the submitting user | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Verified email | User must have a verified email | Email verification on signup |
| Active context | A KYC context must be configured and active | [Context Lifecycle](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

## Entities & Status Lifecycle

```
Status Lifecycle:

DRAFT ──→ CREATED ──→ APPROVED
              │
              ├──→ DENIED (terminal)
              │
              └──→ REQUIRED_REVIEW ──→ CREATED (user resubmits)
                        │
                        └──→ (cycle continues until APPROVED or DENIED)
```

| Status | Meaning | Triggered by |
|--------|---------|-------------|
| `DRAFT` | Submission started but incomplete | Initial document submission (not all mandatory fields) |
| `CREATED` | All mandatory documents submitted, awaiting review | Completing all required fields |
| `REQUIRED_REVIEW` | Approver asked user to resubmit specific fields | Admin action |
| `APPROVED` | All documents approved | Admin action (or auto-approve) |
| `DENIED` | Documents rejected | Admin action |

---

## Flow: Submit KYC Documents

### Step 1: Fetch Form Fields

Get the form definition to know which fields to collect.

**Endpoint (Public):**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-input/slug/{slug}` | None |

Example for the signup form:

```
GET /tenants/{tenantId}/tenant-input/slug/signup
```

**Response (200):**
```json
[
  {
    "id": "input-uuid-1",
    "label": "Full Name",
    "description": "Enter your full legal name",
    "type": "user_name",
    "order": 1,
    "mandatory": true,
    "active": true,
    "attributeName": "fullName",
    "step": 1,
    "options": []
  },
  {
    "id": "input-uuid-2",
    "label": "CPF",
    "description": "Enter your CPF number",
    "type": "cpf",
    "order": 2,
    "mandatory": true,
    "active": true,
    "attributeName": "cpf",
    "step": 1,
    "options": []
  },
  {
    "id": "input-uuid-3",
    "label": "Government ID",
    "description": "Upload front and back of your ID",
    "type": "identification_document",
    "order": 3,
    "mandatory": true,
    "active": true,
    "attributeName": "govId",
    "step": 2,
    "options": []
  },
  {
    "id": "input-uuid-4",
    "label": "Selfie",
    "description": "Take a selfie holding your ID",
    "type": "multiface_selfie",
    "order": 4,
    "mandatory": true,
    "active": true,
    "attributeName": "selfie",
    "step": 2,
    "options": []
  }
]
```

### Step 2: Upload File Assets (if needed)

For file-based inputs (`file`, `image`, `identification_document`, `multiface_selfie`), upload the file first to get an `assetId`.

Files are uploaded to Cloudinary through the W3Block asset service. The upload flow returns an `assetId` that you reference in the document submission.

### Step 3: Submit Documents

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{tenantId}/users/documents/{userId}/context/{contextId}` | Bearer (User) | application/json |

**Minimal Request (text fields only):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-1",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-2",
      "value": "12345678901"
    }
  ]
}
```

**Complete Request (all field types, production example):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-1",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-2",
      "value": "12345678901"
    },
    {
      "inputId": "input-uuid-3",
      "assetId": "asset-uuid-front"
    },
    {
      "inputId": "input-uuid-4",
      "assetId": "asset-uuid-selfie"
    }
  ],
  "currentStep": 2,
  "approverUserId": "approver-uuid",
  "utmParams": {
    "utm_campaign": "onboarding",
    "utm_source": "email"
  }
}
```

**Response (201):**
```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "CREATED",
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-1",
      "status": "CREATED",
      "simpleValue": "John Doe"
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-2",
      "status": "CREATED",
      "simpleValue": "12345678901"
    }
  ]
}
```

**Notes:**
- If all mandatory fields are included, status transitions to `CREATED` (ready for review)
- If some mandatory fields are missing, status stays as `DRAFT`
- If the context has `autoApprove: true`, documents are immediately set to `APPROVED`
- `currentStep` is used for multi-step form validation — only validates mandatory fields up to that step
- The `user_name` input type automatically updates the user's `name` profile field

---

## Value Format Reference

### Simple Values (stored as `simpleValue`)

| Type | Format | Example | Validation |
|------|--------|---------|------------|
| `text` | string | `"John Doe"` | None |
| `email` | string | `"john@example.com"` | Email format |
| `url` | string | `"https://example.com"` | Valid HTTP(S) URL |
| `phone` | string | `"+5511999999999"` | Phone format |
| `cpf` | string (digits only) | `"12345678901"` | 11 digits, valid CPF checksum, unique per tenant |
| `date` | ISO date | `"2026-01-15"` | Valid date |
| `birthdate` | ISO date | `"1990-05-15"` | Valid date |
| `user_name` | string | `"John Doe"` | Updates user.name |
| `simple_select` | string | `"BR"` | Must be in input.options values |
| `checkbox` | boolean | `true` | Boolean |

### Complex Values (stored as `complexValue`)

| Type | Format | Example |
|------|--------|---------|
| `identification_document` | `{ docType, document }` | `{ "docType": "RG", "document": "123456789" }` |
| `simple_location` | `{ region, country, placeId, city?, ... }` | `{ "region": "SP", "country": "BR", "placeId": "ChIJ...", "city": "Sao Paulo" }` |
| `commerce_product` | `{ productId, quantity, variantIds? }` | `{ "productId": "uuid", "quantity": "1" }` |

### File-Based Values (use `assetId`)

| Type | Notes |
|------|-------|
| `file` | Generic file upload |
| `image` | Image file |
| `identification_document` | May combine `assetId` + `complexValue` |
| `multiface_selfie` | Triggers biometric validation. Requires user to have submitted CPF and birthdate |

---

## Multi-Step Submission

For forms with multiple steps, submit documents incrementally:

### Submit Step 1

```json
{
  "documents": [
    { "inputId": "name-input", "value": "John Doe" },
    { "inputId": "email-input", "value": "john@example.com" }
  ],
  "currentStep": 1
}
```

Status: `DRAFT` (step 2 fields still missing)

### Submit Step 2

```json
{
  "documents": [
    { "inputId": "id-doc-input", "assetId": "asset-uuid" },
    { "inputId": "selfie-input", "assetId": "asset-uuid" }
  ],
  "currentStep": 2,
  "userContextId": "uc-uuid-from-step-1"
}
```

Status: `CREATED` (all mandatory fields submitted)

**Important:** Include `userContextId` from Step 1 response to attach documents to the same user context.

---

## Resubmission (After Required Review)

When an approver flags specific fields for review, the user resubmits only those fields:

### Step 1: Check Which Fields Need Review

```
GET /{tenantId}/users/contexts/{userId}/{userContextId}
```

Look at documents with `status: "REQUIRED_REVIEW"` — those are the fields to resubmit.

### Step 2: Resubmit Flagged Fields

```json
{
  "documents": [
    { "inputId": "flagged-input-1", "assetId": "new-asset-uuid" },
    { "inputId": "flagged-input-2", "value": "corrected value" }
  ],
  "userContextId": "uc-uuid"
}
```

Status transitions: `REQUIRED_REVIEW → CREATED` (back in the approval queue).

---

## Auto-Approve Flow

If the context has `autoApprove: true`:

1. User submits documents
2. All documents immediately set to `APPROVED`
3. User context status → `APPROVED`
4. Commerce KYC event dispatched
5. Approval email sent automatically
6. No manual review needed

---

## Validation Chain (Internal)

The API performs these validations on submission:

1. **User verification** — email must be verified (unless passwordless enabled)
2. **Permission check** — user can only submit own documents (unless Admin/Integration)
3. **Asset validation** — all file assets must exist and be on Cloudinary
4. **Type-specific validation** — each value validated per input type
5. **Required fields check** — all mandatory inputs must have documents before `CREATED`
6. **Approver assignment** — must be set if `requireSpecificApprover: true`
7. **CPF uniqueness** — no duplicate CPF values per tenant
8. **Multiface biometric** — selfie validated against external service (requires CPF + birthdate)

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | BadRequestException | Missing required document/invalid value | Check field requirements and value format |
| 400 | CPfDuplicatedException | CPF already used by another user in this tenant | Each CPF must be unique per tenant |
| 400 | InvalidCPFException | CPF checksum validation failed | Verify CPF number (11 digits, valid checksum) |
| 400 | MaxSubmissionReachedException | User exceeded max submissions for this context | Context's `maxSubmissions` limit reached |
| 400 | TenantContextDisabledException | Context is inactive | Contact admin to activate the context |
| 403 | NotAuthorizedAttachmentDocumentToContextException | User can't submit to this context | Check permissions and context access |
| 403 | UserNeedVerifyAccount | Email not verified | Complete email verification first |
| 404 | UserContextNotFoundException | User context not found | Verify userContextId |
| 500 | ItWasNotPossibleRegisterMutiplaceExpection | Biometric validation failed | Retry with a clearer selfie |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | CPF rejected on resubmission | CPF must be unique per tenant. If another user already has this CPF, submission fails |
| 2 | Multiface selfie fails | User must have submitted both CPF and birthdate documents BEFORE the selfie. The biometric service requires these |
| 3 | Status stays DRAFT | Not all mandatory fields have been submitted. Check `mandatory: true` fields and submit missing ones |
| 4 | File upload rejected | Upload file to the asset service first, get `assetId`, then reference it in the document submission |
| 5 | Resubmission creates new context | Include `userContextId` when resubmitting to attach to the existing context instead of creating a new one |
| 6 | `simple_select` value rejected | The submitted value must exactly match one of the `options[].value` defined in the input |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Context Setup | Forms must be configured before submission | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Input Management | Form fields must exist | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| KYC Approval | Admin reviews submissions | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| User Management | User must exist first | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
