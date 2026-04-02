---
id: FLOW_KYC_APPROVAL
title: "KYC - Approval & Review"
module: offpix/kyc
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - approval
  - review
  - admin
  - audit
depends_on:
  - KYC_API_REFERENCE
  - FLOW_KYC_SUBMISSION
---

# KYC Approval & Review

## Overview

This flow covers the admin/approver side of KYC: listing pending submissions, reviewing submitted documents, and taking action (approve, reject, or request review). Admins access this through the dashboard at `/dash/contacts/KYC` for the list view and `/dash/contacts/clients/{id}` for individual review.

W3Block supports three approval workflow types:

| Workflow | Configuration | Behavior |
|----------|--------------|----------|
| Auto-approve | `autoApprove: true` | Documents approved instantly on submission, no admin action needed |
| Standard | `autoApprove: false`, `requireSpecificApprover: false` | Any Admin or KycApprover can review and act |
| Specific approver | `requireSpecificApprover: true` | Only the approver assigned during submission can review |

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin or KycApprover role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Pending submissions | Users must have submitted KYC documents with status `CREATED` | [KYC Submission flow](./FLOW_KYC_SUBMISSION.md) |

---

## Flow: List Pending Submissions

### Option A: Via User Contexts (Full Filtering)

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{tenantId}/users/contexts/find` | Bearer (Admin, KycApprover) |

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | `1` | Page number |
| `limit` | integer | `10` | Items per page |
| `search` | string | -- | Search in user email/name |
| `sortBy` | array | `[{column: 'createdAt', order: 'ASC'}]` | Sort configuration |
| `status[]` | UserContextStatus[] | -- | Filter by status(es) |
| `contextId[]` | UUID[] | -- | Filter by specific context IDs |
| `contextType[]` | ContextType[] | -- | Filter by `user_properties` or `form` |
| `userId[]` | UUID[] | -- | Filter by specific users (Admin only) |
| `excludeSelfContexts` | boolean | -- | KycApprover: exclude own submissions |
| `utmCampaign` | string | -- | Filter by UTM campaign |
| `utmSource` | string | -- | Filter by UTM source |

**Example -- pending submissions sorted oldest first:**
```
GET /{tenantId}/users/contexts/find?status[]=created&sortBy[0][column]=createdAt&sortBy[0][order]=ASC&limit=20
```

**Example -- pending and review-requested submissions:**
```
GET /{tenantId}/users/contexts/find?status[]=created&status[]=requiredReview&limit=20
```

**Response (200):**
```json
{
  "items": [
    {
      "id": "uc-uuid-1",
      "tenantId": "tenant-uuid",
      "userId": "user-uuid-1",
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
    },
    {
      "id": "uc-uuid-2",
      "tenantId": "tenant-uuid",
      "userId": "user-uuid-2",
      "contextId": "ctx-uuid",
      "status": "created",
      "approverUserId": "approver-uuid",
      "utmParams": { "utm_campaign": "onboarding" },
      "createdAt": "2026-01-16T08:30:00Z",
      "updatedAt": "2026-01-16T08:30:00Z",
      "user": {
        "name": "Jane Smith",
        "email": "jane@example.com"
      },
      "context": {
        "slug": "signup",
        "description": "SignUp Form",
        "type": "user_properties"
      }
    }
  ],
  "meta": {
    "totalItems": 42,
    "itemCount": 2,
    "itemsPerPage": 20,
    "totalPages": 3,
    "currentPage": 1
  }
}
```

### Option B: Via Customer Infos (Simplified List)

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

This endpoint provides a simplified view with name, email, and status. For full document details, use the user contexts endpoint.

---

## Flow: Review Submission Details

Once you have a submission from the list, fetch the full details including all documents, input definitions, and asset URLs.

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{tenantId}/users/contexts/{userId}/{userContextId}` | Bearer (User, Admin, KycApprover) |

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
    },
    {
      "id": "doc-uuid-2",
      "inputId": "input-uuid-cpf",
      "status": "created",
      "simpleValue": "12345678901",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid-cpf",
        "label": "CPF",
        "type": "cpf",
        "mandatory": true
      },
      "asset": null
    },
    {
      "id": "doc-uuid-3",
      "inputId": "input-uuid-birthdate",
      "status": "created",
      "simpleValue": "1990-05-15",
      "complexValue": null,
      "assetId": null,
      "input": {
        "id": "input-uuid-birthdate",
        "label": "Date of Birth",
        "type": "birthdate",
        "mandatory": true
      },
      "asset": null
    },
    {
      "id": "doc-uuid-4",
      "inputId": "input-uuid-id-doc",
      "status": "created",
      "simpleValue": null,
      "complexValue": {
        "docType": "rg",
        "document": "123456789"
      },
      "assetId": "asset-uuid-front",
      "input": {
        "id": "input-uuid-id-doc",
        "label": "Government ID",
        "type": "identification_document",
        "mandatory": true
      },
      "asset": {
        "id": "asset-uuid-front",
        "url": "https://res.cloudinary.com/w3block/image/upload/v1/kyc-documents/front.jpg",
        "mimeType": "image/jpeg"
      }
    },
    {
      "id": "doc-uuid-5",
      "inputId": "input-uuid-selfie",
      "status": "created",
      "simpleValue": null,
      "complexValue": null,
      "assetId": "asset-uuid-selfie",
      "input": {
        "id": "input-uuid-selfie",
        "label": "Selfie Verification",
        "type": "multiface_selfie",
        "mandatory": true
      },
      "asset": {
        "id": "asset-uuid-selfie",
        "url": "https://res.cloudinary.com/w3block/image/upload/v1/kyc-documents/selfie.jpg",
        "mimeType": "image/jpeg"
      }
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

**Frontend rendering guide:**
- Display each document using its `input.label` as the field name
- For text types (`simpleValue`): render the value directly
- For complex types (`complexValue`): render structured data (e.g., document type + number)
- For file types (`asset`): render image preview or document viewer using `asset.url`
- Show the current `status` badge for each document
- Display the audit `logs` as a timeline

---

## Flow: Approve

Approve a submission, marking all documents and the context as approved.

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
  "reason": "All documents verified successfully. Identity confirmed.",
  "userContextId": "uc-uuid"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Approval reason (saved in audit log) |
| `userContextId` | UUID | No | Specific user context to approve |

**Response:** 204 No Content

**What happens internally:**

1. All documents in the context are set to status `APPROVED`
2. User context status is set to `APPROVED`
3. Audit log entry appended: `{ moderatorId, status: "approved", reason, registerAt }`
4. KYC approval email sent to user (if `KYC_USER_APPROVED` notification is configured)
5. KYC approval notification sent to admins (if `KYC_ADMIN_APPROVED` notification is configured)
6. Commerce service notified -- may unblock pending orders if `blockCommerceDeliver` is true on the context
7. Webhook `USER_KYC_PROPS_UPDATED` dispatched

---

## Flow: Reject

Reject a submission. This is a terminal state -- the user cannot resubmit to a denied context.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/reject` | Bearer (Admin, KycApprover) | application/json |

**Minimal Request:**
```json
{}
```

**Complete Request:**
```json
{
  "reason": "Submitted documents are fraudulent. Account flagged for review.",
  "userContextId": "uc-uuid"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Rejection reason (sent to user via email) |
| `userContextId` | UUID | No | Specific user context to reject |

**Response:** 204 No Content

**What happens internally:**

1. All documents in the context are set to status `DENIED`
2. User context status is set to `DENIED`
3. Audit log entry appended with moderator ID, status, reason, and timestamp
4. KYC rejection email sent to user with reason (if `KYC_USER_REJECTED` notification is configured)
5. KYC rejection notification sent to admins (if `KYC_ADMIN_REJECTED` notification is configured)
6. Commerce service notified of KYC failure

**IMPORTANT:** `DENIED` is a terminal state. The user cannot resubmit documents to a denied context. If you want the user to correct and resubmit specific fields, use "require review" instead. A new context submission would need to be created if rejection was made in error.

---

## Flow: Request Review

Flag specific fields for resubmission. This is a non-terminal state that allows the user to correct and resubmit only the flagged fields.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{tenantId}/users/contexts/{userId}/{contextId}/require-review` | Bearer (Admin, KycApprover) | application/json |

**Request:**
```json
{
  "inputIds": ["input-uuid-id-doc", "input-uuid-selfie"],
  "reason": "Government ID is expired (expiration date 2025-12-31). Selfie image is too dark to verify identity."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inputIds` | UUID[] | Yes (min: 1) | Input IDs of fields that need resubmission |
| `reason` | string | No | Explanation sent to the user |

**Response:** 204 No Content

**What happens internally:**

1. Only the documents matching the specified `inputIds` are set to status `REQUIRED_REVIEW`
2. Other documents retain their current status (remain `CREATED`)
3. User context status is set to `REQUIRED_REVIEW`
4. Audit log entry appended with `inputIds`, reason, and timestamp
5. Review request email sent to user specifying which fields need attention (if `KYC_REQUIRE_USER_REVIEW` notification is configured)

**After user resubmits:**
- The flagged documents return to `CREATED` status with new values
- The user context returns to `CREATED` status
- The submission re-enters the review queue

---

## Permission Model

### Role-Based Access

| Role | Can list submissions | Can approve | Can reject | Can request review |
|------|:-------------------:|:-----------:|:----------:|:------------------:|
| SuperAdmin | Yes (all) | Yes (all) | Yes (all) | Yes (all) |
| Admin | Yes (all) | Yes (all) | Yes (all) | Yes (all) |
| KycApprover | Yes (restricted) | Yes (restricted) | Yes (restricted) | Yes (restricted) |
| User | No | No | No | No |

### KycApprover Restrictions

KycApprovers have additional restrictions beyond Admin users:

**1. Specific Approver Requirement**
If `requireSpecificApprover: true` on the TenantContext, KycApprovers can only approve submissions where `approverUserId` matches their own user ID. Submissions assigned to other approvers are not accessible.

**2. Whitelist Restriction**
If `approverWhitelistIds` is set on the TenantContext, only KycApprovers who belong to one of those whitelists can see and act on submissions for that context. Approvers not in any of the specified whitelists receive a 403 Forbidden error.

**3. Self-Exclusion**
KycApprovers cannot approve their own submissions. Use the `excludeSelfContexts: true` filter when listing to automatically exclude own submissions from the results.

**4. Auto-Approve Exclusion**
If `autoApprove: true` on the TenantContext, KycApprovers cannot access those contexts since submissions are auto-approved and require no manual review.

### Permission Decision Flow

```
Is user SuperAdmin or Admin?
  YES → Can approve any submission
  NO  → Is user KycApprover?
    NO  → Access denied
    YES → Is autoApprove enabled?
      YES → Access denied (no manual review needed)
      NO  → Is requireSpecificApprover enabled?
        YES → Is approverUserId == current user?
          YES → Can approve
          NO  → Access denied
        NO  → Are approverWhitelistIds set?
          YES → Is user in one of the whitelists?
            YES → Can approve
            NO  → Access denied
          NO  → Is this the user's own submission?
            YES → Access denied
            NO  → Can approve
```

---

## Audit Trail

Every status change on a user context appends a `LogUserContext` entry to the `logs` array. This provides a complete, immutable history of the submission lifecycle.

**Example -- full lifecycle:**
```json
{
  "logs": [
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-15T10:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "requiredReview",
      "inputIds": ["input-uuid-id-doc"],
      "reason": "Government ID is expired",
      "registerAt": "2026-01-16T14:00:00Z"
    },
    {
      "moderatorId": "user-uuid",
      "status": "created",
      "inputIds": [],
      "registerAt": "2026-01-17T09:00:00Z"
    },
    {
      "moderatorId": "admin-uuid",
      "status": "approved",
      "inputIds": [],
      "reason": "All documents verified",
      "registerAt": "2026-01-17T15:00:00Z"
    }
  ]
}
```

**LogUserContext fields:**

| Field | Type | Description |
|-------|------|-------------|
| `moderatorId` | UUID | User who performed the action (user for submissions, admin for reviews) |
| `status` | UserContextStatus | Status set by this action |
| `inputIds` | UUID[] | Input IDs affected (populated for require-review, empty for other actions) |
| `reason` | string (nullable) | Reason provided for the action |
| `registerAt` | DateTime | When the action was performed |

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | `BadRequestException` | Context not in approvable state | Context must be `CREATED` or `REQUIRED_REVIEW` to approve/reject/require-review |
| 400 | `BadRequestException` | No `inputIds` provided for require-review | At least one `inputId` is required when requesting review |
| 403 | `ForbiddenException` | Approver not authorized for this context | Check whitelist membership and specific approver assignment |
| 403 | `ForbiddenException` | KycApprover trying to approve own submission | KycApprovers cannot act on their own submissions |
| 404 | `UserContextNotFoundException` | User context not found | Verify `userId`, `contextId`, `userContextId`, and `tenantId` |

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | KycApprover cannot see submissions | Check if the context has `approverWhitelistIds` set. The approver must be a member of one of the specified whitelists |
| 2 | Rejection is permanent | Unlike "require review," rejection sets status to `DENIED` which is terminal. The user cannot resubmit. Use "require review" for correctable issues |
| 3 | Approve endpoint returns 400 | The context must be in `CREATED` or `REQUIRED_REVIEW` status. A `DRAFT` context (incomplete submission) cannot be approved |
| 4 | Commerce orders still blocked after approval | If `blockCommerceDeliver` was set on the tenant context, verify the commerce service received the KYC approval event |
| 5 | Partial approval confusion | The "require review" action only flags the specified `inputIds`. Other documents keep their current status unchanged |
| 6 | Missing audit trail | Every action is logged in the `logs` array. Check the logs for the full history of status changes, including who acted and when |
| 7 | Approver assigned but different admin approving | When `requireSpecificApprover: true`, only the `approverUserId` set during submission can approve. Other admins (non-SuperAdmin) receive a 403 |

---

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| KYC Submission | User-side document submission | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| KYC Configuration | Approval workflow setup | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Context Setup | How KYC contexts are created | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Input Management | How form fields are defined | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| API Reference | Full endpoint and DTO details | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
