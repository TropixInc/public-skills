---
id: FLOW_PASS_BENEFIT_REDEMPTION
title: "Pass & Benefits - Benefit Redemption"
module: offpix/pass
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - pass
  - benefits
  - redemption
  - qr-code
  - check-in
  - usage
depends_on:
  - PASS_API_REFERENCE
  - FLOW_PASS_LIFECYCLE
---

# Benefit Redemption

## Overview

This flow covers the user and operator sides of benefit redemption: viewing available benefits, generating QR codes, verifying eligibility, and registering uses. There are four redemption methods: operator scan (QR), operator search (userId/CPF), QR code scan, and user self-use.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT (User for self-use, Admin/Operator for registering uses) | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Token ownership | User must own a token edition from the pass collection | Token purchase or mint |
| Configured benefit | Benefit must be active with valid dates | [Pass Lifecycle](./FLOW_PASS_LIFECYCLE.md) |

## Validation Chain

Before a benefit can be used, these checks are performed in order:

```
1. Token ownership → Does user own the edition?
2. Benefit dates → Is current time between eventStartsAt and eventEndsAt?
3. Check-in window → Is current time within a configured check-in slot?
4. Requirements → Does token metadata meet all requirements?
5. Whitelist → Is user in required whitelists?
6. Usage limit → Has the edition's useLimit been reached?
7. Usage rule → Has the temporal limit (daily/weekly/monthly) been reached?
8. Secret validation → Does the QR secret match?
9. Self-use check → If allowSelfUse=false, is the registering user different from the owner?
```

---

## Flow A: View Available Benefits

### Step 1: Get Benefits for Token Edition

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/token-passes/tenants/{tenantId}/{passId}/token-editions/{editionNumber}/benefits` | Bearer (User, Admin, Operator) |

**Response (200):**
```json
{
  "benefits": [
    {
      "id": "benefit-uuid-1",
      "name": "VIP Lounge Access",
      "type": "physical",
      "status": "active",
      "useLimit": 10,
      "usesCount": 3,
      "eventStartsAt": "2026-06-01T00:00:00Z",
      "eventEndsAt": "2026-12-31T23:59:59Z",
      "dynamicQrCode": true,
      "nextCheckInDates": [
        { "start": "2026-04-01T13:00:00Z", "end": "2026-04-01T22:00:00Z" },
        { "start": "2026-04-02T13:00:00Z", "end": "2026-04-02T22:00:00Z" }
      ],
      "tokenPassBenefitAddresses": [
        { "name": "Downtown Lounge", "city": "São Paulo", "state": "SP" }
      ]
    },
    {
      "id": "benefit-uuid-2",
      "name": "Exclusive Content",
      "type": "digital",
      "status": "active",
      "linkUrl": "https://example.com/exclusive"
    }
  ]
}
```

**Status computation:**
- `active` — all checks pass, benefit is usable right now
- `inactive` — current time is before `eventStartsAt`
- `unavailable` — expired or usage limit reached

---

## Flow B: Operator Registers Use (QR Code)

The standard physical check-in flow.

### Step 1: User Generates QR Secret

**Endpoint (User's device):**

| Method | Path | Auth |
|--------|------|------|
| GET | `/token-pass-benefits/tenants/{tenantId}/{benefitId}/{editionNumber}/secret` | Bearer (User) |

**Response:**
```json
{
  "secret": "a1b2c3d4e5f6..."
}
```

For `dynamicQrCode: true`, the secret includes a timestamp and expires after **45 seconds**.

The user's app generates a QR code containing: `{editionNumber},{userId},{secret},{benefitId}`

### Step 2: Operator Scans QR Code

**Endpoint (Operator's device):**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/token-pass-benefits/tenants/{tenantId}/register-use-by-qrcode` | Bearer (Operator) | application/json |

**Request:**
```json
{
  "qrcode": "42,user-uuid,a1b2c3d4e5f6...,benefit-uuid"
}
```

**Response:** 201 Created (use registered)

**What happens internally:**
1. QR code parsed into components
2. Secret validated (SHA256 hash comparison)
3. For dynamic QR: expiration checked (45-second window)
4. All validation chain checks applied
5. Usage record created (`TokenPassBenefitUserUsageEntity`)
6. Aggregate count updated (`TokenPassBenefitUsageEntity`)

---

## Flow C: Operator Registers Use (By User Search)

For situations where QR scanning isn't practical.

### By User ID

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use-by-user
```

```json
{
  "userId": "user-uuid"
}
```

### By CPF

```json
{
  "cpf": "12345678901"
}
```

The system looks up the user by CPF, then verifies token ownership and registers the use.

---

## Flow D: Operator Registers Use (Manual with Secret)

The most explicit method — operator provides all verification data.

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use
```

```json
{
  "userId": "user-uuid",
  "editionNumber": 42,
  "secret": "a1b2c3d4e5f6..."
}
```

---

## Flow E: User Self-Use

Only available when `allowSelfUse: true` on the benefit.

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/use
```

```json
{
  "userId": "user-uuid",
  "editionNumber": 42
}
```

Typically used for digital benefits where no operator is needed.

---

## Flow F: Verify Before Use

Pre-check whether a benefit can be used without actually registering the use.

```
GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/verify
```

**Query:** `?userId={uuid}&editionNumber={number}&secret={hash}`

Returns verification result without side effects.

---

## Flow G: Access via Share Code (Public)

```
GET /token-pass-share-codes/tenants/{tenantId}/{code}
```

No authentication required. Returns pass benefits with token metadata for the specific edition.

---

## Usage Tracking

### View Usage Records

```
GET /token-pass-benefits/tenants/{tenantId}/usages
```

Returns paginated list of all benefit usage records with user details, timestamps, and edition numbers.

### Export Usage Report

```
GET /token-pass-benefits/tenants/{tenantId}/usages/xls
```

Returns XLS file with usage data. Frontend route: `/dash/tokens/pass/usages/{collectionId}`.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | `invalid-secret-hash` | QR secret doesn't match | Regenerate QR code on user's device |
| 400 | `secret-expired` | Dynamic QR > 45 seconds old | Regenerate QR code |
| 400 | `benefit-not-started` | Before `eventStartsAt` | Wait until event starts |
| 400 | `benefit-expired` | After `eventEndsAt` | Benefit is no longer available |
| 400 | `wrong-checkin-time` | Outside check-in window | Check the schedule |
| 400 | `usage-limit-reached` | Hit `useLimit` for this edition | No more uses available |
| 400 | `temporary-usage-limit-reached` | Hit usage rule limit for period | Wait for reset (daily/weekly/monthly) |
| 403 | `invalid-token-owner` | User doesn't own the edition | Verify token ownership |
| 403 | `self-use-not-allowed` | `allowSelfUse: false` | Must be registered by an operator |
| 403 | `unsatisfied-benefit-requirements` | Token metadata doesn't match | Check token requirements |
| 403 | `unauthorized-operator` | Operator not assigned | Assign operator to this benefit |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | QR code rejected after refresh | Dynamic QR codes expire in 45 seconds. Don't refresh too early |
| 2 | Usage count not resetting | Check `usageRule` — without it, counts are permanent |
| 3 | User can't self-use | `allowSelfUse` defaults to `false`. Enable it for digital benefits |
| 4 | Operator can't register use | Operator must be explicitly assigned to the benefit |
| 5 | Benefit shows `unavailable` | Check: event dates, usage limits, check-in windows, and requirements |
| 6 | CPF lookup fails | CPF must exist in the user's KYC documents for the tenant |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Pass Lifecycle | Create and configure passes | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| API Reference | Full endpoint details | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) |
| KYC (for CPF lookup) | CPF from documents | [FLOW_CONTACTS_KYC_SUBMISSION](../contacts/FLOW_CONTACTS_KYC_SUBMISSION.md) |
