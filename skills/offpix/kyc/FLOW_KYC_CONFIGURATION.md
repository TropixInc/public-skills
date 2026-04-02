---
id: FLOW_KYC_CONFIGURATION
title: "KYC - Configuration"
module: offpix/kyc
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - configuration
  - tenant-context
  - notifications
  - input-types
depends_on:
  - KYC_API_REFERENCE
---

# KYC Configuration

## Overview

This flow covers how to configure KYC contexts for a tenant, including approval workflows, notification settings, and the full catalog of input types. Configuration determines how submissions are processed, who can approve them, and what notifications are sent throughout the lifecycle.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Context created | A KYC context must exist | [Context Lifecycle](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Inputs defined | Form fields must be created | [Input Management](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |

---

## Flow: View KYC Configurations

Retrieve all tenant context configurations to see current approval and notification settings.

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/tenant-context/{tenantId}` | Bearer (Admin) |

**Response (200):**
```json
[
  {
    "id": "tc-uuid-1",
    "tenantId": "tenant-uuid",
    "contextId": "ctx-uuid-signup",
    "active": true,
    "data": null,
    "approverWhitelistIds": null,
    "requireSpecificApprover": false,
    "autoApprove": false,
    "blockCommerceDeliver": false,
    "notificationsConfig": {
      "KYC_APPROVAL_REQUEST": { "enabled": true },
      "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
      "KYC_USER_APPROVED": { "enabled": true },
      "KYC_ADMIN_APPROVED": { "enabled": true },
      "KYC_USER_REJECTED": { "enabled": true },
      "KYC_ADMIN_REJECTED": { "enabled": true }
    }
  },
  {
    "id": "tc-uuid-2",
    "tenantId": "tenant-uuid",
    "contextId": "ctx-uuid-onboarding",
    "active": true,
    "data": null,
    "approverWhitelistIds": ["whitelist-uuid-1", "whitelist-uuid-2"],
    "requireSpecificApprover": true,
    "autoApprove": false,
    "blockCommerceDeliver": true,
    "notificationsConfig": {
      "KYC_APPROVAL_REQUEST": { "enabled": true },
      "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
      "KYC_USER_APPROVED": { "enabled": true },
      "KYC_ADMIN_APPROVED": { "enabled": false },
      "KYC_USER_REJECTED": { "enabled": true },
      "KYC_ADMIN_REJECTED": { "enabled": false }
    }
  }
]
```

---

## Flow: Configure Approval Workflow

The TenantContextEntity controls how KYC submissions are processed. The key configuration options are:

### autoApprove

| Value | Behavior |
|-------|----------|
| `true` | Submissions are automatically approved upon creation. No manual review is needed. All documents and the user context are set to `APPROVED` immediately. Approval emails are sent automatically. |
| `false` | Submissions require manual review by an Admin or KycApprover. Status transitions to `CREATED` and stays there until an approver takes action. |

**Use case:** Enable auto-approve for low-risk contexts (e.g., newsletter signup) where document verification is not critical. Disable for high-risk contexts (e.g., identity verification for financial transactions).

### requireSpecificApprover

| Value | Behavior |
|-------|----------|
| `true` | Users must specify an `approverUserId` when submitting documents. Only that specific approver can review and act on the submission. |
| `false` | Any Admin or authorized KycApprover can review the submission. |

**Use case:** Enable when submissions should be routed to a specific person (e.g., a regional manager who knows the applicant, or a designated compliance officer).

**Submission impact:** When enabled, the `approverUserId` field becomes required in the `AttachDocumentsToUser` DTO. Submissions without it will fail.

### approverWhitelistIds

| Value | Behavior |
|-------|----------|
| `null` or empty | Any KycApprover can review submissions for this context |
| `UUID[]` | Only KycApprovers who are members of one of the specified whitelists can review. Admins and SuperAdmins are not affected by this restriction. |

**Use case:** Restrict approval to a specific team. For example, only compliance team members (in whitelist `compliance-team-uuid`) can approve identity verification contexts.

**Example configuration:**
```json
{
  "approverWhitelistIds": ["whitelist-uuid-compliance", "whitelist-uuid-senior-reviewers"]
}
```

With this configuration, a KycApprover must be a member of either the `compliance` or `senior-reviewers` whitelist to see and act on submissions.

### blockCommerceDeliver

| Value | Behavior |
|-------|----------|
| `true` | Commerce order delivery is blocked until the user's KYC is approved for this context. When KYC is approved, the commerce service is notified and pending orders are unblocked. |
| `false` | Commerce orders are not affected by KYC status. |

**Use case:** Enable for contexts where product delivery requires verified identity (e.g., age-restricted products, regulated goods).

### Configuration Combinations

| autoApprove | requireSpecific | whitelistIds | Result |
|:-----------:|:---------------:|:------------:|--------|
| `true` | `false` | `null` | Instant approval, no review |
| `false` | `false` | `null` | Any admin/approver can review |
| `false` | `true` | `null` | Only assigned approver can review |
| `false` | `false` | `[ids]` | Only whitelisted approvers can review |
| `false` | `true` | `[ids]` | Only assigned approver, who must be in whitelist |
| `true` | `true` | any | Auto-approve overrides; no manual review |

---

## Flow: Configure Notifications

The `notificationsConfig` field on TenantContextEntity controls which email notifications are sent during the KYC lifecycle.

### Notification Types

#### KYC_APPROVAL_REQUEST

| Field | Value |
|-------|-------|
| **Recipient** | Approvers (Admin, KycApprover) |
| **Trigger** | New submission created with status `CREATED` |
| **Content** | Notification that a new KYC submission is ready for review |
| **Use case** | Keep approvers informed of pending work |

#### KYC_REQUIRE_USER_REVIEW

| Field | Value |
|-------|-------|
| **Recipient** | User who submitted |
| **Trigger** | Approver requests review via `require-review` endpoint |
| **Content** | Notification specifying which fields need resubmission and the reason |
| **Use case** | Prompt user to correct and resubmit specific documents |

#### KYC_USER_APPROVED

| Field | Value |
|-------|-------|
| **Recipient** | User who submitted |
| **Trigger** | Submission approved |
| **Content** | Confirmation that KYC verification is complete |
| **Use case** | Inform user their identity is verified |

#### KYC_ADMIN_APPROVED

| Field | Value |
|-------|-------|
| **Recipient** | Admins |
| **Trigger** | Submission approved |
| **Content** | Notification that a submission was approved (includes which admin approved) |
| **Use case** | Audit trail for admin team awareness |

#### KYC_USER_REJECTED

| Field | Value |
|-------|-------|
| **Recipient** | User who submitted |
| **Trigger** | Submission rejected |
| **Content** | Rejection notification with reason |
| **Use case** | Inform user their submission was denied |

#### KYC_ADMIN_REJECTED

| Field | Value |
|-------|-------|
| **Recipient** | Admins |
| **Trigger** | Submission rejected |
| **Content** | Notification that a submission was rejected (includes which admin rejected and reason) |
| **Use case** | Audit trail for admin team awareness |

### Notification Configuration Example

**Enable all notifications:**
```json
{
  "notificationsConfig": {
    "KYC_APPROVAL_REQUEST": { "enabled": true },
    "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
    "KYC_USER_APPROVED": { "enabled": true },
    "KYC_ADMIN_APPROVED": { "enabled": true },
    "KYC_USER_REJECTED": { "enabled": true },
    "KYC_ADMIN_REJECTED": { "enabled": true }
  }
}
```

**User-only notifications (disable admin notifications):**
```json
{
  "notificationsConfig": {
    "KYC_APPROVAL_REQUEST": { "enabled": false },
    "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
    "KYC_USER_APPROVED": { "enabled": true },
    "KYC_ADMIN_APPROVED": { "enabled": false },
    "KYC_USER_REJECTED": { "enabled": true },
    "KYC_ADMIN_REJECTED": { "enabled": false }
  }
}
```

---

## Input Types Catalog

Complete reference for all 19 supported `DataTypesEnum` input types, with storage format, validation rules, example values, and frontend rendering hints.

### TEXT

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` |
| **Validation** | None |
| **Example value** | `"John Doe"` |
| **Frontend** | Standard text input (`<input type="text">`) |

### EMAIL

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` |
| **Validation** | RFC 5322 email format |
| **Example value** | `"john@example.com"` |
| **Frontend** | Email input (`<input type="email">`) |

### PHONE

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` (single number) or `complexValue` (with `extraPhones`) |
| **Validation** | Phone number format |
| **Example value (simple)** | `"+5511999999999"` |
| **Example value (with extras)** | `{ "phone": "+5511999999999", "extraPhones": ["+5511888888888"] }` |
| **Frontend** | Phone input with country code selector; optional "add phone" button for extras |

### URL

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` |
| **Validation** | Valid HTTP or HTTPS URL |
| **Example value** | `"https://example.com/profile"` |
| **Frontend** | URL input (`<input type="url">`) |

### CPF

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` (digits only, 11 characters) |
| **Validation** | Exactly 11 digits; mod-11 checksum validation; unique per tenant |
| **Sanitization** | Non-numeric characters stripped before storage |
| **Example value** | `"12345678901"` |
| **Frontend** | Masked input with CPF format (`###.###.###-##`); real-time checksum validation |

### BIRTHDATE

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` (ISO date string) |
| **Validation** | Valid date; minimum age check (default: 18 years) |
| **Example value** | `"1990-05-15"` |
| **Frontend** | Date picker with max date set to 18 years ago |

### DATE

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` (ISO date string) |
| **Validation** | Valid ISO date |
| **Example value** | `"2026-01-15"` |
| **Frontend** | Date picker (`<input type="date">`) |

### USER_NAME

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` |
| **Validation** | None |
| **Side effect** | Automatically updates the user's `name` field on their profile |
| **Example value** | `"John Doe"` |
| **Frontend** | Text input; may pre-fill from user profile |

### CHECKBOX

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` (boolean) |
| **Validation** | Boolean value |
| **Example value** | `true` |
| **Frontend** | Checkbox input; label from input definition |

### SIMPLE_SELECT

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` (selected option value) |
| **Validation** | Must match one of `options[].value`; if `isMultiple`, each array item must match |
| **Example value (single)** | `"BR"` |
| **Example value (multiple)** | `["BR", "US"]` |
| **Frontend** | Dropdown select or radio buttons; use multi-select if `isMultiple` is true |

**Options format:**
```json
{
  "options": [
    { "label": "Brazil", "value": "BR" },
    { "label": "United States", "value": "US" },
    { "label": "Portugal", "value": "PT" }
  ]
}
```

### DYNAMIC_SELECT

| Property | Value |
|----------|-------|
| **Storage** | `simpleValue` |
| **Validation** | Options fetched from server at render time |
| **Example value** | `"option-uuid-1"` |
| **Frontend** | Dropdown select with options loaded from an API endpoint |

### FILE

| Property | Value |
|----------|-------|
| **Storage** | `assetId` (reference to Cloudinary upload) |
| **Validation** | Asset must exist |
| **Submission format** | `{ "inputId": "...", "assetId": "asset-uuid" }` |
| **Frontend** | File upload widget; show filename after upload; preview for supported types |

### IMAGE

| Property | Value |
|----------|-------|
| **Storage** | `assetId` (reference to Cloudinary upload) |
| **Validation** | Asset must exist; should be an image MIME type |
| **Submission format** | `{ "inputId": "...", "assetId": "asset-uuid" }` |
| **Frontend** | Image upload widget with preview; drag-and-drop support |

### IDENTIFICATION_DOCUMENT

| Property | Value |
|----------|-------|
| **Storage** | `complexValue` + optional `assetId` |
| **Validation** | `docType` required (one of: `passport`, `cpf`, `rg`); `document` number required |
| **Submission format** | `{ "inputId": "...", "value": { "docType": "rg", "document": "123456789" }, "assetId": "asset-uuid" }` |
| **Frontend** | Document type selector + document number input + file upload for document photo |

**complexValue format:**
```json
{
  "docType": "rg",
  "document": "123456789"
}
```

Supported `docType` values:
| docType | Description |
|---------|-------------|
| `passport` | International passport |
| `cpf` | Brazilian CPF card |
| `rg` | Brazilian RG (identity card) |

### MULTIFACE_SELFIE

| Property | Value |
|----------|-------|
| **Storage** | `assetId` (reference to Cloudinary upload) |
| **Validation** | Asset must exist; CPF and birthdate must be submitted first |
| **Processing** | Triggers biometric validation service call |
| **Submission format** | `{ "inputId": "...", "assetId": "asset-uuid" }` |
| **Frontend** | Camera capture widget; show preview; indicate biometric processing status |

**Prerequisites:** The user must have already submitted documents of type `CPF` and `BIRTHDATE` (in any context within the same tenant) before the multiface selfie can be processed. The biometric service uses the CPF and birthdate for identity matching.

### SIMPLE_LOCATION

| Property | Value |
|----------|-------|
| **Storage** | `complexValue` |
| **Validation** | `placeId`, `region`, `country` are required |
| **Submission format** | `{ "inputId": "...", "value": { "placeId": "...", "region": "SP", "country": "BR", ... } }` |
| **Frontend** | Address autocomplete (Google Places API); manual fallback for region/country |

**complexValue format:**
```json
{
  "placeId": "ChIJrTLr-GyuEmsRBfy61i59si0",
  "region": "SP",
  "country": "BR",
  "city": "Sao Paulo",
  "postal_code": "01000-000",
  "street_address_1": "Av Paulista 1000",
  "street_address_2": "Suite 500",
  "latitude": -23.5505,
  "longitude": -46.6333
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `placeId` | Yes | Google Places place ID |
| `region` | Yes | State/region code |
| `country` | Yes | Country code (ISO 3166-1 alpha-2) |
| `city` | No | City name |
| `postal_code` | No | Postal/ZIP code |
| `street_address_1` | No | Street address line 1 |
| `street_address_2` | No | Street address line 2 |
| `latitude` | No | Latitude coordinate |
| `longitude` | No | Longitude coordinate |

### COMMERCE_PRODUCT

| Property | Value |
|----------|-------|
| **Storage** | `complexValue` |
| **Validation** | `productId` must reference a valid product in the commerce service |
| **Submission format** | `{ "inputId": "...", "value": { "productId": "uuid", "quantity": "1" } }` |
| **Frontend** | Product selector with quantity input; may include variant selection |

**complexValue format:**
```json
{
  "productId": "product-uuid",
  "quantity": "1",
  "variantIds": ["variant-uuid-1", "variant-uuid-2"]
}
```

### SEPARATOR

| Property | Value |
|----------|-------|
| **Storage** | Not stored (skipped during processing) |
| **Validation** | None |
| **Submission format** | Not submitted |
| **Frontend** | Visual divider or section header using the input's `label` and `description` |

### IFRAME

| Property | Value |
|----------|-------|
| **Storage** | Not stored (display-only) |
| **Validation** | None |
| **Submission format** | Not submitted |
| **Frontend** | Render an `<iframe>` element; URL typically provided in the input's `data` or `description` field |

---

## Integration Points

### Commerce Integration

When `blockCommerceDeliver: true` is set on a TenantContext:

1. When a user places a commerce order, the system checks if the user's KYC is approved for the relevant context
2. If KYC is not approved, order delivery is blocked (order is created but held)
3. When KYC is subsequently approved, the commerce service receives a notification event
4. The commerce service unblocks and processes the pending delivery

### Webhook Integration

The platform dispatches a `USER_KYC_PROPS_UPDATED` webhook event when:
- A user's KYC status changes (created, approved, denied, required review)
- Documents are submitted or updated

This webhook can be consumed by external systems for real-time KYC status tracking.

### Email Integration

Email notifications are configurable per notification type on each TenantContext. Emails use the tenant's email templates and respect the user's `i18nLocale` preference for localization.

### Multiface Biometric Service

The multiface selfie input type integrates with an external biometric validation service:

1. User submits a selfie photo (uploaded to Cloudinary)
2. System retrieves the user's CPF and birthdate from previously submitted documents
3. System calls the biometric service with the selfie, CPF, and birthdate
4. Biometric service performs facial recognition and identity matching
5. If validation passes, the document is accepted
6. If validation fails, `ItWasNotPossibleRegisterMutiplaceExpection` is thrown

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | `TenantContextDisabledException` | Context is inactive | Activate the context before users can submit |
| 403 | `ForbiddenException` | Insufficient permissions to view/modify configuration | Requires Admin role |
| 404 | `NotFoundException` | Tenant context not found | Verify `tenantId` and `contextId` |

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Auto-approve enabled but want manual review | Set `autoApprove: false` on the TenantContext. Existing auto-approved submissions are not affected |
| 2 | KycApprover cannot see submissions | Ensure the approver is a member of one of the `approverWhitelistIds` whitelists, or remove the whitelist restriction |
| 3 | Notifications not being sent | Check `notificationsConfig` for the specific notification type. Each type must have `{ "enabled": true }` |
| 4 | Commerce orders not blocking | Set `blockCommerceDeliver: true` on the TenantContext. Only affects orders placed after the setting is changed |
| 5 | Multiface validation failing | Ensure users submit CPF and birthdate documents before the selfie. The biometric service requires these fields |
| 6 | Input type not storing value | `SEPARATOR` and `IFRAME` types are display-only and do not store values. Do not submit documents for these types |
| 7 | requireSpecificApprover but no approver assigned | When `requireSpecificApprover: true`, the submission endpoint requires `approverUserId` in the request body |

---

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| KYC Submission | How users submit documents | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| KYC Approval | How admins review submissions | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Context Lifecycle | Creating and managing contexts | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Input Management | Creating and managing form fields | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Whitelist Management | Managing approver whitelists | [WHITELIST_SKILL_INDEX](../whitelist/WHITELIST_SKILL_INDEX.md) |
| API Reference | Full endpoint and DTO details | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
