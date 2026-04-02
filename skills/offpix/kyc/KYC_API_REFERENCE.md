---
id: KYC_API_REFERENCE
title: "KYC - API Reference"
module: offpix/kyc
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - documents
  - verification
  - approval
  - api-reference
---

# KYC API Reference

Complete endpoint reference for the W3Block KYC (Know Your Customer) module. This module covers document submission, validation, approval workflows, and tenant-level KYC configuration. All endpoints are served by the PixwayID backend.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://pixwayid.w3block.io` |
| Swagger | https://pixwayid.w3block.io/docs/ |

> **Note:** A staging environment is also available for development and testing purposes.

## Authentication

All endpoints require Bearer token unless explicitly marked as public:

```
Authorization: Bearer {accessToken}
```

---

## Enums

### UserDocumentStatus

Status of an individual document within a submission.

| Value | String | Description |
|-------|--------|-------------|
| `CREATED` | `created` | Document submitted, awaiting review |
| `APPROVED` | `approved` | Document approved by moderator |
| `DENIED` | `denied` | Document rejected by moderator |
| `REQUIRED_REVIEW` | `requiredReview` | Moderator requested resubmission of this document |

### UserContextStatus

Status of the overall user context (aggregation of all documents in a submission).

| Value | String | Description |
|-------|--------|-------------|
| `DRAFT` | `draft` | Submission started but not all mandatory fields provided |
| `CREATED` | `created` | All mandatory documents submitted, awaiting review |
| `REQUIRED_REVIEW` | `requiredReview` | Moderator flagged specific fields for resubmission |
| `APPROVED` | `approved` | All documents approved |
| `DENIED` | `denied` | Submission rejected (terminal state) |

### DataTypesEnum

The 19 supported input types for KYC form fields.

| Value | Description | Storage | Validation |
|-------|-------------|---------|------------|
| `TEXT` | Free-form text | `simpleValue` | None |
| `EMAIL` | Email address | `simpleValue` | RFC 5322 format |
| `PHONE` | Phone number | `simpleValue` | Phone format; supports array or single with `extraPhones` |
| `URL` | Web URL | `simpleValue` | Valid HTTP(S) URL |
| `CPF` | Brazilian tax ID | `simpleValue` | 11 digits, checksum validation, unique per tenant |
| `BIRTHDATE` | Date of birth | `simpleValue` | Valid date, minimum age check (default 18) |
| `DATE` | Generic date | `simpleValue` | Valid ISO date |
| `USER_NAME` | User's full name | `simpleValue` | Auto-updates `user.name` profile field |
| `CHECKBOX` | Boolean checkbox | `simpleValue` | Boolean value |
| `SIMPLE_SELECT` | Dropdown/radio selection | `simpleValue` | Must match `option.value`; respects `isMultiple` flag |
| `DYNAMIC_SELECT` | Server-populated select | `simpleValue` | Options fetched dynamically |
| `FILE` | Generic file upload | `assetId` | Requires uploaded asset (Cloudinary) |
| `IMAGE` | Image upload | `assetId` | Requires uploaded asset (Cloudinary) |
| `IDENTIFICATION_DOCUMENT` | Government ID | `complexValue` + `assetId` | Requires `docType` (passport, cpf, rg) and `document` value |
| `MULTIFACE_SELFIE` | Biometric selfie | `assetId` | Requires CPF + birthdate submitted first; triggers biometric service |
| `SIMPLE_LOCATION` | Address/location | `complexValue` | Requires `placeId`, `region`, `country` |
| `COMMERCE_PRODUCT` | Product reference | `complexValue` | Validates product exists in commerce service |
| `SEPARATOR` | Visual separator | N/A | Not stored; skipped during processing |
| `IFRAME` | Embedded iframe | N/A | Display-only; not stored |

### TenantContextNotificationType

Email notification types configurable per tenant context.

| Value | Recipient | Description |
|-------|-----------|-------------|
| `KYC_APPROVAL_REQUEST` | Approvers | Sent when a new submission is created and ready for review |
| `KYC_REQUIRE_USER_REVIEW` | User | Sent when a moderator requests resubmission of specific fields |
| `KYC_USER_APPROVED` | User | Sent when the user's submission is approved |
| `KYC_ADMIN_APPROVED` | Admin | Sent to admins when a submission is approved |
| `KYC_USER_REJECTED` | User | Sent when the user's submission is rejected |
| `KYC_ADMIN_REJECTED` | Admin | Sent to admins when a submission is rejected |

---

## Entities & Relationships

```
User (1) ──→ (N) UsersContextsEntity ──→ (N) UsersDocumentEntity
                       │                           │
                       │ status, logs[],            │ inputId, simpleValue,
                       │ approverUserId,            │ complexValue, assetId,
                       │ utmParams                  │ status
                       │
TenantContextEntity ───┘ (defines approval config)
       │
       └──→ (N) TenantInput (defines form fields)
```

### UsersContextsEntity

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant this context belongs to |
| `userId` | UUID | User who submitted |
| `contextId` | UUID | Reference to the TenantContext definition |
| `status` | UserContextStatus | Current submission status |
| `logs` | LogUserContext[] | Audit trail of all status changes |
| `approverUserId` | UUID (nullable) | Assigned approver (when `requireSpecificApprover` is true) |
| `utmParams` | object (nullable) | UTM tracking parameters from submission |
| `createdAt` | DateTime | Creation timestamp |
| `updatedAt` | DateTime | Last update timestamp |

### UsersDocumentEntity

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant this document belongs to |
| `userId` | UUID | User who submitted |
| `inputId` | UUID | Reference to the TenantInput definition |
| `contextId` | UUID | Reference to the TenantContext |
| `userContextId` | UUID (nullable) | Reference to the UsersContextsEntity |
| `status` | UserDocumentStatus | Current document status |
| `simpleValue` | string (nullable) | Stored value for simple types (text, email, cpf, etc.) |
| `complexValue` | object (nullable) | Stored value for complex types (location, identification, etc.) |
| `assetId` | UUID (nullable) | Reference to uploaded file asset (Cloudinary) |
| `createdAt` | DateTime | Creation timestamp |
| `updatedAt` | DateTime | Last update timestamp |

### TenantContextEntity

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `tenantId` | UUID | Tenant this context belongs to |
| `contextId` | UUID | Reference to the context definition |
| `active` | boolean | Whether this KYC context is active |
| `data` | object (nullable) | Additional configuration data |
| `approverWhitelistIds` | UUID[] (nullable) | Whitelist IDs restricting which approvers can review |
| `requireSpecificApprover` | boolean | Whether submissions must specify an `approverUserId` |
| `autoApprove` | boolean | Whether submissions are auto-approved on creation |
| `blockCommerceDeliver` | boolean | Whether to block commerce order delivery until KYC is approved |
| `notificationsConfig` | object (nullable) | Email notification configuration per notification type |

### LogUserContext

| Field | Type | Description |
|-------|------|-------------|
| `moderatorId` | UUID | User who performed the action |
| `status` | UserContextStatus | Status set by this action |
| `inputIds` | UUID[] | Input IDs affected (for require-review actions) |
| `reason` | string (nullable) | Reason provided for the action |
| `registerAt` | DateTime | When the action was performed |

---

## Endpoints: Document Submission

### POST /{tenantId}/users/documents/{userId}/context/{contextId} -- Submit Documents

**Auth:** Bearer (User, Admin, KycApprover)

Submit one or more documents against a KYC context. Creates or updates a user context submission.

**Request (Minimal -- text fields only):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-name",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-cpf",
      "value": "12345678901"
    }
  ]
}
```

**Request (Complete -- all options):**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-name",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-cpf",
      "value": "12345678901"
    },
    {
      "inputId": "input-uuid-email",
      "value": "john@example.com"
    },
    {
      "inputId": "input-uuid-location",
      "value": {
        "region": "SP",
        "country": "BR",
        "placeId": "ChIJrTLr-GyuEmsRBfy61i59si0",
        "city": "Sao Paulo",
        "postal_code": "01000-000",
        "street_address_1": "Av Paulista 1000"
      }
    },
    {
      "inputId": "input-uuid-id-doc",
      "value": {
        "docType": "rg",
        "document": "123456789"
      },
      "assetId": "asset-uuid-front-photo"
    },
    {
      "inputId": "input-uuid-selfie",
      "assetId": "asset-uuid-selfie"
    }
  ],
  "currentStep": 2,
  "approverUserId": "approver-uuid",
  "userContextId": "existing-uc-uuid",
  "utmParams": {
    "utm_campaign": "onboarding",
    "utm_source": "email",
    "utm_medium": "link"
  }
}
```

**DTO: AttachDocumentsToUser**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `documents` | DocumentDto[] | Yes (min: 1) | Array of document submissions |
| `currentStep` | number | No | Current form step for multi-step validation |
| `approverUserId` | UUID | Conditional | Required if context has `requireSpecificApprover: true` |
| `userContextId` | UUID | No | Attach to existing user context (for resubmission or multi-step) |
| `utmParams` | UTMParamsDto | No | UTM tracking parameters |

**DTO: DocumentDto**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inputId` | UUID | Yes | TenantInput ID this value belongs to |
| `value` | string or object | Conditional | Field value (for text types or complex types) |
| `assetId` | UUID | Conditional | Uploaded file asset ID (for file/image types) |

**Response (201):** `UsersContextsEntity` with documents.

**Status Transitions:**
- If not all mandatory fields submitted: status = `DRAFT`
- If all mandatory fields submitted: status = `CREATED`
- If `autoApprove` is true on context: status = `APPROVED` immediately

---

### GET /{tenantId}/users/documents/{userId} -- Get User Documents (Paginated)

**Auth:** Bearer (Admin, User -- user can only fetch own)

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | `1` | Page number |
| `limit` | integer | `10` | Items per page |
| `type[]` | DataTypesEnum[] | -- | Filter by input types |
| `contextId` | UUID | -- | Filter by context |
| `inputId` | UUID | -- | Filter by specific input |

**Response (200):** Paginated list of `UsersDocumentEntity` with input and asset relations.

---

### GET /{tenantId}/users/documents/{userId}/context/{contextId} -- Get Documents by Context

**Auth:** Bearer (Admin, User -- user can only fetch own)

Returns all documents for a specific context as a non-paginated array.

**Response (200):**
```json
[
  {
    "id": "doc-uuid-1",
    "inputId": "input-uuid-name",
    "status": "created",
    "simpleValue": "John Doe",
    "complexValue": null,
    "assetId": null,
    "input": {
      "id": "input-uuid-name",
      "label": "Full Name",
      "type": "user_name",
      "mandatory": true
    },
    "asset": null
  },
  {
    "id": "doc-uuid-2",
    "inputId": "input-uuid-file",
    "status": "created",
    "simpleValue": null,
    "complexValue": null,
    "assetId": "asset-uuid",
    "input": {
      "id": "input-uuid-file",
      "label": "Government ID",
      "type": "identification_document",
      "mandatory": true
    },
    "asset": {
      "id": "asset-uuid",
      "url": "https://res.cloudinary.com/w3block/image/upload/v1/...",
      "mimeType": "image/jpeg"
    }
  }
]
```

---

### GET /documents/find-user-by-any -- Find User by CPF or ID

**Auth:** Bearer (Admin, Integration)

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `cpf` | string | CPF number (11 digits) |
| `userId` | UUID | User ID |

At least one of `cpf` or `userId` is required. CPF lookup searches across submitted KYC documents within the tenant.

**Response (200):**
```json
{
  "id": "user-uuid",
  "name": "John Doe",
  "email": "john@example.com",
  "tenantId": "tenant-uuid"
}
```

---

## Endpoints: User Contexts

### GET /{tenantId}/users/contexts/find -- List Contexts with Filters

**Auth:** Bearer (User, Admin, KycApprover)

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | `1` | Page number |
| `limit` | integer | `10` | Items per page |
| `search` | string | -- | Search in user email/name |
| `sortBy` | array | `[{column: 'createdAt', order: 'ASC'}]` | Sort configuration |
| `status[]` | UserContextStatus[] | -- | Filter by status(es) |
| `contextId[]` | UUID[] | -- | Filter by context IDs |
| `contextType[]` | ContextType[] | -- | Filter by `user_properties` or `form` |
| `userId[]` | UUID[] | -- | Filter by user IDs (Admin only) |
| `excludeSelfContexts` | boolean | -- | KycApprover: exclude own submissions |
| `preOrder` | boolean | -- | Filter pre-order contexts |
| `utmCampaign` | string | -- | Filter by UTM campaign |
| `utmSource` | string | -- | Filter by UTM source |

**Response (200):**
```json
{
  "items": [
    {
      "id": "uc-uuid",
      "tenantId": "tenant-uuid",
      "userId": "user-uuid",
      "contextId": "ctx-uuid",
      "status": "created",
      "approverUserId": null,
      "utmParams": null,
      "createdAt": "2026-01-15T10:00:00Z",
      "updatedAt": "2026-01-15T10:00:00Z",
      "user": {
        "name": "John Doe",
        "email": "john@example.com"
      },
      "context": {
        "slug": "signup",
        "description": "SignUp Form",
        "type": "user_properties"
      }
    }
  ],
  "meta": {
    "totalItems": 50,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 5,
    "currentPage": 1
  }
}
```

---

### GET /{tenantId}/users/contexts/{userId} -- Get User Contexts

**Auth:** Bearer (Admin, User -- user can only see own)

**Query Parameters:** `page`, `limit`, `search`, `sortBy`, `status`, `contextId`

Returns paginated list of user contexts for a specific user.

---

### GET /{tenantId}/users/contexts/{userId}/{userContextId} -- Get Context with Documents

**Auth:** Bearer (User, Admin, KycApprover)

**Response (200):**
```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "created",
  "approverUserId": null,
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "utmParams": null,
  "documents": [
    {
      "id": "doc-uuid-1",
      "inputId": "input-uuid-name",
      "status": "created",
      "simpleValue": "John Doe",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid-name",
        "label": "Full Name",
        "type": "user_name",
        "mandatory": true
      },
      "asset": null
    }
  ],
  "context": {
    "id": "ctx-uuid",
    "slug": "signup",
    "description": "SignUp Form",
    "type": "user_properties"
  },
  "user": {
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

---

## Endpoints: Approval Actions

### PATCH /{tenantId}/users/contexts/{userId}/{contextId}/approve -- Approve Submission

**Auth:** Bearer (Admin, KycApprover)

**DTO: UserContextStatusDto**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Approval reason (saved in audit log) |
| `userContextId` | UUID | No | Specific user context to approve |

**Request (Minimal):**
```json
{}
```

**Request (Complete):**
```json
{
  "reason": "All documents verified successfully",
  "userContextId": "uc-uuid"
}
```

**Response:** 204 No Content

**Side Effects:**
1. All documents in the context set to `APPROVED`
2. User context status set to `APPROVED`
3. Audit log entry appended with moderator ID, timestamp, reason
4. KYC approval email sent to user (if configured)
5. Commerce service notified (may unblock orders if `blockCommerceDeliver` is true)
6. Webhook `USER_KYC_PROPS_UPDATED` dispatched

---

### PATCH /{tenantId}/users/contexts/{userId}/{contextId}/reject -- Reject Submission

**Auth:** Bearer (Admin, KycApprover)

**DTO: UserContextStatusDto**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Rejection reason (sent to user via email) |
| `userContextId` | UUID | No | Specific user context to reject |

**Request:**
```json
{
  "reason": "Document is expired. Please contact support for further assistance."
}
```

**Response:** 204 No Content

**Side Effects:**
1. All documents in the context set to `DENIED`
2. User context status set to `DENIED`
3. Audit log entry appended
4. KYC rejection email sent to user with reason
5. Commerce service notified

**IMPORTANT:** `DENIED` is a terminal state. The user cannot resubmit to a denied context. Use "require review" instead if resubmission is desired.

---

### PATCH /{tenantId}/users/contexts/{userId}/{contextId}/require-review -- Request Review

**Auth:** Bearer (Admin, KycApprover)

**DTO: RequiredReviewContextStatusDto**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inputIds` | UUID[] | Yes (min: 1) | Input IDs of fields that need resubmission |
| `reason` | string | No | Explanation sent to the user |

**Request:**
```json
{
  "inputIds": ["input-uuid-id-doc", "input-uuid-selfie"],
  "reason": "Government ID is blurry and selfie does not match the document photo"
}
```

**Response:** 204 No Content

**Side Effects:**
1. Only the specified documents set to `REQUIRED_REVIEW`
2. Other documents retain their current status
3. User context status set to `REQUIRED_REVIEW`
4. Audit log entry appended with `inputIds` and `reason`
5. Review request email sent to user specifying which fields need attention

---

## Input Type Validation Rules

### CPF

| Rule | Detail |
|------|--------|
| Format | Exactly 11 numeric digits |
| Checksum | Validated using the Brazilian CPF algorithm (mod-11 check digits) |
| Sanitization | Non-numeric characters are stripped before validation |
| Uniqueness | Each CPF value must be unique per tenant |
| Storage | `simpleValue` (digits only) |

### EMAIL

| Rule | Detail |
|------|--------|
| Format | RFC 5322 compliant email address |
| Storage | `simpleValue` |

### PHONE

| Rule | Detail |
|------|--------|
| Format | Phone number string (e.g., `"+5511999999999"`) |
| Extra phones | Supports `extraPhones` array for additional numbers |
| Storage | `simpleValue` (single) or `complexValue` (with extras) |

### BIRTHDATE

| Rule | Detail |
|------|--------|
| Format | ISO date string (e.g., `"1990-05-15"`) |
| Minimum age | Default minimum age is 18 years; configurable per input |
| Storage | `simpleValue` |

### SIMPLE_SELECT

| Rule | Detail |
|------|--------|
| Format | String matching one of the `options[].value` from the input definition |
| Multiple | If `isMultiple` is true, value can be an array of option values |
| Storage | `simpleValue` |

### SIMPLE_LOCATION

| Rule | Detail |
|------|--------|
| Required fields | `placeId`, `region`, `country` |
| Optional fields | `city`, `postal_code`, `street_address_1`, `street_address_2`, `latitude`, `longitude` |
| Storage | `complexValue` |

Example:
```json
{
  "placeId": "ChIJrTLr-GyuEmsRBfy61i59si0",
  "region": "SP",
  "country": "BR",
  "city": "Sao Paulo",
  "postal_code": "01000-000",
  "street_address_1": "Av Paulista 1000"
}
```

### IDENTIFICATION_DOCUMENT

| Rule | Detail |
|------|--------|
| Required fields | `docType` (one of: `passport`, `cpf`, `rg`), `document` (document number) |
| File attachment | Optionally include `assetId` for document photo |
| Storage | `complexValue` + optional `assetId` |

Example:
```json
{
  "docType": "rg",
  "document": "123456789"
}
```

### FILE / IMAGE

| Rule | Detail |
|------|--------|
| Requirement | Must provide `assetId` from a prior Cloudinary upload |
| File validation | Asset must exist and be accessible |
| Storage | `assetId` |

### MULTIFACE_SELFIE

| Rule | Detail |
|------|--------|
| Requirement | Must provide `assetId` from a prior Cloudinary upload |
| Prerequisites | User must have submitted CPF and birthdate documents BEFORE the selfie |
| Processing | Triggers biometric validation service call |
| Storage | `assetId` |

### COMMERCE_PRODUCT

| Rule | Detail |
|------|--------|
| Required fields | `productId`, `quantity` |
| Validation | Product must exist in the commerce service |
| Storage | `complexValue` |

Example:
```json
{
  "productId": "product-uuid",
  "quantity": "1",
  "variantIds": ["variant-uuid-1"]
}
```

---

## Error Reference

| Status | Exception | Cause | Resolution |
|--------|-----------|-------|------------|
| 400 | `BadRequestException` | Missing required document or invalid value format | Check field requirements and value format per input type |
| 400 | `CPfDuplicatedException` | CPF already used by another user in this tenant | Each CPF must be unique per tenant; check existing submissions |
| 400 | `InvalidCPFException` | CPF checksum validation failed | Verify CPF is exactly 11 digits with valid mod-11 check digits |
| 400 | `MaxSubmissionReachedException` | User exceeded max submissions for this context | Context's `maxSubmissions` limit reached; cannot create new submissions |
| 400 | `TenantContextDisabledException` | Context is inactive | Contact admin to activate the context |
| 400 | `BadRequestException` (approve) | Context not in approvable state | Context must be `CREATED` or `REQUIRED_REVIEW` to approve/reject |
| 400 | `BadRequestException` (review) | No `inputIds` provided for require-review | At least one inputId is required |
| 403 | `NotAuthorizedAttachmentDocumentToContextException` | User cannot submit to this context | Check user permissions and context access rules |
| 403 | `UserNeedVerifyAccount` | Email not verified | Complete email verification first (unless passwordless) |
| 403 | `ForbiddenException` | Approver not authorized for this context | Check whitelist membership and specific approver assignment |
| 404 | `UserContextNotFoundException` | User context not found | Verify userId, contextId, userContextId, and tenantId |
| 500 | `ItWasNotPossibleRegisterMutiplaceExpection` | Biometric validation service failed | Retry with a clearer selfie photo; ensure CPF and birthdate are submitted |
