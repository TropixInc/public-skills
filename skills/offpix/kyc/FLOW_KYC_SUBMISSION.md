---
id: FLOW_KYC_SUBMISSION
title: "KYC - Document Submission"
module: offpix/kyc
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - submission
  - documents
  - verification
  - user
depends_on:
  - KYC_API_REFERENCE
---

# KYC Document Submission

## Overview

This flow covers how users submit KYC documents for verification. The process is: fetch the form definition (inputs), collect user data, optionally upload files, submit documents, and track the submission through the approval pipeline. Submissions follow the status lifecycle: `DRAFT -> CREATED -> APPROVED / DENIED / REQUIRED_REVIEW`.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT for the submitting user | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| `userId` | User UUID (typically the authenticated user) | Auth flow / user profile |
| Verified email | User must have a verified email (unless passwordless enabled) | Email verification on signup |
| Context slug or ID | The KYC context to submit against | Admin provides slug; or get from context list |

## Status Lifecycle

```
                     +-----------+
                     |   DRAFT   |  (incomplete submission)
                     +-----+-----+
                           |
                     all mandatory fields submitted
                           |
                     +-----v-----+
              +------|  CREATED  |------+
              |      +-----+-----+      |
              |            |            |
         approve      require-review   reject
              |            |            |
        +-----v-----+ +---v--------+ +-v--------+
        |  APPROVED  | | REQUIRED   | |  DENIED  |
        +------------+ |  _REVIEW   | +----------+
                       +---+--------+  (terminal)
                           |
                      user resubmits
                           |
                     +-----v-----+
                     |  CREATED  |  (back in queue)
                     +-----------+
```

| Status | Meaning | Triggered by |
|--------|---------|-------------|
| `DRAFT` | Submission started but not all mandatory fields provided | Partial document submission |
| `CREATED` | All mandatory documents submitted, awaiting review | Completing all required fields |
| `REQUIRED_REVIEW` | Approver flagged specific fields for resubmission | Admin/approver action |
| `APPROVED` | All documents approved | Admin action or auto-approve |
| `DENIED` | Submission rejected (terminal, no resubmission) | Admin action |

---

## Flow: Fetch Form Definition

Before submitting documents, the frontend fetches the form definition to know which fields to display.

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-input/slug/{slug}` | None (public) |

**Example:**
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
    "description": "Enter your CPF number (11 digits)",
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
    "label": "Date of Birth",
    "description": "You must be at least 18 years old",
    "type": "birthdate",
    "order": 3,
    "mandatory": true,
    "active": true,
    "attributeName": "birthdate",
    "step": 1,
    "options": []
  },
  {
    "id": "input-uuid-4",
    "label": "Country",
    "description": "Select your country",
    "type": "simple_select",
    "order": 4,
    "mandatory": true,
    "active": true,
    "attributeName": "country",
    "step": 1,
    "options": [
      { "label": "Brazil", "value": "BR" },
      { "label": "United States", "value": "US" },
      { "label": "Portugal", "value": "PT" }
    ]
  },
  {
    "id": "input-uuid-5",
    "label": "Government ID",
    "description": "Upload front and back of your ID document",
    "type": "identification_document",
    "order": 5,
    "mandatory": true,
    "active": true,
    "attributeName": "govId",
    "step": 2,
    "options": []
  },
  {
    "id": "input-uuid-6",
    "label": "Selfie Verification",
    "description": "Take a selfie for biometric verification",
    "type": "multiface_selfie",
    "order": 6,
    "mandatory": true,
    "active": true,
    "attributeName": "selfie",
    "step": 2,
    "options": []
  }
]
```

**Key fields for rendering:**
- `type`: determines the UI component (text input, file upload, select dropdown, etc.)
- `mandatory`: whether the field is required
- `step`: which step of the form this field belongs to (for multi-step forms)
- `order`: display order within the step
- `options`: available options for select-type inputs
- `active`: only render active inputs

---

## Flow: Upload File Assets

For file-based input types (`file`, `image`, `identification_document`, `multiface_selfie`), files must be uploaded to Cloudinary first to obtain an `assetId`.

### Step 1: Get Upload Signature

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| POST | `/cloudinary/get-signature` | Bearer (User) |

**Request:**
```json
{
  "folder": "kyc-documents"
}
```

**Response (200):**
```json
{
  "signature": "abcdef1234567890",
  "timestamp": 1705312000,
  "cloudName": "w3block",
  "apiKey": "123456789",
  "folder": "kyc-documents"
}
```

### Step 2: Upload to Cloudinary

Use the returned signature to upload the file directly to Cloudinary's upload API. The upload returns an `assetId` that you reference in the document submission.

### Step 3: Reference in Submission

Use the `assetId` in the `documents` array when submitting:

```json
{
  "inputId": "input-uuid-5",
  "assetId": "asset-uuid-from-upload"
}
```

---

## Flow: Submit Documents

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{tenantId}/users/documents/{userId}/context/{contextId}` | Bearer (User, Admin, KycApprover) | application/json |

### Minimal Request (text fields only, single step)

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
      "value": "1990-05-15"
    },
    {
      "inputId": "input-uuid-4",
      "value": "BR"
    }
  ]
}
```

### Complete Request (all field types, multi-step, with approver)

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
      "value": "1990-05-15"
    },
    {
      "inputId": "input-uuid-4",
      "value": "BR"
    },
    {
      "inputId": "input-uuid-5",
      "value": {
        "docType": "rg",
        "document": "123456789"
      },
      "assetId": "asset-uuid-front"
    },
    {
      "inputId": "input-uuid-6",
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

### Response (201)

```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "created",
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-1",
      "status": "created",
      "simpleValue": "John Doe"
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-2",
      "status": "created",
      "simpleValue": "12345678901"
    },
    {
      "id": "doc-uuid-3",
      "inputId": "input-uuid-3",
      "status": "created",
      "simpleValue": "1990-05-15"
    },
    {
      "id": "doc-uuid-4",
      "inputId": "input-uuid-4",
      "status": "created",
      "simpleValue": "BR"
    }
  ]
}
```

### Status Determination

- If all mandatory fields for the current step (or all steps) are submitted: status = `CREATED` (ready for review)
- If some mandatory fields are missing: status = `DRAFT` (incomplete)
- If `autoApprove: true` on the TenantContext: status = `APPROVED` immediately, no manual review needed

---

## Flow: Multi-Step Submission

For forms with multiple steps, submit documents incrementally using the `currentStep` parameter. The backend only validates mandatory fields up to the specified step.

### Step 1: Submit First Page

```json
{
  "documents": [
    { "inputId": "input-uuid-1", "value": "John Doe" },
    { "inputId": "input-uuid-2", "value": "12345678901" },
    { "inputId": "input-uuid-3", "value": "1990-05-15" },
    { "inputId": "input-uuid-4", "value": "BR" }
  ],
  "currentStep": 1
}
```

**Result:** Status = `DRAFT` (step 2 mandatory fields still missing). Response includes `userContextId` for the next step.

### Step 2: Submit Second Page

```json
{
  "documents": [
    { "inputId": "input-uuid-5", "value": { "docType": "rg", "document": "123456789" }, "assetId": "asset-uuid-front" },
    { "inputId": "input-uuid-6", "assetId": "asset-uuid-selfie" }
  ],
  "currentStep": 2,
  "userContextId": "uc-uuid-from-step-1"
}
```

**Result:** Status = `CREATED` (all mandatory fields now submitted).

**Important:** Always include `userContextId` from the Step 1 response when submitting subsequent steps. This ensures documents are attached to the same user context rather than creating a new one.

---

## Flow: Resubmission After Review

When an approver flags specific fields via "require review," the user can resubmit only those fields.

### Step 1: Check Which Fields Need Review

```
GET /{tenantId}/users/contexts/{userId}/{userContextId}
```

Look for documents with `status: "requiredReview"` -- those are the fields that need resubmission.

**Example response snippet:**
```json
{
  "status": "requiredReview",
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-1",
      "status": "created",
      "simpleValue": "John Doe"
    },
    {
      "id": "doc-uuid-5",
      "inputId": "input-uuid-5",
      "status": "requiredReview",
      "assetId": "asset-uuid-old"
    },
    {
      "id": "doc-uuid-6",
      "inputId": "input-uuid-6",
      "status": "requiredReview",
      "assetId": "asset-uuid-old-selfie"
    }
  ],
  "logs": [
    {
      "moderatorId": "admin-uuid",
      "status": "requiredReview",
      "inputIds": ["input-uuid-5", "input-uuid-6"],
      "reason": "ID document is expired, selfie is blurry",
      "registerAt": "2026-01-16T14:00:00Z"
    }
  ]
}
```

### Step 2: Resubmit Flagged Fields

```json
{
  "documents": [
    { "inputId": "input-uuid-5", "value": { "docType": "rg", "document": "123456789" }, "assetId": "new-asset-uuid-front" },
    { "inputId": "input-uuid-6", "assetId": "new-asset-uuid-selfie" }
  ],
  "userContextId": "uc-uuid"
}
```

**Result:** Documents return to `CREATED` status. User context returns to `CREATED` (back in the review queue).

---

## Special Processing by Input Type

### CPF Processing

1. Non-numeric characters are stripped (e.g., `"123.456.789-01"` becomes `"12345678901"`)
2. Length validated: must be exactly 11 digits
3. Checksum validated using the Brazilian CPF mod-11 algorithm
4. Tenant uniqueness check: no other user in the same tenant can have the same CPF
5. Stored as digits only in `simpleValue`

### USER_NAME Processing

1. Value stored as `simpleValue`
2. Automatically updates the user's `name` field on their profile record
3. This means the user's display name changes throughout the platform

### MULTIFACE_SELFIE Processing

1. Validates that the user has already submitted a CPF document (any context, same tenant)
2. Validates that the user has already submitted a birthdate document (any context, same tenant)
3. Uploads the selfie asset
4. Calls the external biometric validation service with the selfie, CPF, and birthdate
5. If biometric validation fails, throws `ItWasNotPossibleRegisterMutiplaceExpection`
6. Stored as `assetId`

### COMMERCE_PRODUCT Processing

1. Validates that the referenced `productId` exists in the commerce service
2. Validates quantity and optional variant IDs
3. Stored as `complexValue`

### BIRTHDATE Processing

1. Validates the date is a valid ISO date
2. Checks minimum age requirement (default: 18 years from current date)
3. Stored as `simpleValue`

### SIMPLE_SELECT Processing

1. Validates the submitted value matches one of the `options[].value` from the input definition
2. If `isMultiple` is true on the input, accepts an array of values (each must match an option)
3. Stored as `simpleValue`

---

## Validation Chain

The backend performs these validations in order on each submission:

1. **User verification** -- email must be verified (unless passwordless is enabled for the tenant)
2. **Permission check** -- user can only submit their own documents (unless Admin or Integration role)
3. **Context active check** -- the TenantContext must be active
4. **Max submissions check** -- user has not exceeded the `maxSubmissions` limit for this context
5. **Asset validation** -- all referenced `assetId` values must correspond to existing Cloudinary assets
6. **Type-specific validation** -- each document value is validated according to its input type rules
7. **Required fields check** -- if `currentStep` covers all steps, all mandatory inputs must have documents
8. **Approver assignment** -- if `requireSpecificApprover: true`, the `approverUserId` must be provided
9. **CPF uniqueness** -- no duplicate CPF values allowed per tenant
10. **Multiface biometric** -- if multiface_selfie type, biometric service is called

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | `BadRequestException` | Missing required document or invalid value | Check field requirements and value format |
| 400 | `CPfDuplicatedException` | CPF already used by another user in this tenant | Each CPF must be unique per tenant |
| 400 | `InvalidCPFException` | CPF checksum validation failed | Verify CPF is 11 digits with valid checksum |
| 400 | `MaxSubmissionReachedException` | User exceeded max submissions for this context | Context's `maxSubmissions` limit reached |
| 400 | `TenantContextDisabledException` | Context is inactive | Contact admin to activate the context |
| 403 | `NotAuthorizedAttachmentDocumentToContextException` | User cannot submit to this context | Check permissions and context access |
| 403 | `UserNeedVerifyAccount` | Email not verified | Complete email verification first |
| 404 | `UserContextNotFoundException` | User context not found (invalid `userContextId`) | Verify the `userContextId` from a previous submission |
| 500 | `ItWasNotPossibleRegisterMutiplaceExpection` | Biometric validation service failed | Retry with a clearer selfie; ensure CPF and birthdate are submitted first |

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Email not verified error | Users must verify their email before submitting KYC documents. Check `UserNeedVerifyAccount` exception. Passwordless tenants are exempt |
| 2 | CPF duplicate rejection | CPF values must be unique per tenant. If another user has already submitted the same CPF, submission fails with `CPfDuplicatedException` |
| 3 | Multiface selfie fails | The biometric service requires CPF and birthdate documents to exist for the user BEFORE submitting the selfie. Submit those fields first |
| 4 | Status stays DRAFT | Not all mandatory fields have been submitted. Check which inputs have `mandatory: true` and ensure all are included |
| 5 | Resubmission creates new context | When resubmitting after "require review," always include `userContextId` to attach to the existing context |
| 6 | `maxSubmissions` exceeded | The context has a limit on submissions. Once reached, no new submissions are allowed for that user/context pair |
| 7 | `simple_select` value rejected | The submitted value must exactly match one of the `options[].value` defined on the input. Check case sensitivity |
| 8 | File upload not accepted | Upload the file to Cloudinary first via the asset service, get the `assetId`, then reference it in the document submission |

---

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| KYC Approval | Admin reviews and acts on submissions | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| KYC Configuration | Tenant context and input setup | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Context Setup | How KYC contexts are created | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Input Management | How form fields are defined | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Auth Sign-In | Obtain JWT for submission | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| API Reference | Full endpoint and DTO details | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
