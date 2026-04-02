---
id: FLOW_CONFIGURATIONS_INPUT_MANAGEMENT
title: "Configurations - Input (Form Field) Management"
module: offpix/configurations
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - tenant-input
  - form-fields
  - kyc
depends_on:
  - CONFIGURATIONS_API_REFERENCE
  - FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
---

# Input (Form Field) Management

## Overview

Inputs are the individual fields within a context form — text fields, file uploads, selects, document requests, etc. This flow covers creating, updating, ordering, and retrieving form fields. In the frontend, this is managed under **KYC > Form Designer > [Context] > Inputs** (`/dash/kyc/inputs?id={contextId}`).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| `contextId` | UUID of the context to add fields to | [Context Lifecycle](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

## Entities

```
TenantInput
├── label         (display name shown to user)
├── description   (help text / placeholder)
├── type          (DataTypesEnum: text, email, file, etc.)
├── order         (integer: display sequence)
├── mandatory     (boolean: is this field required?)
├── active        (boolean: is this field shown?)
├── step          (number: form page/step grouping)
├── attributeName (custom key for the submitted value)
├── options       (array: for select-type fields)
└── data          (object: type-specific config)
```

**19 input types** across 5 categories:

| Category | Types |
|----------|-------|
| **Text input** | `text`, `email`, `phone`, `cpf`, `url`, `user_name` |
| **Date** | `date`, `birthdate` |
| **File upload** | `file`, `image`, `identification_document`, `multiface_selfie` |
| **Selection** | `simple_select`, `dynamic_select`, `checkbox` |
| **Special** | `simple_location`, `commerce_product`, `iframe`, `separator` |

---

## Flow: Add Fields to a Form

### Step 1: Create a Form Field

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/tenants/{tenantId}/tenant-input` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "contextId": "ctx-uuid",
  "label": "Full Name",
  "description": "Enter your full legal name",
  "type": "text",
  "order": 1,
  "mandatory": true,
  "active": true
}
```

**Complete Request (production example):**
```json
{
  "contextId": "ctx-uuid",
  "label": "Full Name",
  "description": "Enter your full legal name",
  "type": "text",
  "order": 1,
  "mandatory": true,
  "active": true,
  "attributeName": "fullName",
  "step": 1,
  "options": null,
  "data": null
}
```

**Response (201):**
```json
{
  "id": "input-uuid",
  "contextId": "ctx-uuid",
  "tenantId": "tenant-uuid",
  "label": "Full Name",
  "description": "Enter your full legal name",
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
}
```

**Field Reference:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `contextId` | UUID | Yes | — | Context this field belongs to |
| `label` | string | Yes | — | Field label displayed to the user |
| `description` | string | Yes | — | Help text or placeholder |
| `type` | DataTypesEnum | Yes | — | Field type (cannot be changed after creation) |
| `order` | integer | Yes | — | Display order (1-based, ascending) |
| `mandatory` | boolean | Yes | — | Whether the field is required for form submission |
| `active` | boolean | Yes | — | Whether the field is currently visible |
| `attributeName` | string | No | Same as `type` | Custom key for the submitted value. Use this to give meaningful names to responses |
| `step` | number | No | `null` | Group fields into form steps/pages |
| `options` | array | No | `[]` | For select types: `[{ "label": "Display", "value": "stored" }]` |
| `data` | object | No | `null` | Type-specific configuration |

**Notes:**
- If `attributeName` is not provided, it defaults to the `type` value (e.g., "email", "text"). Set a custom value to distinguish multiple fields of the same type
- When a **mandatory + active** input is created, all existing users are flagged for review in the users-contexts system. This can trigger KYC resubmission notifications

### Step 2: Add More Fields

Repeat Step 1 for each field, incrementing the `order` value:

```
Order 1: Full Name (text, mandatory)
Order 2: Email (email, mandatory)
Order 3: Phone (phone, optional)
Order 4: ID Document Front (identification_document, mandatory)
Order 5: Selfie (multiface_selfie, mandatory)
```

---

## Type-Specific Examples

### Select Field (Dropdown)

```json
{
  "contextId": "ctx-uuid",
  "label": "Country",
  "description": "Select your country",
  "type": "simple_select",
  "order": 3,
  "mandatory": true,
  "active": true,
  "attributeName": "country",
  "options": [
    { "label": "Brazil", "value": "BR" },
    { "label": "United States", "value": "US" },
    { "label": "Portugal", "value": "PT" }
  ]
}
```

### Commerce Product Selector

```json
{
  "contextId": "ctx-uuid",
  "label": "Select a Plan",
  "description": "Choose your subscription plan",
  "type": "commerce_product",
  "order": 6,
  "mandatory": true,
  "active": true,
  "attributeName": "selectedPlan",
  "data": {
    "productIds": ["product-uuid-1", "product-uuid-2"]
  }
}
```

**Validation:** Each product ID in `productIds` is verified against the Commerce service. Invalid IDs throw `InvalidTenantDataException` (400).

### Document Upload

```json
{
  "contextId": "ctx-uuid",
  "label": "Government ID",
  "description": "Upload front and back of your ID",
  "type": "identification_document",
  "order": 4,
  "mandatory": true,
  "active": true,
  "attributeName": "govId"
}
```

### Separator (Layout Element)

```json
{
  "contextId": "ctx-uuid",
  "label": "--- Documents Section ---",
  "description": "Upload the following documents",
  "type": "separator",
  "order": 3,
  "mandatory": false,
  "active": true
}
```

**Note:** Separators are skipped during form validation — they are visual-only elements.

---

## Flow: Update a Field

### Step 1: Update the Input

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/tenants/{tenantId}/tenant-input/{inputId}` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "label": "Updated Label",
  "description": "Updated description",
  "order": 2,
  "mandatory": false,
  "active": true,
  "attributeName": "updatedAttr",
  "step": 2,
  "options": null,
  "data": null
}
```

**Notes:**
- `contextId` and `type` **cannot be changed** after creation
- If the update makes a field `mandatory + active` (and it wasn't before), all existing users are flagged for review

---

## Flow: Retrieve Form Fields (Public)

This endpoint is used to render forms to end users. It returns only **active** fields ordered by `order ASC`.

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-input/slug/{slug}` | **None (Public)** |

**Response (200):**
```json
[
  {
    "id": "input-uuid-1",
    "label": "Full Name",
    "type": "text",
    "order": 1,
    "mandatory": true,
    "active": true,
    "attributeName": "fullName",
    "step": 1,
    "options": [],
    "data": null
  },
  {
    "id": "input-uuid-2",
    "label": "Email Address",
    "type": "email",
    "order": 2,
    "mandatory": true,
    "active": true,
    "attributeName": "email",
    "step": 1,
    "options": [],
    "data": null
  }
]
```

**Notes:**
- Only active inputs are returned
- Results are ordered by `order ASC`
- If the context or tenant-context is not found or inactive, the API returns appropriate errors (404 or 400)

---

## Flow: List Inputs (Admin, Paginated)

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/admin/tenant-context/{tenantContextId}/inputs` | Bearer (Admin) |

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 10 | Items per page (frontend uses 4 on mobile) |
| `search` | string | — | Search in label/description |

**Frontend usage:** The inputs page (`/dash/kyc/inputs?id={contextId}`) fetches inputs sorted by `order ASC` and renders them as an editable list.

---

## Multi-Step Forms

Group fields into steps using the `step` field:

```
Step 1: Personal Info
  ├── Order 1: Full Name (text)
  ├── Order 2: Email (email)
  └── Order 3: Phone (phone)

Step 2: Documents
  ├── Order 4: ID Document (identification_document)
  └── Order 5: Selfie (multiface_selfie)

Step 3: Additional
  └── Order 6: Country (simple_select)
```

The `step` field is used for:
- Grouping fields into form pages/tabs in the frontend
- Validating that required documents exist per step (via `checkAllRequiredDocumentsHaveBeenSubmitted`)

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 404 | TenantParamsNotFoundException | Input not found | Verify inputId |
| 400 | InvalidTenantDataException | Type-specific validation failed | Check `data` field for the specific type (e.g., valid productIds for commerce_product) |
| 404 | ContextBySlugNotFoundException | Slug not found (public endpoint) | Verify slug spelling and tenant |
| 404 | TenantContextNotFoundException | Context not linked to tenant | Create the tenant-context association first |
| 400 | TenantContextDisabledException | Context is inactive (public endpoint) | Activate the context |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | `attributeName` defaults to type string | Always set a custom `attributeName` when you have multiple fields of the same type (e.g., two `text` fields). Otherwise both would share the key "text" |
| 2 | Adding mandatory field triggers user reviews | Creating or updating a field to `mandatory + active` flags all existing users for resubmission. Plan your form structure before creating inputs |
| 3 | `type` cannot be changed after creation | If you need a different type, delete the old input and create a new one |
| 4 | Public slug endpoint returns empty for inactive fields | Only active inputs are returned. Deactivated fields are hidden from end users but still visible in admin views |
| 5 | Commerce product validation fails | For `commerce_product` type, `data.productIds` must contain valid UUIDs of existing products in the Commerce service |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Context Lifecycle | Create the context before adding inputs | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Signup Activation | Toggle signup form visibility | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) |
| API Reference | Full endpoint and enum details | [CONFIGURATIONS_API_REFERENCE](./CONFIGURATIONS_API_REFERENCE.md) |
