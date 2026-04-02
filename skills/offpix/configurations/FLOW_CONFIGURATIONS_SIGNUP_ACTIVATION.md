---
id: FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION
title: "Configurations - Signup Activation / Deactivation"
module: offpix/configurations
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - signup
  - activation
  - kyc
depends_on:
  - CONFIGURATIONS_API_REFERENCE
  - FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
---

# Signup Activation / Deactivation

## Overview

This flow covers enabling and disabling the signup registration form for a tenant. When activated, new users can register through a KYC-enabled signup form. When deactivated, the signup form is hidden. In the frontend, this is a toggle switch on the **KYC Configuration** page (`/dash/kyc`), labeled "Registration".

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |

## Flow: Check Current State

### Step 1: Fetch Tenant Contexts

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenants/{tenantId}/tenant-context` | Bearer (Admin) |

**Query (optional):** `?type=user_properties`

Check the response for a context with `context.slug === 'signup'` (the frontend matches by `context.description.toUpperCase() === 'SIGNUP'`).

- If found with `active: true` → Signup is ON
- If found with `active: false` → Signup is OFF
- If not found → Signup has never been configured

---

## Flow: Activate Signup

### Step 1: Call Activate Endpoint

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenant-contexts/activate-signup/{tenantId}` | Bearer (Admin) |

No request body required. This is a GET request that triggers the activation.

**Response (200):** HTTP 200 status indicates success.

**Notes:**
- This endpoint creates the signup context if it doesn't exist, or activates it if it was previously deactivated
- The frontend treats HTTP 200 as success and updates the toggle UI accordingly

---

## Flow: Deactivate Signup

### Step 1: Call Deactivate Endpoint

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenant-contexts/deactivate-signup/{tenantId}` | Bearer (Admin) |

No request body required.

**Response (200):** HTTP 200 status indicates success.

**Notes:**
- Deactivating signup hides the registration form from end users
- Existing registered users are not affected
- The signup context and its inputs are preserved — reactivating restores the previous configuration

---

## Flow: Configure the Signup Form Content

After activating signup, configure the form appearance and fields:

### Step 1: Create or Update Signup Context

If the signup context doesn't exist yet, create it:

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
  "active": true,
  "data": {
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": false,
      "position": "center",
      "textContainer": {
        "mainText": "Create Your Account",
        "subtitleText": "Fill in the details below"
      }
    }
  }
}
```

If it already exists, update it:

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
        "subtitleText": "Get started in minutes"
      }
    }
  }
}
```

### Step 2: Add Form Fields

Add inputs to the signup context. See [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md).

Typical signup form fields:

| Order | Type | Label | Mandatory |
|-------|------|-------|-----------|
| 1 | `user_name` | Full Name | Yes |
| 2 | `email` | Email Address | Yes |
| 3 | `cpf` | CPF | Yes (Brazil) |
| 4 | `phone` | Phone Number | No |
| 5 | `identification_document` | Government ID | Yes |
| 6 | `multiface_selfie` | Selfie | Yes |

---

## Fetching Enabled Document Types

The KYC configuration page also displays which document types are enabled for the tenant.

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/data-types/{tenantId}` | Bearer (Admin) |

**Response (200):**
```json
[
  {
    "id": "dtype-uuid",
    "tenantId": "tenant-uuid",
    "label": "ID Document",
    "type": "document",
    "description": "Government ID document",
    "enabled": true,
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-01-01T00:00:00Z"
  }
]
```

The frontend filters this list to show only `enabled: true` items, displaying the `label` field.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 401 | Unauthorized | Invalid or missing token | Re-authenticate |
| 403 | Forbidden | User lacks Admin role | Use an Admin account |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Signup toggle doesn't stick | Verify the activate/deactivate endpoint returns 200. The frontend only updates UI state on success |
| 2 | Signup form shows but has no fields | Activation only enables the form — you still need to create inputs (fields) for it |
| 3 | Deactivated form loses configuration | No — deactivation preserves all context and input data. Reactivation restores everything |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Context Lifecycle | Full context management | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Input Management | Add fields to the signup form | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Auth Signup | The user-facing signup flow | [FLOW_AUTH_SIGNUP](../auth/FLOW_AUTH_SIGNUP.md) |
