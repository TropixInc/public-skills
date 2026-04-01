---
id: FLOW_PASS_LIFECYCLE
title: "Pass & Benefits - Pass Lifecycle"
module: offpix/pass
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - pass
  - token-pass
  - benefits
  - create
  - operators
depends_on:
  - PASS_API_REFERENCE
  - FLOW_CONTRACTS_COLLECTION_MANAGEMENT
---

# Pass Lifecycle (Create, Configure, Manage)

## Overview

A Token Pass is an NFT-based access pass created from a published token collection. This flow covers the admin journey: creating a pass, adding benefits (digital and physical), configuring check-in schedules, assigning operators, and managing share codes. In the frontend, this is accessed under **Tokens > Pass** (`/dash/tokens/pass`).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Published collection | Token collection with `status: published` | [Collection Management](../contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |

---

## Flow: Create a Token Pass

### Step 1: Create Pass from Collection

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/token-passes/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Minimal Request:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Pass",
  "name": "VIP Access"
}
```

**Complete Request:**
```json
{
  "collectionId": "collection-uuid",
  "tokenName": "VIP Pass",
  "name": "VIP Access",
  "description": "Exclusive access to premium events and lounges",
  "rules": "Must present QR code at entrance. Valid for token holder only.",
  "imageUrl": "https://cdn.example.com/vip-pass.png"
}
```

**Response (201):**
```json
{
  "id": "pass-uuid",
  "tokenName": "VIP Pass",
  "name": "VIP Access",
  "contractAddress": "0xContract...",
  "chainId": 137,
  "description": "Exclusive access to premium events and lounges",
  "rules": "Must present QR code at entrance.",
  "imageUrl": "https://cdn.example.com/vip-pass.png",
  "tenantId": "tenant-uuid",
  "createdAt": "2026-04-01T10:00:00Z"
}
```

**Notes:**
- The collection must be published with a valid contract address
- The pass ID is derived from the collection ID
- After creation, the frontend prompts to add benefits (Step 2)

### Step 2: Add a Digital Benefit

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/token-pass-benefits/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "name": "Exclusive Video Content",
  "description": "Access to behind-the-scenes footage",
  "tokenPassId": "pass-uuid",
  "type": "digital",
  "eventStartsAt": "2026-06-01T00:00:00Z",
  "linkUrl": "https://example.com/exclusive-content",
  "linkRules": "Link valid for 24 hours after first access"
}
```

### Step 3: Add a Physical Benefit (with Check-In)

**Request:**
```json
{
  "name": "VIP Lounge Access",
  "description": "Access to the VIP lounge",
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
  "tokenPassBenefitAddresses": [
    {
      "name": "Downtown Lounge",
      "street": "Av Paulista",
      "number": 1000,
      "city": "São Paulo",
      "state": "SP",
      "postalCode": "01310-100",
      "rules": "Enter through VIP entrance on the 2nd floor"
    }
  ]
}
```

### Step 4: Assign Operators

Operators are users (Admin or Operator role) who can register benefit uses on behalf of token holders.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/token-pass-benefit-operators/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "benefit-uuid"
}
```

**Response (201):**
```json
{
  "id": "operator-uuid",
  "userId": "operator-user-uuid",
  "tokenPassBenefitId": "benefit-uuid",
  "name": "John Operator",
  "email": "john@example.com",
  "walletAddress": "0xOperator...",
  "associatedTokens": [
    { "id": "pass-uuid", "imageUrl": "...", "tokenName": "VIP Pass" }
  ]
}
```

### Step 5 (Optional): Create Share Code

Share codes allow public access to view pass benefits without authentication.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/token-pass-share-codes/tenants/{tenantId}` | Bearer (Admin) | application/json |

**Request:**
```json
{
  "contractAddress": "0xContract...",
  "chainId": 137,
  "editionNumber": 1,
  "expiresIn": "2026-12-31T23:59:59Z",
  "data": { "campaign": "launch-promo" }
}
```

**Response:** Share code with auto-generated `code` string (uppercase).

---

## Check-In Configuration Patterns

### Daily Schedule (Most Common)

```json
{
  "checkIn": {
    "mon": [{ "start": "09:00", "end": "18:00" }],
    "tue": [{ "start": "09:00", "end": "18:00" }],
    "wed": [{ "start": "09:00", "end": "18:00" }],
    "thu": [{ "start": "09:00", "end": "18:00" }],
    "fri": [{ "start": "09:00", "end": "20:00" }]
  }
}
```

### All Days Same Schedule

```json
{
  "checkIn": {
    "all": [{ "start": "10:00", "end": "22:00" }]
  }
}
```

### Multiple Windows Per Day

```json
{
  "checkIn": {
    "sat": [
      { "start": "10:00", "end": "14:00" },
      { "start": "18:00", "end": "23:00" }
    ]
  }
}
```

### Special Pattern: 2nd Friday of Every Month

```json
{
  "checkIn": {
    "special": [
      {
        "type": "nth_week_day_of_month",
        "start": "19:00",
        "end": "23:00",
        "data": { "nthWeek": 2, "weekday": 5 }
      }
    ]
  }
}
```

---

## Usage Rules

### No Rule (Default)
Total uses counted per edition against `useLimit`. No reset.

### Daily Reset
```json
{ "type": "day_relative", "typeValue": "day" }
```
User can use the benefit `useLimit` times per day.

### Weekly Reset
```json
{ "type": "day_relative", "typeValue": "week" }
```

### Timestamp-Based Reset
```json
{ "type": "timestamp_relative", "typeValue": 3600000 }
```
Resets after 1 hour (3600000 ms).

---

## Token Metadata Requirements

Benefits can require specific token metadata to be eligible:

### Simple Equality
```json
{ "tier": "vip" }
```
Token must have `tokenData.tier === "vip"` (case-insensitive).

### Numeric Comparison
```json
{ "level": { "validator": "gt", "value": "5" } }
```
Token must have `tokenData.level > 5`.

### Time-Based
```json
{ "expiryTimestamp": { "validator": "timestamp_now_relative_secs_gt", "value": 0 } }
```
Token's `expiryTimestamp` must be in the future.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Collection not published | Collection doesn't have a contract address | Publish the collection first |
| 400 | Operator must be Admin or Operator | User doesn't have correct role | Assign appropriate role first |
| 409 | Operator already assigned | Duplicate operator assignment | Skip — operator already has access |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Can't create pass | Collection must be published with a valid contract address |
| 2 | Check-in times not working | Ensure `timezoneOrUtcOffset` matches the location's timezone |
| 3 | Usage doesn't reset | Configure `usageRule` — without it, uses count permanently against `useLimit` |
| 4 | Benefits not showing for user | Check that the user owns the token edition and meets metadata requirements |
| 5 | Dynamic QR expired | Dynamic QR codes are valid for 45 seconds. Regenerate before scanning |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Benefit Redemption | User/operator side of benefits | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Collection Management | Create the collection first | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](../contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Whitelist | Whitelist requirements for benefits | [FLOW_WHITELIST_MANAGEMENT](../whitelist/FLOW_WHITELIST_MANAGEMENT.md) |
