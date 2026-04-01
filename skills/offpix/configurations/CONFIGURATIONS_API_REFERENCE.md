---
id: CONFIGURATIONS_API_REFERENCE
title: "Configurations - API Reference"
module: offpix/configurations
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - contexts
  - tenant-context
  - tenant-input
  - api-reference
---

# Configurations API Reference

Complete endpoint reference for the W3Block Configurations module. This module spans three related resources — Contexts (form definitions), Tenant Contexts (tenant-specific configurations of those forms), and Tenant Inputs (individual form fields) — all served by the PixwayID backend.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://pixwayid.w3block.io` |
| Staging | `https://pixwayid.stg.w3block.io` |
| Swagger | https://pixwayid.w3block.io/docs/ |

## Authentication

Endpoints marked **Auth: Bearer** require:

```
Authorization: Bearer {accessToken}
```

Most endpoints require Admin or SuperAdmin roles. Public endpoints (slug-based lookups) are noted individually.

---

## Enums

### ContextType

| Value | Description |
|-------|-------------|
| `user_properties` | KYC / user profile data collection form |
| `form` | Generic form (surveys, custom data collection) |

### DataTypesEnum (Input Field Types)

| Value | Category | Description |
|-------|----------|-------------|
| `text` | Input | Free text field |
| `email` | Input | Email address with validation |
| `phone` | Input | Phone number |
| `cpf` | Input | Brazilian CPF document number |
| `url` | Input | URL with validation |
| `date` | Input | Date picker |
| `birthdate` | Input | Birth date (may include age validation) |
| `user_name` | Input | User display name |
| `file` | Upload | Generic file upload |
| `image` | Upload | Image file upload |
| `identification_document` | Upload | Government ID (front/back) |
| `multiface_selfie` | Upload | Biometric selfie with document |
| `simple_select` | Select | Dropdown with static options |
| `dynamic_select` | Select | Dropdown with dynamic options |
| `checkbox` | Toggle | Boolean checkbox |
| `simple_location` | Location | Address / location input |
| `commerce_product` | Commerce | Product selector (requires `data.productIds`) |
| `iframe` | Embed | Embedded iframe content |
| `separator` | Layout | Visual separator (not a data field, skipped in validation) |

### TenantContextNotificationType

| Value | Description |
|-------|-------------|
| `kyc_approval_request` | Notification sent when a user submits KYC for approval |
| `kyc_require_user_review` | Notification asking user to review/resubmit |
| `kyc_user_approved` | Notification sent to user on approval |
| `kyc_admin_approved` | Notification sent to admin on approval |
| `kyc_user_rejected` | Notification sent to user on rejection |
| `kyc_admin_rejected` | Notification sent to admin on rejection |

---

## Entities & Relationships

```
Context (1) ──→ (N) TenantContext ──→ (N) TenantInput
   │                     │
   │                     └── Links a Context to a Tenant with config (approval, notifications)
   │
   └── Master form definition (slug, type, maxSubmissions)

TenantInput: Individual form field within a TenantContext (label, type, order, mandatory)
```

**Key concepts:**
- A **Context** is a reusable form template (e.g., "signup", "kyc-verification")
- A **Tenant Context** binds a Context to a specific tenant, adding config like auto-approval, screen layout, and notification settings
- **Tenant Inputs** are the individual fields/questions inside a form, ordered by `order` and optionally grouped by `step`
- Contexts can be **global** (`tenantId = null`) or **tenant-specific**

---

## Endpoints: Contexts (SuperAdmin only)

Base path: `/contexts`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/contexts` | Bearer | SuperAdmin | Create a new context definition |
| 2 | GET | `/contexts` | Bearer | SuperAdmin, Admin | List all contexts (global + tenant) |
| 3 | PATCH | `/contexts/:id` | Bearer | SuperAdmin | Update a context |
| 4 | DELETE | `/contexts/:id` | Bearer | SuperAdmin | Soft-delete a context |

### POST /contexts

Create a new master context definition.

**Request:**
```json
{
  "description": "KYC Verification",
  "slug": "kyc-verification",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `description` | string | Yes | — | Human-readable name |
| `slug` | string | Yes | — | URL-safe identifier. Lowercase, alphanumeric + hyphens. Must match `^[a-z0-9]+(?:-[a-z0-9]+)*$` |
| `type` | ContextType | No | `user_properties` | Form type |
| `maxSubmissions` | integer | No | `1` | Max times a user can submit this form (min: 1) |
| `isPreOrder` | boolean | No | `false` | Whether this is a pre-order context |
| `tenantId` | UUID | No | `null` | If null, context is global |

**Response (201):**
```json
{
  "id": "ctx-uuid",
  "description": "KYC Verification",
  "slug": "kyc-verification",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "tenantId": null,
  "createdAt": "2026-03-31T10:00:00Z",
  "updatedAt": "2026-03-31T10:00:00Z"
}
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 409 | DuplicatedContextException | Slug + tenantId combination already exists |

---

## Endpoints: Tenant Contexts

Base path: `/tenants/:tenantId/tenant-context`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/tenants/:tenantId/tenant-context` | Bearer | Admin, Integration | Create tenant-context association |
| 2 | GET | `/tenants/:tenantId/tenant-context` | Bearer | Admin, User | List tenant contexts (paginated, filterable) |
| 3 | GET | `/tenants/:tenantId/tenant-context/:tenantContextId` | Bearer | Admin, User | Get single tenant context |
| 4 | GET | `/tenants/:tenantId/tenant-context/slug/:slug` | Bearer | Admin, User | Get tenant context by slug |
| 5 | PATCH | `/tenants/:tenantId/tenant-context/:tenantContextId` | Bearer | Admin | Update tenant context |
| 6 | GET | `/tenants/:tenantId/tenant-context/:tenantContextId/approvers` | Bearer | Admin, User | List KYC approvers for context |

### POST /tenants/:tenantId/tenant-context

Link an existing context to a tenant.

**Minimal Request:**
```json
{
  "contextId": "ctx-uuid",
  "active": true
}
```

**Complete Request:**
```json
{
  "contextId": "ctx-uuid",
  "active": true,
  "data": {
    "requireSpecificApprover": true,
    "alternativeSignUp": false,
    "screenConfig": {
      "postKycUrl": "https://example.com/success",
      "skipConfirmation": false,
      "position": "center",
      "textContainer": {
        "mainText": "Complete your profile",
        "subtitleText": "Enter your information",
        "auxiliarText": "This is required"
      }
    },
    "profileScreen": {
      "hidden": false
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contextId` | UUID | Yes | ID of the context to associate |
| `active` | boolean | Yes | Whether this form is currently active |
| `data` | object | No | Tenant-specific configuration (screen layout, approval rules, etc.) |

**Response (201):** `TenantContextDto` (see response shape below)

**Note:** If a tenant-context for the same `contextId + tenantId` already exists, this endpoint **updates** it instead of creating a duplicate.

### GET /tenants/:tenantId/tenant-context

List all tenant contexts with pagination and filtering.

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 10 | Items per page |
| `search` | string | — | Search in description/slug |
| `sortBy` | string | `createdAt` | Sort column |
| `orderBy` | `ASC` \| `DESC` | `DESC` | Sort direction |
| `active` | boolean | — | Filter by active status |
| `type` | ContextType[] | — | Filter by context type |
| `preOrder` | boolean | — | Filter by isPreOrder |

**Response (200):**
```json
{
  "items": [
    {
      "id": "tc-uuid",
      "contextId": "ctx-uuid",
      "tenantId": "tenant-uuid",
      "active": true,
      "data": { "..." : "..." },
      "autoApprove": false,
      "requireSpecificApprover": false,
      "blockCommerceDeliver": false,
      "approverWhitelistIds": null,
      "context": {
        "id": "ctx-uuid",
        "slug": "signup",
        "description": "SignUp Form",
        "type": "user_properties",
        "maxSubmissions": 1,
        "isPreOrder": false
      },
      "createdAt": "2026-03-31T10:00:00Z",
      "updatedAt": "2026-03-31T10:00:00Z"
    }
  ],
  "meta": {
    "totalItems": 1,
    "totalPages": 1,
    "currentPage": 1,
    "itemsPerPage": 10
  }
}
```

### GET /tenants/:tenantId/tenant-context/slug/:slug

Fetch a tenant context by the underlying context slug. Useful for well-known forms like `signup` or `admin-notification-settings`.

**Response (200):** Single `TenantContextDto` with nested `context`.

### GET /tenants/:tenantId/tenant-context/:tenantContextId/approvers

Returns the list of users with `kycApprover` role for this context. Cached for 300 seconds.

**Response (200):**
```json
{
  "approvers": [
    {
      "id": "user-uuid",
      "email": "approver@example.com",
      "name": "John Doe"
    }
  ]
}
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 404 | TenantContextNotFoundException | Context not found |
| 400 | TenantContextDisabledException | Context is not active |
| 400 | TenantContextCantHaveApproversException | Context has `autoApprove = true` |

---

## Endpoints: Admin Tenant Contexts

Base path: `/tenants/:tenantId/admin/tenant-context`

These endpoints create/update both the **context** and the **tenant-context** atomically in a single transaction.

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/tenants/:tenantId/admin/tenant-context` | Bearer | Admin, Integration | Create context + tenant-context in one step |
| 2 | PATCH | `/tenants/:tenantId/admin/tenant-context/:tenantContextId` | Bearer | Admin | Update context + tenant-context atomically |
| 3 | GET | `/tenants/:tenantId/admin/tenant-context/:tenantContextId/inputs` | Bearer | Admin | List inputs for a tenant context |

### POST /tenants/:tenantId/admin/tenant-context

Creates both the underlying Context and the TenantContext in a single transaction. This is the endpoint the frontend uses when creating new forms.

**Minimal Request:**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "active": true
}
```

**Complete Request (production example from frontend):**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": true,
  "data": {
    "requireSpecificApprover": true,
    "alternativeSignUp": false,
    "profileScreen": {
      "hidden": false
    },
    "screenConfig": {
      "postKycUrl": "https://example.com/verify-success",
      "skipConfirmation": false,
      "position": "left",
      "textContainer": {
        "mainText": "Verify Your Identity",
        "subtitleText": "Upload your documents",
        "auxiliarText": "This step is mandatory"
      }
    }
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `slug` | string | Yes | — | URL-safe identifier (lowercase, alphanumeric + hyphens) |
| `description` | string | Yes | — | Human-readable form name |
| `type` | ContextType | No | `user_properties` | Form type |
| `maxSubmissions` | integer | No | `1` | Max user submissions |
| `isPreOrder` | boolean | No | `false` | Pre-order flag |
| `active` | boolean | Yes | — | Whether form is active |
| `data` | object | No | `null` | Screen config, approval rules (see data schema below) |

**`data` object schema (frontend naming → API field):**

| Frontend Name | API Path | Type | Description |
|---------------|----------|------|-------------|
| Require Specific Approver | `data.requireSpecificApprover` | boolean | Require a specific KYC approver |
| Alternative Sign Up | `data.alternativeSignUp` | boolean | Enable alternative signup flow |
| Hide Profile Screen | `data.profileScreen.hidden` | boolean | Skip profile step in the form |
| Redirect URL (Post-KYC) | `data.screenConfig.postKycUrl` | string | URL to redirect after form completion |
| Skip Confirmation | `data.screenConfig.skipConfirmation` | boolean | Skip confirmation step |
| Form Position | `data.screenConfig.position` | `"left"` \| `"center"` | Form alignment on screen |
| Main Text | `data.screenConfig.textContainer.mainText` | string | Form heading |
| Subtitle | `data.screenConfig.textContainer.subtitleText` | string | Form subheading |
| Helper Text | `data.screenConfig.textContainer.auxiliarText` | string | Additional instructions |

### PATCH /tenants/:tenantId/admin/tenant-context/:tenantContextId

Same body as POST (except `type` cannot be changed). Updates both context and tenant-context atomically.

**Note:** Only SuperAdmin can update **global** contexts (tenantId = null). Attempting to update a global context as Admin throws `UnableToChangeGlobalContextException` (403).

### GET /tenants/:tenantId/admin/tenant-context/:tenantContextId/inputs

List all inputs belonging to this tenant context.

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 10 | Items per page |
| `search` | string | — | Search in label/description |

**Response (200):** Paginated list of `TenantInputEntityDto`.

---

## Endpoints: Tenant Inputs (Form Fields)

Base path: `/tenants/:tenantId/tenant-input`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/tenants/:tenantId/tenant-input` | Bearer | SuperAdmin, Admin, Integration | Create a new form field |
| 2 | PATCH | `/tenants/:tenantId/tenant-input/:inputId` | Bearer | SuperAdmin, Admin, Integration | Update a form field |
| 3 | GET | `/tenants/:tenantId/tenant-input/slug/:slug` | **None (Public)** | — | Get active inputs by context slug |
| 4 | GET | `/tenants/:tenantId/tenant-input` | Bearer | SuperAdmin, Admin, Integration | List inputs (paginated) |
| 5 | GET | `/tenants/:tenantId/tenant-input/:inputId` | Bearer | SuperAdmin, Admin, Integration | Get single input |

### POST /tenants/:tenantId/tenant-input

Create a new form field within a context.

**Minimal Request:**
```json
{
  "contextId": "ctx-uuid",
  "label": "Email Address",
  "description": "Enter your email",
  "type": "email",
  "order": 1,
  "mandatory": true,
  "active": true
}
```

**Complete Request (production example):**
```json
{
  "contextId": "ctx-uuid",
  "label": "Email Address",
  "description": "Enter your email address",
  "type": "email",
  "order": 1,
  "mandatory": true,
  "active": true,
  "attributeName": "userEmail",
  "step": 1,
  "options": null,
  "data": null
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `contextId` | UUID | Yes | — | Context this input belongs to |
| `label` | string | Yes | — | Field label shown to the user |
| `description` | string | Yes | — | Field help text / placeholder |
| `type` | DataTypesEnum | Yes | — | Field type (see enum above) |
| `order` | integer | Yes | — | Display order (ascending) |
| `mandatory` | boolean | Yes | — | Whether this field is required |
| `active` | boolean | Yes | — | Whether this field is currently shown |
| `attributeName` | string | No | Same as `type` | Custom attribute key for the submitted value |
| `step` | number | No | `null` | Form step/page number for multi-step forms |
| `options` | array | No | `[]` | For `simple_select` / `dynamic_select`: `[{ "label": "Option A", "value": "a" }]` |
| `data` | object | No | `null` | Type-specific config (e.g., `{ "productIds": ["uuid"] }` for `commerce_product`) |

**Side effects:** If the new input is `mandatory` and `active`, all existing users are flagged for review via the users-contexts system.

### PATCH /tenants/:tenantId/tenant-input/:inputId

Update an existing form field. Cannot change `contextId` or `type`.

**Side effects:** Same as POST — if `mandatory` or `active` changes to `mandatory + active`, users are flagged for review.

### GET /tenants/:tenantId/tenant-input/slug/:slug (Public)

Returns all **active** inputs for a context identified by slug, ordered by `order ASC`. This is the endpoint used to render forms to end users.

**Response (200):**
```json
[
  {
    "id": "input-uuid-1",
    "contextId": "ctx-uuid",
    "tenantId": "tenant-uuid",
    "label": "Full Name",
    "description": "Enter your full name",
    "type": "text",
    "order": 1,
    "mandatory": true,
    "active": true,
    "attributeName": "fullName",
    "step": 1,
    "options": [],
    "data": null,
    "createdAt": "2026-03-31T10:00:00Z",
    "updatedAt": "2026-03-31T10:00:00Z"
  },
  {
    "id": "input-uuid-2",
    "contextId": "ctx-uuid",
    "tenantId": "tenant-uuid",
    "label": "Email Address",
    "description": "Enter your email",
    "type": "email",
    "order": 2,
    "mandatory": true,
    "active": true,
    "attributeName": "email",
    "step": 1,
    "options": [],
    "data": null,
    "createdAt": "2026-03-31T10:00:00Z",
    "updatedAt": "2026-03-31T10:00:00Z"
  }
]
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 404 | ContextBySlugNotFoundException | No context with this slug |
| 404 | TenantContextNotFoundException | Context exists but not linked to this tenant |
| 400 | TenantContextDisabledException | Tenant context is inactive |

---

## Type-Specific Validation: commerce_product

When creating/updating an input with `type: "commerce_product"`, the `data` field is validated:

```json
{
  "data": {
    "productIds": ["product-uuid-1", "product-uuid-2"]
  }
}
```

- `productIds` must be a non-empty array of valid UUIDs
- Each product ID is verified against the Commerce service — invalid IDs throw `InvalidTenantDataException`
