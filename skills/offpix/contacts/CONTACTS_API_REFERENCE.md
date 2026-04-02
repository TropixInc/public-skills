---
id: CONTACTS_API_REFERENCE
title: "Contacts - API Reference"
module: offpix/contacts
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contacts
  - users
  - kyc
  - documents
  - royalty
  - api-reference
---

# Contacts API Reference

Complete endpoint reference for the W3Block Contacts module. This module covers user management (invite, edit, list), KYC document submission and approval workflows, royalty eligibility, and external contacts. Served by the PixwayID backend with some endpoints on the Commerce/Registry backends.

## Base URLs

| Environment | Service | URL |
|-------------|---------|-----|
| Production | PixwayID (users, KYC) | `https://pixwayid.w3block.io` |
| Production | Registry (royalty, external contacts) | `https://api.w3block.io` |
| Production | Commerce (payment providers) | `https://commerce.w3block.io` |
| Staging | PixwayID | *(staging environment available — use staging base URL)* |
| Swagger | PixwayID | https://pixwayid.w3block.io/docs/ |
| Swagger | Registry | https://api.w3block.io/docs/ |

## Authentication

All endpoints require Bearer token unless explicitly marked as public:

```
Authorization: Bearer {accessToken}
```

---

## Enums

### UserRoleEnum

| Value | Frontend Label | Description |
|-------|---------------|-------------|
| `superAdmin` | — | Platform-level super administrator |
| `admin` | Admin | Tenant administrator |
| `operator` | Operator | Tenant operator |
| `user` | Client | Regular user / client |
| `loyaltyOperator` | Loyalty Operator | Loyalty program operator |
| `commerceOrderReceiver` | ERC Receiver | Commerce order receiver |
| `kycApprover` | — | KYC approval operator |
| `keyErc20Receiver` | — | ERC-20 token receiver |

### UserContextStatus (KYC Status)

| Value | Frontend Label | Color | Description |
|-------|---------------|-------|-------------|
| `APPROVED` | Approved | Green | All documents approved |
| `DENIED` | Denied | Red | Documents rejected |
| `REQUIRED_REVIEW` | Required Review | Orange | Approver requested resubmission of specific fields |
| `CREATED` | Created | Orange | All docs submitted, awaiting approval |
| `DRAFT` | Draft | — | Submission in progress (incomplete) |

### KYCStatusType (Frontend-specific, from customer-infos)

| Value | Color | Description |
|-------|-------|-------------|
| `approved` | Green | KYC approved |
| `denied` | Red | KYC denied |
| `waitingInfo` | Orange | Waiting for info |
| `missingDocuments` | Orange | Missing required documents |
| `processing` | Orange | Processing |
| `created` | Orange | Awaiting review |

### ContactTypes (Frontend navigation)

| Value | Frontend Route | Filtered Roles |
|-------|---------------|----------------|
| `clients` | `/dash/contacts/clients` | `user` |
| `team` | `/dash/contacts/team` | `admin`, `operator`, `loyaltyOperator`, `commerceOrderReceiver` |
| `partiner` | `/dash/contacts/partiners` | Partners/associates |
| `external` | — | External wallet contacts (no user account) |

---

## Entities & Relationships

```
User (1) ──→ (N) UsersContext ──→ (N) UsersDocument
  │                  │                     │
  │ email, name,     │ status, logs,       │ inputId, value,
  │ phone, roles,    │ approverUserId,     │ assetId, status
  │ wallets          │ utmParams           │
  │                  │                     │
  │                  └── Links user to context with approval status
  │                       (DRAFT → CREATED → APPROVED/DENIED)
  │
  ├──→ (N) Wallet (mainWallet + additional)
  └──→ (1) RoyaltyEligible (optional)

TenantContext defines the form → TenantInput defines the fields
UsersContext tracks a user's submission → UsersDocument stores each field value
```

---

## Endpoints: User Management

### GET /users — List Users by Tenant

**Path:** `GET /{tenantId}/users` (via SDK: `sdk.api.users.getUsersByTenantId`)
**Auth:** Bearer (Admin)

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `role` | UserRoleEnum[] | `['user']` | Filter by role(s) |
| `orderBy` | `ASC` \| `DESC` | `DESC` | Sort direction |
| `sortBy` | string | `createdAt` | Sort column |
| `search` | string | — | Search in email/name |
| `limit` | integer | `10` | Items per page |
| `page` | integer | `1` | Page number |

**Response (200):**
```json
{
  "items": [
    {
      "id": "user-uuid",
      "tenantId": "tenant-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+5511999999999",
      "roles": ["user"],
      "verified": true,
      "verifiedEmailAt": "2026-01-01T00:00:00Z",
      "mainWalletId": "wallet-uuid",
      "mainWallet": {
        "id": "wallet-uuid",
        "address": "0x1234...abcd",
        "type": "vault",
        "status": "active"
      },
      "wallets": [],
      "createdAt": "2026-01-01T00:00:00Z",
      "updatedAt": "2026-01-01T00:00:00Z"
    }
  ],
  "meta": {
    "totalItems": 100,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 10,
    "currentPage": 1
  }
}
```

### GET /users/:id — Get User Profile

**Path:** `GET /{tenantId}/users/{userId}` (via SDK: `sdk.api.users.getProfileUserById`)
**Auth:** Bearer (Admin, User — user can only fetch own profile)

**Response (200):** Same as list item shape, with full wallet details.

### POST /users/invite — Invite New User

**Path:** `POST /users/invite` (via SDK: `sdk.api.users.invite`)
**Auth:** Bearer (Admin)

**Request:**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "user",
  "royaltyEligible": false,
  "tenantId": "tenant-uuid",
  "i18nLocale": "pt-br",
  "generateRandomPassword": true,
  "sendEmail": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | User's full name |
| `email` | string | Yes | User's email (must be unique per tenant) |
| `role` | UserRoleEnum | Yes | Role to assign |
| `tenantId` | UUID | Yes | Target tenant |
| `i18nLocale` | string | No | Locale for emails (`pt-br`, `en`) |
| `generateRandomPassword` | boolean | Yes | Always `true` — generates temp password |
| `sendEmail` | boolean | Yes | Always `true` — sends invitation email |

**Response (201):** User object with `id`.

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 409 | Conflict | Email already in use for this tenant |

### PATCH /users/:id — Edit User

**Path:** `PATCH /{tenantId}/users/{userId}` (via SDK: `sdk.api.users.update`)
**Auth:** Bearer (Admin)

**Request:**
```json
{
  "name": "Updated Name",
  "phone": "+5511888888888",
  "roles": ["user", "operator"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Updated name |
| `phone` | string | No | Updated phone |
| `roles` | UserRoleEnum[] | No | Updated roles |

### GET /users/find-by-any — Find User by CPF or ID

**Path:** `GET /{tenantId}/users/documents/find-user-by-any`
**Auth:** Bearer (Admin, Integration)

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `userId` | UUID | Find by user ID |
| `cpf` | string | Find by CPF (11 digits, valid checksum) |

One of `userId` or `cpf` is required. CPF lookup searches in submitted KYC documents.

### GET /users/by-wallet — Find User by Wallet Address

**Path:** `GET /{tenantId}/users/wallets/by-address/{address}` (via SDK)
**Auth:** Bearer (Admin)

---

## Endpoints: KYC User Contexts

Base path: `/{tenantId}/users/contexts`

### GET /users/contexts/find — List All User Context Submissions

**Auth:** Bearer (Admin, User, KycApprover)

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | `1` | Page number |
| `limit` | integer | `10` | Items per page |
| `search` | string | — | Search in user email/name |
| `sortBy` | array | `[{column: 'createdAt', order: 'ASC'}]` | Sort config |
| `userId[]` | UUID[] | — | Filter by user IDs (Admin only) |
| `status[]` | UserContextStatus[] | — | Filter by status |
| `contextId[]` | UUID[] | — | Filter by context IDs |
| `contextType[]` | ContextType[] | — | Filter by `user_properties` or `form` |
| `preOrder` | boolean | — | Filter pre-order contexts |
| `utmCampaign` | string | — | Filter by UTM campaign |
| `utmSource` | string | — | Filter by UTM source |
| `excludeSelfContexts` | boolean | — | KycApprover: exclude own submissions |

**Response (200):** Paginated list of `UsersContextsEntity` with user and context relations.

### GET /users/contexts/:userId — Get User's Contexts

**Auth:** Bearer (Admin, User — user can only see own)

**Query Parameters:** `page`, `limit`, `search`, `sortBy`, `status`, `contextId`

### GET /users/contexts/:userId/:userContextId — Get Specific Context with Documents

**Auth:** Bearer (Admin, User, KycApprover)

**Response (200):**
```json
{
  "id": "uc-uuid",
  "tenantId": "tenant-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "CREATED",
  "approverUserId": null,
  "logs": [
    {
      "moderatorId": "admin-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    }
  ],
  "utmParams": null,
  "documents": [
    {
      "id": "doc-uuid",
      "inputId": "input-uuid",
      "status": "CREATED",
      "simpleValue": "john@example.com",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid",
        "label": "Email Address",
        "type": "email",
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

## Endpoints: KYC Document Submission

### POST /users/documents/:userId/context/:contextId — Submit Documents

**Path:** `POST /{tenantId}/users/documents/{userId}/context/{contextId}`
**Auth:** Bearer (User, Admin, KycApprover)

**Request:**
```json
{
  "documents": [
    {
      "inputId": "input-uuid-name",
      "value": "John Doe"
    },
    {
      "inputId": "input-uuid-email",
      "value": "john@example.com"
    },
    {
      "inputId": "input-uuid-cpf",
      "value": "12345678901"
    },
    {
      "inputId": "input-uuid-file",
      "assetId": "asset-uuid"
    },
    {
      "inputId": "input-uuid-location",
      "value": {
        "region": "SP",
        "country": "BR",
        "placeId": "ChIJ...",
        "city": "Sao Paulo",
        "postal_code": "01000-000",
        "street_address_1": "Av Paulista"
      }
    }
  ],
  "currentStep": 1,
  "approverUserId": "approver-uuid",
  "userContextId": "existing-uc-uuid",
  "utmParams": {
    "utm_campaign": "onboarding",
    "utm_source": "email"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `documents` | DocumentDto[] | Yes (min: 1) | Array of field submissions |
| `documents[].inputId` | UUID | Yes | TenantInput ID this value belongs to |
| `documents[].value` | string \| object | Conditional | Field value (text, object for complex types) |
| `documents[].assetId` | UUID | Conditional | File asset ID (for file/image/document uploads) |
| `currentStep` | number | No | Current form step (for multi-step validation) |
| `approverUserId` | UUID | Conditional | Required if context has `requireSpecificApprover: true` |
| `userContextId` | UUID | No | Attach to existing user context (for resubmission) |
| `utmParams` | object | No | UTM tracking parameters |

**Response (201):** `UsersContextsEntity` (created or updated).

**Value formats by input type:**

| Input Type | Value Format | Example |
|------------|-------------|---------|
| `text`, `email`, `url`, `user_name` | string | `"john@example.com"` |
| `cpf` | string (11 digits) | `"12345678901"` |
| `phone` | string | `"+5511999999999"` |
| `date`, `birthdate` | ISO date string | `"1990-05-15"` |
| `simple_select` | string (must be in options) | `"BR"` |
| `checkbox` | boolean | `true` |
| `file`, `image`, `multiface_selfie` | — (use `assetId`) | — |
| `identification_document` | object | `{"docType": "RG", "document": "123456789"}` |
| `simple_location` | object | `{"region": "SP", "country": "BR", "placeId": "..."}` |
| `commerce_product` | object | `{"productId": "uuid", "quantity": "1"}` |
| `separator` | — (skipped) | — |

### GET /users/documents/:userId — Get User's Documents

**Path:** `GET /{tenantId}/users/documents/{userId}`
**Auth:** Bearer (Admin, User)

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number |
| `limit` | integer | Items per page |
| `type[]` | DataTypesEnum[] | Filter by input types |
| `contextId` | UUID | Filter by context |
| `inputId` | UUID | Filter by specific input |

### GET /users/documents/:userId/context/:contextId — Get Documents for Context

**Path:** `GET /{tenantId}/users/documents/{userId}/context/{contextId}`
**Auth:** Bearer (Admin, User)

Returns all documents for a specific context (non-paginated array).

---

## Endpoints: KYC Approval Actions

### PATCH /users/contexts/:userId/:contextId/approve

**Auth:** Bearer (Admin, KycApprover)

**Request:**
```json
{
  "reason": "Documents verified successfully"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Approval reason (logged in audit trail) |
| `userContextId` | UUID | No | Specific user context to approve |

**Response:** 204 No Content

**Side effects:**
- All documents set to `APPROVED`
- User context status → `APPROVED`
- Log entry added with moderator ID, timestamp, reason
- KYC approval email sent to user
- Commerce service notified (may unblock orders)
- Webhook `USER_KYC_PROPS_UPDATED` dispatched

### PATCH /users/contexts/:userId/:contextId/reject

**Auth:** Bearer (Admin, KycApprover)

**Request:**
```json
{
  "reason": "Document image is blurry, please resubmit"
}
```

**Response:** 204 No Content

**Side effects:**
- All documents set to `DENIED`
- User context status → `DENIED`
- KYC rejection email sent
- Commerce service notified

### PATCH /users/contexts/:userId/:contextId/require-review

**Auth:** Bearer (Admin, KycApprover)

**Request:**
```json
{
  "inputIds": ["input-uuid-1", "input-uuid-2"],
  "reason": "ID document is expired, selfie is blurry"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inputIds` | UUID[] | Yes (min: 1) | Input IDs that need resubmission |
| `reason` | string | No | Explanation for user |

**Response:** 204 No Content

**Side effects:**
- Specified documents set to `REQUIRED_REVIEW`
- User context status → `REQUIRED_REVIEW`
- KYC review request email sent
- User can resubmit only the flagged fields

---

## Endpoints: Customer Infos (KYC List)

### GET /customer-infos/:tenantId/search — List KYC Submissions

**Auth:** Bearer (Admin)

**Query Parameters:** `page`, `limit`, `sortBy`, `orderBy`

**Response (200):**
```json
{
  "items": [
    {
      "id": "ci-uuid",
      "customerId": "user-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "status": "approved",
      "tenantId": "tenant-uuid",
      "createdAt": "2026-01-15T10:00:00Z",
      "updatedAt": "2026-01-15T10:00:00Z"
    }
  ]
}
```

---

## Endpoints: Royalty Eligible

Base path: `/{companyId}/contracts/royalty-eligible` (Registry backend)

### GET / — List Royalty Eligible Contacts

**Auth:** Bearer (Admin)

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `userIds` | string | — | Comma-separated user IDs |
| `externalContactIds` | string | — | Comma-separated external contact IDs |
| `wallets` | boolean | `true` | Include wallet info |
| `search` | string | — | Search |
| `limit` | integer | `50` | Items per page |

**Response (200):**
```json
{
  "items": [
    {
      "id": "re-uuid",
      "companyId": "company-uuid",
      "active": true,
      "displayName": "John Doe",
      "userId": "user-uuid",
      "externalContactId": "ext-uuid",
      "walletAddress": "0x1234...abcd",
      "createdAt": "2026-01-01T00:00:00Z",
      "updatedAt": "2026-01-01T00:00:00Z"
    }
  ]
}
```

### POST /create — Create Royalty Eligible Record

**Auth:** Bearer (Admin)

**Request:**
```json
{
  "active": true,
  "displayName": "John Doe",
  "userId": "user-uuid"
}
```

### PATCH / — Update Royalty Eligible

**Auth:** Bearer (Admin)

**Request:**
```json
{
  "active": false,
  "userId": "user-uuid",
  "externalContactId": "ext-uuid"
}
```

---

## Endpoints: External Contacts

Base path: `/{companyId}/external-contacts` (Registry backend)

### GET / — List External Contacts

**Auth:** Bearer (Admin)

### POST /import — Create/Import External Contact

**Auth:** Bearer (Admin)

**Request:**
```json
{
  "name": "External Partner",
  "walletAddress": "0xabcd...1234",
  "email": "partner@example.com",
  "phone": "+5511999999999",
  "royaltyEligible": true,
  "description": "Royalty partner"
}
```

---

## Endpoints: Payment Providers (per User)

Base path: `/companies/{companyId}/users/{userId}/providers` (Commerce backend)

### GET / — List User's Payment Providers

**Auth:** Bearer (Admin)

Returns configured payment providers (Stripe, ASAAS, etc.) for a user.

### DELETE /:provider — Delete Payment Provider

**Auth:** Bearer (Admin)

Removes a specific payment provider configuration from a user.

---

## Referral System

### PATCH /users/:companyId/:userId/referrer — Edit Referral

**Path:** `PATCH /users/{companyId}/{userId}/referrer`
**Auth:** Bearer (Admin)

**Request:**
```json
{
  "referrerCode": "ABC123",
  "referrerUserId": "referrer-uuid"
}
```

Links a user to a referrer via referral code or user ID.
