---
id: FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
title: "Configurations - Context Lifecycle"
module: offpix/configurations
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - contexts
  - tenant-context
  - forms
  - kyc
depends_on:
  - AUTH_API_REFERENCE
  - CONFIGURATIONS_API_REFERENCE
---

# Context Lifecycle

## Overview

A Context in W3Block represents a form definition (KYC, surveys, onboarding forms). This flow covers creating, configuring, and managing contexts through the admin panel. The full lifecycle is: create a context + tenant association (single API call), then add form fields (inputs) to it.

In the frontend, this module is accessed under **KYC > Form Designer** (`/dash/kyc/form-designer`).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT access token with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Admin role | User must have Admin or SuperAdmin role | Tenant setup |

## Entities & Relationships

```
Context (1) ──→ (N) TenantContext ──→ (N) TenantInput
  │                     │                    │
  │ slug, type,         │ active, data,      │ label, type, order,
  │ maxSubmissions      │ approval config,   │ mandatory, options,
  │                     │ notifications      │ step, attributeName
  │                     │
  │                     └── Links context to tenant with screen config
  │
  └── Master form definition (can be global or tenant-specific)
```

**Context types:**
- `user_properties` — KYC / user profile forms (shown in frontend as "onboarding" forms)
- `form` — Generic forms (surveys, custom data collection)

---

## Flow A: Create a New Form (Context + Tenant Association)

The admin endpoint creates both the Context and the TenantContext in a single transaction. This is the recommended approach.

### Step 1: Create Context via Admin Endpoint

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/tenants/{tenantId}/admin/tenant-context` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "active": true
}
```

**Complete Request (production example):**
```json
{
  "slug": "kyc-verification",
  "description": "KYC Verification Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": true,
  "data": {
    "requireSpecificApprover": false,
    "alternativeSignUp": false,
    "profileScreen": {
      "hidden": false
    },
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": false,
      "position": "center",
      "textContainer": {
        "mainText": "Verify Your Identity",
        "subtitleText": "Upload your documents to continue",
        "auxiliarText": "This process takes about 5 minutes"
      }
    }
  }
}
```

**Response (201):**
```json
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
    "slug": "kyc-verification",
    "description": "KYC Verification Form",
    "type": "user_properties",
    "maxSubmissions": 1,
    "isPreOrder": false
  },
  "createdAt": "2026-03-31T10:00:00Z",
  "updatedAt": "2026-03-31T10:00:00Z"
}
```

**Field Reference:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `slug` | string | Yes | — | URL-safe identifier (`^[a-z0-9]+(?:-[a-z0-9]+)*$`) |
| `description` | string | Yes | — | Form name (frontend: form title) |
| `type` | `user_properties` \| `form` | No | `user_properties` | Form type |
| `maxSubmissions` | integer | No | `1` | How many times a user can submit (min: 1) |
| `isPreOrder` | boolean | No | `false` | Pre-order context flag |
| `active` | boolean | Yes | — | Whether the form is enabled |
| `data` | object | No | `null` | Screen configuration and approval rules |

**Notes:**
- The slug must be unique per tenant. If a context with the same slug already exists for this tenant, the API throws `DuplicatedContextException` (409)
- After creation, the frontend redirects to the inputs page to add form fields
- The `data` object is stored as JSONB — you can include any custom fields, but the frontend specifically uses the schema documented above

### Step 2: Add Form Fields (Inputs)

After creating the context, add fields to it. See [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md).

---

## Flow B: Update an Existing Form

### Step 1: Fetch the Current Context

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-context/{tenantContextId}` | Bearer (Admin) |

**Response (200):** Full `TenantContextDto` with nested `context` object.

### Step 2: Update Context + Tenant Context

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/tenants/{tenantId}/admin/tenant-context/{tenantContextId}` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "slug": "kyc-verification",
  "description": "Updated KYC Form",
  "maxSubmissions": 3,
  "active": true,
  "data": {
    "requireSpecificApprover": true,
    "screenConfig": {
      "postKycUrl": "https://myapp.com/new-welcome",
      "position": "left"
    }
  }
}
```

**Notes:**
- `type` cannot be changed after creation
- Only SuperAdmin can update **global** contexts (those with `tenantId = null`). Admin users attempting this get `UnableToChangeGlobalContextException` (403)
- The update is atomic — both context and tenant-context are updated in a single transaction

---

## Flow C: List and Filter Contexts

### List All Contexts for a Tenant

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-context` | Bearer (Admin, User) |

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Items per page (default: 10) |
| `active` | boolean | Filter by active status |
| `type` | ContextType[] | Filter by type (`user_properties`, `form`) |
| `preOrder` | boolean | Filter by pre-order flag |
| `search` | string | Search in description/slug |
| `sortBy` | string | Sort column (default: `createdAt`) |
| `orderBy` | `ASC` \| `DESC` | Sort direction (default: `DESC`) |

**Frontend usage patterns:**
- Form Designer page: fetches all contexts, sorts signup context first
- KYC Config page: filters for signup context specifically (`item.context.slug === 'signup'`)
- Clients page: filters `active === true && type === 'user_properties'`

### Fetch by Slug

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-context/slug/{slug}` | Bearer (Admin, User) |

Common slugs used in the platform:
- `signup` — the main signup/registration form
- `admin-notification-settings` — notification configuration context

---

## Flow D: Configure the Signup Form (Special Case)

The signup form has a dedicated flow in the frontend (`/dash/kyc/signup`). It uses the slug `signup` and type `user_properties`.

### Step 1: Check if Signup Context Exists

```
GET /tenants/{tenantId}/tenant-context?type=user_properties
```

Filter the response for `item.context.slug === 'signup'`.

### Step 2a: Create if Not Exists

```
POST /tenants/{tenantId}/admin/tenant-context
```

```json
{
  "slug": "signup",
  "description": "SignUp Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": false
}
```

### Step 2b: Update if Exists

```
PATCH /tenants/{tenantId}/tenant-context/{tenantContextId}
```

```json
{
  "active": true,
  "data": {
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": true,
      "position": "center",
      "textContainer": {
        "mainText": "Welcome!",
        "subtitleText": "Complete your profile to get started"
      }
    }
  }
}
```

**Note:** The signup form update uses the non-admin PATCH endpoint (`/tenant-context/{id}`) which only updates `active` and `data` — not the underlying context fields.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 409 | DuplicatedContextException | Slug + tenantId already exists | Use a different slug or update the existing context |
| 404 | TenantContextNotFoundException | Context not found or not linked to tenant | Verify the tenantContextId and tenantId |
| 404 | ContextBySlugNotFoundException | No context with this slug | Check slug spelling |
| 403 | UnableToChangeGlobalContextException | Admin tried to update a global context | Only SuperAdmin can modify global contexts |
| 400 | TenantContextDisabledException | Context is inactive | Activate the context first |
| 400 | TenantContextCantHaveApproversException | Auto-approved context has no approvers | Set `autoApprove: false` to enable approvers |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Slug validation fails | Slug must be lowercase, alphanumeric with hyphens only. No spaces, underscores, or special characters. Pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*$` |
| 2 | Using wrong PATCH endpoint | Admin PATCH (`/admin/tenant-context/{id}`) updates both context + tenant-context. Regular PATCH (`/tenant-context/{id}`) only updates `active` and `data` |
| 3 | Global context update fails for Admin | Global contexts (tenantId = null) can only be modified by SuperAdmin |
| 4 | Context appears duplicated | The same Context can be linked to multiple tenants via TenantContext. Each tenant gets its own config |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Auth | Required before any admin operation | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| Input Management | Next step after creating a context | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Signup Activation | Toggle signup form on/off | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) |
