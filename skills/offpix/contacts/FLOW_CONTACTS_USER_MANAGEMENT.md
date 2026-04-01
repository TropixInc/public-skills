---
id: FLOW_CONTACTS_USER_MANAGEMENT
title: "Contacts - User Management"
module: offpix/contacts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contacts
  - users
  - invite
  - roles
  - royalty
depends_on:
  - AUTH_API_REFERENCE
  - CONTACTS_API_REFERENCE
---

# User Management (Contacts)

## Overview

This flow covers managing users within a tenant: listing contacts, inviting new users, editing profiles, managing roles, and toggling royalty eligibility. In the frontend, this is the **Contacts** section (`/dash/contacts/`) with sub-sections for Clients, Team, and Partners.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |

## Entities

```
User
├── name, email, phone
├── roles (UserRoleEnum[])
├── verified (boolean, computed from verifiedEmailAt)
├── mainWallet (vault or metamask address)
├── wallets[] (additional wallets)
└── RoyaltyEligible (optional link)
```

**Contact types** in the frontend are determined by role:
- **Clients** (`/dash/contacts/clients`): Users with `user` role
- **Team** (`/dash/contacts/team`): Users with `admin`, `operator`, `loyaltyOperator`, `commerceOrderReceiver` roles
- **Partners** (`/dash/contacts/partiners`): Associates/partners

---

## Flow A: List Contacts

### Step 1: Fetch Users

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{tenantId}/users` | Bearer (Admin) |

**Minimal Request (clients):**
```
GET /{tenantId}/users?role=user&sortBy=createdAt&orderBy=DESC&limit=10&page=1
```

**Team members:**
```
GET /{tenantId}/users?role=admin&role=operator&role=loyaltyOperator&role=commerceOrderReceiver&sortBy=createdAt&orderBy=DESC
```

**Response (200):**
```json
{
  "items": [
    {
      "id": "user-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+5511999999999",
      "roles": ["user"],
      "verified": true,
      "mainWallet": {
        "id": "wallet-uuid",
        "address": "0x1234...abcd",
        "type": "vault"
      },
      "createdAt": "2026-01-01T00:00:00Z"
    }
  ],
  "meta": {
    "totalItems": 100,
    "totalPages": 10,
    "currentPage": 1,
    "itemsPerPage": 10
  }
}
```

**Frontend display mapping:**

| Column Header | API Field | Notes |
|---------------|-----------|-------|
| Email | `email` | Primary identifier displayed |
| Wallet Address | `mainWallet.address` | Truncated in UI |
| Creation Date | `createdAt` | Formatted date |
| Status | Computed | `verified` + `mainWallet` presence determines: active, verified, inactive, pending |
| KYC Status | `kycStatus` | Only shown if KYC is enabled for the tenant |

---

## Flow B: Invite a New Contact

### Step 1: Invite User

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/users/invite` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "user",
  "tenantId": "tenant-uuid",
  "generateRandomPassword": true,
  "sendEmail": true
}
```

**Complete Request (with royalty eligibility):**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "user",
  "royaltyEligible": true,
  "tenantId": "tenant-uuid",
  "i18nLocale": "pt-br",
  "generateRandomPassword": true,
  "sendEmail": true
}
```

**Response (201):**
```json
{
  "id": "new-user-uuid",
  "email": "jane@example.com",
  "name": "Jane Doe",
  "roles": ["user"],
  "tenantId": "tenant-uuid"
}
```

**Field Reference:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Full name (frontend: "Full Name") |
| `email` | string | Yes | — | Email address (must be unique per tenant) |
| `role` | UserRoleEnum | Yes | — | Role to assign (frontend: "Type") |
| `tenantId` | UUID | Yes | — | Target tenant |
| `i18nLocale` | string | No | — | Locale for invitation email (`pt-br`, `en`) |
| `generateRandomPassword` | boolean | Yes | — | Always `true` |
| `sendEmail` | boolean | Yes | — | Always `true` |

**Notes:**
- The invited user receives an email with a temporary password
- If `royaltyEligible` is true in the frontend, a follow-up call is made to create the royalty record (Step 2)
- Email must be unique per tenant — returns 409 if already registered

### Step 2 (Optional): Create Royalty Eligible Record

If the frontend toggle "Have Royalties" is enabled:

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/contracts/royalty-eligible/create` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "active": true,
  "displayName": "Jane Doe",
  "userId": "new-user-uuid"
}
```

**Note:** This endpoint is on the **Registry backend** (`api.w3block.io`), not PixwayID.

---

## Flow C: Edit a Contact

### Step 1: Fetch Current Profile

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{tenantId}/users/{userId}` | Bearer (Admin) |

### Step 2: Update Profile

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/{tenantId}/users/{userId}` | Bearer (Admin) | application/json |

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
| `name` | string | No | Updated full name |
| `phone` | string | No | Updated phone number |
| `roles` | UserRoleEnum[] | No | Updated role list (frontend: multi-select "Type of User") |

**Frontend editable fields:**

| Frontend Label | API Field | Editable | Notes |
|---------------|-----------|----------|-------|
| Full Name | `name` | Yes | Text input |
| Phone | `phone` | Yes | Phone input (may come from KYC documents) |
| Type of User | `roles` | Yes | Multi-select dropdown |
| Email | `email` | **No** | Read-only |
| Wallet Address | `mainWallet.address` | **No** | Read-only |
| Have Royalties | — | Yes | Separate endpoint (royalty eligible) |

### Step 3 (Optional): Toggle Royalty Eligibility

**To enable:** POST `/{companyId}/contracts/royalty-eligible/create`
**To update:** PATCH `/{companyId}/contracts/royalty-eligible` with `{ "active": true/false, "userId": "..." }`

### Step 4 (Optional): Edit Referral

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PATCH | `/users/{companyId}/{userId}/referrer` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "referrerCode": "ABC123",
  "referrerUserId": "referrer-uuid"
}
```

---

## Flow D: Find User by CPF or Wallet

### By CPF

```
GET /{tenantId}/users/documents/find-user-by-any?cpf=12345678901
```

CPF is looked up in submitted KYC documents. Must be 11 digits with valid checksum.

### By Wallet Address

```
GET /{tenantId}/users/wallets/by-address/0x1234...abcd
```

---

## Flow E: External Contacts (Wallet-Only)

External contacts are wallet addresses without a user account. Used for royalty recipients.

### Create External Contact

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/{companyId}/external-contacts/import` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "name": "External Partner",
  "walletAddress": "0xabcd...1234",
  "email": "partner@example.com",
  "phone": "+5511999999999",
  "royaltyEligible": true,
  "description": "Royalty partner for collection X"
}
```

**Note:** This is on the **Registry backend**. External contacts don't have user accounts and can't log in — they're tracked for royalty/token distribution purposes only.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 409 | Conflict | Email already registered for this tenant | Use existing user or different email |
| 404 | NotFoundException | User not found | Verify userId |
| 403 | ForbiddenException | Insufficient permissions | Requires Admin role |
| 400 | InvalidCPFException | CPF checksum invalid | Verify CPF number |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Invite returns 409 | Email is already registered in the tenant. Search for existing user instead |
| 2 | Phone number from KYC vs profile | User's phone may exist in both profile (`user.phone`) and KYC documents. The frontend reads it from KYC documents (input type `phone`) |
| 3 | Roles must be an array | Even for a single role, send as `["user"]` not `"user"` when editing |
| 4 | Royalty eligible is separate from invite | Inviting a user does NOT automatically create royalty eligibility. It requires a second API call to the Registry backend |
| 5 | No delete user endpoint | Users cannot be deleted — they can only be deactivated or have roles changed |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Auth | Required before any operation | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| KYC Submission | User submits KYC documents | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| KYC Approval | Admin reviews KYC submissions | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Configurations | KYC forms are defined here | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
