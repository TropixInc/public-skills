---
id: FLOW_CONTACTS_KYC_APPROVAL
title: "Contacts - KYC Approval & Review"
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
  - approval
  - review
  - documents
depends_on:
  - CONTACTS_API_REFERENCE
  - FLOW_CONTACTS_KYC_SUBMISSION
---

# KYC Approval & Review

## Overview

This flow covers the admin/approver side of KYC: listing pending submissions, reviewing documents, and taking action (approve, reject, or request review). In the frontend, admins access this through **Contacts > KYC** (`/dash/contacts/KYC`) for the list, and **Contact Details** (`/dash/contacts/clients/{id}`) for individual review.

W3Block supports three approval workflows:
- **Auto-approve** — documents approved instantly on submission (no admin action needed)
- **Standard approval** — any Admin or KycApprover can review
- **Specific approver** — only the assigned approver can review

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin or KycApprover role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Pending submissions | Users must have submitted KYC documents | [KYC Submission flow](./FLOW_CONTACTS_KYC_SUBMISSION.md) |

## Approval Workflow Types

```
Type 1: Auto-Approve (autoApprove=true on TenantContext)
  User submits → Documents auto-approved → Done

Type 2: Standard (autoApprove=false, requireSpecificApprover=false)
  User submits → Any Admin/KycApprover reviews → Approve/Reject/Review

Type 3: Specific Approver (requireSpecificApprover=true)
  User submits (with approverUserId) → Only that approver reviews → Approve/Reject/Review
```

**Whitelist filtering:** If the TenantContext has `approverWhitelistIds` set, only KycApprovers who belong to those whitelists can see and approve the submissions.

---

## Flow: List KYC Submissions

### Option A: Via Customer Infos (KYC List page)

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/customer-infos/{tenantId}/search` | Bearer (Admin) |

**Query Parameters:** `page`, `limit`, `sortBy=createdAt`, `orderBy`

**Response (200):**
```json
{
  "items": [
    {
      "id": "ci-uuid",
      "customerId": "user-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "status": "created",
      "tenantId": "tenant-uuid",
      "createdAt": "2026-01-15T10:00:00Z"
    }
  ]
}
```

**Frontend display:** Name, creation date, and status badge (green/red/orange).

### Option B: Via User Contexts (Detailed filtering)

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{tenantId}/users/contexts/find` | Bearer (Admin, KycApprover) |

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `status[]` | UserContextStatus[] | Filter: `CREATED`, `REQUIRED_REVIEW`, etc. |
| `contextId[]` | UUID[] | Filter by specific context |
| `contextType[]` | ContextType[] | `user_properties` or `form` |
| `userId[]` | UUID[] | Filter by specific users |
| `excludeSelfContexts` | boolean | KycApprover: exclude own submissions |
| `page`, `limit` | integer | Pagination |

**Example — pending submissions:**
```
GET /{tenantId}/users/contexts/find?status[]=CREATED&status[]=REQUIRED_REVIEW&sortBy[0][column]=createdAt&sortBy[0][order]=ASC
```

---

## Flow: Review a Submission

### Step 1: Fetch User Context with Documents

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{tenantId}/users/contexts/{userId}/{userContextId}` | Bearer (Admin, KycApprover) |

**Response (200):**
```json
{
  "id": "uc-uuid",
  "userId": "user-uuid",
  "contextId": "ctx-uuid",
  "status": "CREATED",
  "approverUserId": null,
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
      "inputId": "input-uuid-name",
      "status": "CREATED",
      "simpleValue": "John Doe",
      "complexValue": null,
      "assetId": null,
      "input": {
        "label": "Full Name",
        "type": "user_name",
        "mandatory": true
      }
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-cpf",
      "status": "CREATED",
      "simpleValue": "12345678901",
      "complexValue": null,
      "assetId": null,
      "input": {
        "label": "CPF",
        "type": "cpf",
        "mandatory": true
      }
    },
    {
      "id": "doc-uuid-3",
      "inputId": "input-uuid-doc",
      "status": "CREATED",
      "simpleValue": null,
      "complexValue": null,
      "assetId": "asset-uuid",
      "input": {
        "label": "Government ID",
        "type": "identification_document",
        "mandatory": true
      },
      "asset": {
        "id": "asset-uuid",
        "url": "https://res.cloudinary.com/...",
        "mimeType": "image/jpeg"
      }
    }
  ],
  "context": {
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

**Frontend rendering:** Each document is displayed using its `input.label` and the value (`simpleValue` for text, `asset.url` for files). The `input.type` determines how the value is rendered (text, image preview, document viewer, etc.).

### Step 2: Take Action

Choose one of three actions:

---

## Action A: Approve

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/approve` | Bearer (Admin, KycApprover) | application/json |

**Minimal Request:**
```json
{}
```

**Complete Request:**
```json
{
  "reason": "All documents verified successfully",
  "userContextId": "uc-uuid"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Approval reason (saved in audit log) |
| `userContextId` | UUID | No | Specific user context to approve |

**Response:** 204 No Content

**What happens internally:**
1. All documents → status `APPROVED`
2. User context → status `APPROVED`
3. Audit log entry added: `{ moderatorId, status: APPROVED, reason, registerAt }`
4. Email: KYC approval notification sent to user
5. Commerce: KYC status event dispatched (may unblock orders if `blockCommerceDeliver` was set)
6. Webhook: `USER_KYC_PROPS_UPDATED` fired

---

## Action B: Reject

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/reject` | Bearer (Admin, KycApprover) | application/json |

**Request:**
```json
{
  "reason": "Document is expired. Please submit a valid government ID."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Rejection reason (sent to user via email) |

**Response:** 204 No Content

**What happens internally:**
1. All documents → status `DENIED`
2. User context → status `DENIED`
3. Audit log entry added
4. Email: KYC rejection notification sent to user with reason
5. Commerce: KYC failure event dispatched

**Note:** Rejection is a terminal state. The user cannot resubmit to a `DENIED` context. A new context submission would need to be created.

---

## Action C: Request Review (Partial Rejection)

This is the most nuanced action — it flags specific fields for resubmission while keeping the rest intact.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/require-review` | Bearer (Admin, KycApprover) | application/json |

**Request:**
```json
{
  "inputIds": ["input-uuid-doc", "input-uuid-selfie"],
  "reason": "Government ID is blurry and selfie doesn't match the document photo"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inputIds` | UUID[] | Yes (min: 1) | Input IDs of fields that need resubmission |
| `reason` | string | No | Explanation sent to the user |

**Response:** 204 No Content

**What happens internally:**
1. Only the specified documents → status `REQUIRED_REVIEW`
2. Other documents remain in their current status
3. User context → status `REQUIRED_REVIEW`
4. Audit log entry added with `inputIds` and `reason`
5. Email: Review request notification sent to user, specifying which fields need attention

**After user resubmits:** The flagged documents return to `CREATED` status, and the user context returns to `CREATED` — back in the review queue.

---

## Approval Permission Rules

| Scenario | Who can approve |
|----------|----------------|
| Standard context | Any Admin or KycApprover |
| Context with `approverWhitelistIds` | Only KycApprovers who are in one of the specified whitelists |
| Context with `requireSpecificApprover: true` | Only the user specified in `approverUserId` on the submission |
| KycApprover reviewing own submission | Not allowed (`excludeSelfContexts` filter) |
| SuperAdmin | Can approve any context regardless of whitelist/approver rules |

---

## Audit Trail

Every action is logged in the `logs` array on the user context:

```json
{
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "REQUIRED_REVIEW",
      "inputIds": ["input-uuid-doc"],
      "reason": "Document is blurry",
      "registerAt": "2026-01-16T14:00:00Z"
    },
    {
      "moderatorId": "user-uuid",
      "status": "CREATED",
      "inputIds": [],
      "registerAt": "2026-01-17T09:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "APPROVED",
      "inputIds": [],
      "reason": "Documents verified",
      "registerAt": "2026-01-17T15:00:00Z"
    }
  ]
}
```

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 404 | UserContextNotFoundException | User context not found | Verify userId, contextId, and tenantId |
| 403 | ForbiddenException | Approver not authorized for this context | Check whitelist/specific approver rules |
| 400 | BadRequestException | Context not in approvable state | Context must be `CREATED` or `REQUIRED_REVIEW` to approve/reject |
| 400 | BadRequestException | No inputIds provided for require-review | At least one inputId is required |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | KycApprover can't see submissions | Check if the context has `approverWhitelistIds` — the approver may not be in the whitelist |
| 2 | Rejection is permanent | Unlike "require review", rejection sets status to `DENIED` which is terminal. Use "require review" for resubmission |
| 3 | Approve endpoint returns 400 | The context must be in `CREATED` or `REQUIRED_REVIEW` status. Check current status first |
| 4 | Commerce orders still blocked after approval | If `blockCommerceDeliver` was set on the tenant context, verify the commerce service received the KYC event |
| 5 | Partial approval confusion | When using `require-review`, only the specified `inputIds` are flagged. Other documents keep their status |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| KYC Submission | User-side document submission | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| User Management | Managing the user record | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Context Configuration | How forms are set up | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Input Configuration | How form fields are defined | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
