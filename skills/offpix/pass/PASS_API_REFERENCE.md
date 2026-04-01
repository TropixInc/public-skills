---
id: PASS_API_REFERENCE
title: "Pass & Benefits - API Reference"
module: offpix/pass
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - pass
  - benefits
  - token-pass
  - operators
  - check-in
  - qr-code
  - api-reference
---

# Pass & Benefits API Reference

Complete endpoint reference for the W3Block Pass & Benefits module. Covers token passes (NFT-based access passes), benefits (digital and physical), operators, addresses, share codes, usage tracking, and QR-based redemption. Served by the dedicated Pass backend.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://pass.w3block.io` |
| Staging | `https://pass.stg.w3block.io` |
| Swagger | https://pass.w3block.io/docs |

## Authentication

All endpoints require Bearer token unless marked as public:

```
Authorization: Bearer {accessToken}
```

All endpoints are scoped to a tenant via `{tenantId}` path parameter.

---

## Enums

### TokenPassBenefitTypeEnum

| Value | Description |
|-------|-------------|
| `digital` | Online/link-based benefit |
| `physical` | Location-based benefit requiring check-in |

### BenefitUseStatusEnum

| Value | Color | Description |
|-------|-------|-------------|
| `active` | Green | Available for use right now |
| `inactive` | Gray | Not yet available (before event start) |
| `unavailable` | Red | Expired or usage limit reached |

### BenefitUsageRuleTypeEnum

| Value | Description |
|-------|-------------|
| `timestamp_relative` | Reset usage count after N milliseconds |
| `day_relative` | Reset daily, weekly, monthly, or yearly |

### TokenPassCheckInKeys

| Value | Description |
|-------|-------------|
| `sun`, `mon`, `tue`, `wed`, `thu`, `fri`, `sat` | Specific day of week |
| `all` | Every day |
| `special` | Custom pattern (weekly repeat, monthly repeat, nth weekday) |

### TokenPassCheckInSpecialType

| Value | Description |
|-------|-------------|
| `all` | Every day pattern |
| `repeat_all_week` | Weekly repeating pattern |
| `repeat_all_month` | Monthly repeating pattern |
| `nth_week_day_of_month` | Specific weekday of month (e.g., 2nd Tuesday) |

---

## Entities & Relationships

```
TokenPass (1) ──→ (N) TokenPassBenefit ──→ (N) TokenPassBenefitAddress
    │                     │                        (physical locations)
    │                     │
    │                     ├──→ (N) TokenPassBenefitOperator
    │                     │        (users who can register check-ins)
    │                     │
    │                     ├──→ (N) TokenPassBenefitUsage
    │                     │        (aggregate: uses per edition)
    │                     │
    │                     └──→ (N) TokenPassBenefitUserUsage
    │                              (detailed: each use with user + timestamp)
    │
    └──→ (N) TokenPassShareCode
               (public codes to access pass benefits)
```

**Key concepts:**
- A **TokenPass** is created from a published token collection — it represents an NFT-based access pass
- **Benefits** are the perks attached to a pass — either digital (links) or physical (locations with check-in)
- **Operators** are Admin/Operator users authorized to register benefit uses on behalf of end users
- **Share codes** provide public access to view pass benefits without authentication
- **Usage tracking** supports per-edition limits, temporal rules (daily/weekly/monthly resets), and check-in windows

---

## Endpoints: Token Passes

Base path: `/token-passes/tenants/{tenantId}`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create token pass from collection |
| 2 | GET | `/` | Bearer | Admin | List all passes (paginated) |
| 3 | GET | `/{id}` | **Public** | — | Get pass details |
| 4 | GET | `/{id}/token-editions/{editionNumber}/benefits` | Bearer | Admin, Operator, User | Get benefits for a specific token edition |
| 5 | GET | `/users/{userId}` | Bearer | Admin, Operator | List passes for operator's user |
| 6 | PATCH | `/{id}` | Bearer | Admin, Operator | Update pass |
| 7 | DELETE | `/{id}` | Bearer | Admin | Delete pass |

### POST /token-passes/tenants/{tenantId}

Create a token pass from a published token collection.

**Minimal Request:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Access Pass",
  "name": "VIP Pass"
}
```

**Complete Request:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Access Pass",
  "name": "VIP Pass",
  "description": "Exclusive access to all premium events",
  "rules": "Must present QR code at entrance. Non-transferable.",
  "imageUrl": "https://cdn.example.com/pass-image.png",
  "tokenPassBenefits": [
    {
      "name": "Main Stage Access",
      "type": "physical",
      "useLimit": 1,
      "eventStartsAt": "2026-06-01T00:00:00Z",
      "eventEndsAt": "2026-06-30T23:59:59Z",
      "dynamicQrCode": true,
      "tokenPassBenefitAddresses": [
        {
          "name": "Main Venue",
          "street": "Av Paulista",
          "number": 1000,
          "city": "São Paulo",
          "state": "SP",
          "postalCode": "01310-100"
        }
      ]
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `collectionId` | UUID | Yes | Published token collection ID (from Key service) |
| `tokenName` | string | Yes | Token name |
| `name` | string | Yes | Pass display name |
| `description` | string | No | Pass description (max 1000 chars) |
| `rules` | string | No | General rules/terms (max 1000 chars) |
| `imageUrl` | URL | No | Pass cover image |
| `tokenPassBenefits` | array | No | Benefits to create with the pass |

**Notes:**
- The collection must be published and have a contract address
- The pass `id` is generated from the collection ID
- Creating a pass updates the collection status in the Key service

### GET /token-passes/tenants/{tenantId}/{id}/token-editions/{editionNumber}/benefits

Returns all benefits for a specific token edition, with computed status (`active`, `inactive`, `unavailable`) and remaining uses.

**Response (200):**
```json
{
  "benefits": [
    {
      "id": "benefit-uuid",
      "name": "Main Stage Access",
      "type": "physical",
      "status": "active",
      "useLimit": 3,
      "usesCount": 1,
      "eventStartsAt": "2026-06-01T00:00:00Z",
      "eventEndsAt": "2026-06-30T23:59:59Z",
      "nextCheckInDates": [
        { "start": "2026-06-15T10:00:00Z", "end": "2026-06-15T22:00:00Z" }
      ],
      "tokenPassBenefitAddresses": [
        { "name": "Main Venue", "city": "São Paulo", "state": "SP" }
      ]
    }
  ]
}
```

---

## Endpoints: Benefits

Base path: `/token-pass-benefits/tenants/{tenantId}`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create benefit |
| 2 | GET | `/` | Bearer | Admin, Operator | List benefits (paginated) |
| 3 | GET | `/usages` | Bearer | Admin, Operator | List all usage records |
| 4 | GET | `/usages/xls` | Bearer | Admin, Operator | Export usages as XLS |
| 5 | GET | `/{id}` | **Public** | — | Get benefit details |
| 6 | GET | `/{id}/{editionNumber}/secret` | Bearer | User | Generate QR secret for check-in |
| 7 | GET | `/{id}/verify` | Bearer | All | Verify benefit availability |
| 8 | PATCH | `/{id}` | Bearer | Admin | Update benefit |
| 9 | DELETE | `/{id}` | Bearer | Admin | Delete benefit |
| 10 | POST | `/{id}/register-use` | Bearer | Admin, Operator | Register use (by operator) |
| 11 | POST | `/{id}/register-use-by-user` | Bearer | Admin, Operator | Register use by userId/CPF |
| 12 | POST | `/register-use-by-qrcode` | Bearer | Admin, Operator | Register use by QR code scan |
| 13 | POST | `/{id}/use` | Bearer | User | User self-registers use |

### POST /token-pass-benefits/tenants/{tenantId}

**Minimal Request (digital):**
```json
{
  "name": "Exclusive Content",
  "tokenPassId": "pass-uuid",
  "type": "digital",
  "eventStartsAt": "2026-06-01T00:00:00Z"
}
```

**Complete Request (physical with check-in):**
```json
{
  "name": "VIP Lounge Access",
  "description": "Access to the VIP lounge during events",
  "rules": "Maximum 2 guests per visit",
  "tokenPassId": "pass-uuid",
  "type": "physical",
  "useLimit": 10,
  "eventStartsAt": "2026-06-01T00:00:00Z",
  "eventEndsAt": "2026-12-31T23:59:59Z",
  "dynamicQrCode": true,
  "allowSelfUse": false,
  "timezoneOrUtcOffset": "America/Sao_Paulo",
  "checkIn": {
    "mon": [{ "start": "10:00", "end": "22:00" }],
    "tue": [{ "start": "10:00", "end": "22:00" }],
    "wed": [{ "start": "10:00", "end": "22:00" }],
    "thu": [{ "start": "10:00", "end": "22:00" }],
    "fri": [{ "start": "10:00", "end": "23:00" }],
    "sat": [{ "start": "12:00", "end": "23:00" }]
  },
  "usageRule": {
    "type": "day_relative",
    "typeValue": "week"
  },
  "requirements": {
    "tier": "vip",
    "minLevel": { "validator": "gt", "value": "5" }
  },
  "requiredWhitelistIds": ["whitelist-uuid"],
  "tokenPassBenefitAddresses": [
    {
      "name": "VIP Lounge Downtown",
      "street": "Av Paulista",
      "number": 1000,
      "city": "São Paulo",
      "state": "SP",
      "postalCode": "01310-100",
      "rules": "Enter through gate B"
    }
  ]
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | — | Benefit name (max 70 chars) |
| `description` | string | No | — | Description (max 1000 chars) |
| `rules` | string | No | — | Rules/terms (max 1000 chars) |
| `tokenPassId` | UUID | Yes | — | Parent token pass |
| `type` | `digital` \| `physical` | No | `digital` | Benefit type |
| `useLimit` | integer | No | — | Max uses per edition (null = unlimited) |
| `eventStartsAt` | ISO datetime | No | — | When benefit becomes available |
| `eventEndsAt` | ISO datetime | No | — | When benefit expires |
| `dynamicQrCode` | boolean | No | `false` | If true, QR codes expire after 45 seconds |
| `allowSelfUse` | boolean | No | `false` | Whether user can redeem own benefit |
| `checkIn` | object | No | — | Day-based check-in windows (see schema below) |
| `usageRule` | object | No | — | Temporal usage limits |
| `requirements` | object | No | — | Token metadata requirements for eligibility |
| `requiredWhitelistIds` | UUID[] | No | — | Whitelist requirements |
| `timezoneOrUtcOffset` | string | No | `America/Sao_Paulo` | Timezone for check-in times |
| `linkUrl` | string | No | — | URL for digital benefits |
| `linkRules` | string | No | — | Rules for link access |
| `tokenPassBenefitAddresses` | array | No | — | Physical locations |

### Check-In Configuration Schema

```json
{
  "mon": [{ "start": "10:00", "end": "22:00" }],
  "fri": [
    { "start": "10:00", "end": "14:00" },
    { "start": "18:00", "end": "23:00" }
  ],
  "special": [
    {
      "type": "nth_week_day_of_month",
      "start": "19:00",
      "end": "23:00",
      "data": { "nthWeek": 2, "weekday": 5 }
    }
  ]
}
```

Each day supports multiple time windows. The `special` key supports recurring patterns.

### Usage Rule Schema

```json
{
  "type": "day_relative",
  "typeValue": "day"
}
```

| type | typeValue | Description |
|------|-----------|-------------|
| `day_relative` | `day` | Reset daily |
| `day_relative` | `week` | Reset weekly |
| `day_relative` | `month` | Reset monthly |
| `day_relative` | `year` | Reset yearly |
| `timestamp_relative` | number (ms) | Reset after N milliseconds |

### Requirements Schema

Simple equality:
```json
{ "tier": "vip", "color": "gold" }
```

Advanced validators:
```json
{
  "level": { "validator": "gt", "value": "5" },
  "expiryTimestamp": { "validator": "timestamp_now_relative_secs_gt", "value": 0 }
}
```

Validators: `gt`, `lt`, `timestamp_now_relative_secs_gt`, `timestamp_now_relative_secs_lt`

### POST /{id}/register-use — Operator Registers Use

**Request:**
```json
{
  "userId": "user-uuid",
  "editionNumber": 42,
  "secret": "sha256-hash-string"
}
```

### POST /{id}/register-use-by-user — Register by User ID or CPF

```json
{
  "userId": "user-uuid"
}
```
or
```json
{
  "cpf": "12345678901"
}
```

### POST /register-use-by-qrcode — Register by QR Code Scan

```json
{
  "qrcode": "42,user-uuid,secret-hash,benefit-uuid"
}
```

QR code format: `{editionNumber},{userId},{secret},{benefitId}`

### GET /{id}/{editionNumber}/secret — Generate QR Secret

Returns a hash for QR code generation. For `dynamicQrCode: true`, the hash expires after 45 seconds.

**Response:**
```json
{
  "secret": "sha256-hash-string"
}
```

**Secret computation:**
- Static: `SHA256(userId + benefitId + editionNumber + secretKey)`
- Dynamic: `SHA256(userId + benefitId + editionNumber + secretKey + expiration)`

---

## Endpoints: Benefit Addresses

Base path: `/token-pass-benefit-addresses/tenants/{tenantId}`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create address |
| 2 | GET | `/` | Bearer | Admin | List addresses (paginated) |
| 3 | GET | `/{id}` | Bearer | Admin | Get address |
| 4 | PATCH | `/{id}` | Bearer | Admin | Update address |
| 5 | DELETE | `/{id}` | Bearer | Admin | Delete address |

### Address Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Location name (max 80 chars) |
| `street` | string | No | Street address |
| `number` | integer | No | Building number |
| `city` | string | Yes | City |
| `state` | string | Yes | State/province code |
| `postalCode` | string | No | Postal code |
| `rules` | string | No | Location-specific rules |
| `tokenPassBenefitId` | UUID | Yes | Parent benefit |

---

## Endpoints: Benefit Operators

Base path: `/token-pass-benefit-operators/tenants/{tenantId}`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Assign operator to benefit |
| 2 | GET | `/` | Bearer | Admin | List all operators (paginated) |
| 3 | GET | `/{id}` | Bearer | Admin, Operator | Get operator details |
| 4 | GET | `/benefits/{benefitId}` | Bearer | Admin, Operator | List operators for benefit |
| 5 | DELETE | `/{id}` | Bearer | Admin | Remove operator |

### POST — Assign Operator

```json
{
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "benefit-uuid"
}
```

The user must have Admin or Operator role. Each user can only be assigned once per benefit.

**Response includes enriched data:** operator name, email, wallet address, and associated token passes.

---

## Endpoints: Share Codes

Base path: `/token-pass-share-codes/tenants/{tenantId}`

| # | Method | Path | Auth | Roles | Description |
|---|--------|------|------|-------|-------------|
| 1 | POST | `/` | Bearer | Admin | Create share code |
| 2 | GET | `/{code}` | **Public** | — | Get benefits by share code |

### POST — Create Share Code

```json
{
  "contractAddress": "0xContractAddress...",
  "chainId": 137,
  "editionNumber": 42,
  "expiresIn": "2026-12-31T23:59:59Z",
  "data": { "campaign": "summer-promo" }
}
```

### GET /{code} — Retrieve Benefits

Returns all benefits for the pass associated with the share code, including token metadata and computed benefit status.

---

## Error Codes

| Code | HTTP | Cause |
|------|------|-------|
| `invalid-secret-hash` | 400 | QR secret doesn't match computed hash |
| `secret-expired` | 400 | Dynamic QR expired (> 45 seconds) |
| `undefined-benefit-start-date` | 400 | Benefit missing `eventStartsAt` |
| `benefit-not-started` | 400 | Current time before `eventStartsAt` |
| `benefit-expired` | 400 | Current time after `eventEndsAt` |
| `temporary-usage-limit-reached` | 400 | Hit usage rule period limit |
| `usage-limit-reached` | 400 | Hit permanent `useLimit` |
| `wrong-checkin-time` | 400 | Not within check-in window |
| `undefined-checkin-times` | 400 | No check-in times configured |
| `invalid-token-owner` | 403 | User doesn't own the token edition |
| `self-use-not-allowed` | 403 | `allowSelfUse: false` and user tried to redeem own benefit |
| `unsatisfied-benefit-requirements` | 403 | Token doesn't meet metadata/whitelist requirements |
| `unauthorized-operator` | 403 | Operator not assigned to this benefit |
| `benefit-not-found` | 404 | Invalid benefit ID |
| `token-not-found` | 404 | Token edition not found |
