---
id: FLOW_AUTH_SIGNUP
title: "Auth - Sign-Up Flows"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - signup
  - registration
depends_on:
  - AUTH_API_REFERENCE
---

# Sign-Up Flows

## Overview

W3Block supports two sign-up flows: registering a user into an **existing tenant** (Flow A) and creating a **new tenant** with the owner as its first admin (Flow B). Flow B is a multi-step process that includes email verification, profile completion, and billing setup.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Base URL | Identity service URL | `https://pixwayid.w3block.io` (prod) |
| `tenantId` | Target tenant UUID (Flow A only) | Environment config or tenant lookup |

---

## Flow A: Register User in Existing Tenant

Single API call. Used when onboarding users to an already-configured tenant.

### Step 1: Create User

```
POST /auth/signup
Content-Type: application/json
```

**Minimal Request:**

```json
{
  "tenantId": "uuid-tenant-id",
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd"
}
```

**Complete Request (production example):**

```json
{
  "tenantId": "uuid-tenant-id",
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd",
  "name": "John Doe",
  "phone": "+5511999998888",
  "i18nLocale": "pt-BR",
  "callbackUrl": "https://myapp.com/auth/verify-email",
  "verificationType": "numeric",
  "referrer": "REFERRAL_CODE",
  "utmParams": {
    "utm_source": "google",
    "utm_medium": "cpc",
    "utm_campaign": "launch"
  }
}
```

**Response (201):** `SignInResponse` — the user receives a JWT immediately, but `data.emailVerified` will be `false` until the email is confirmed.

### Step 2 (Optional): Verify Email

If `callbackUrl` was provided, the user receives an email. For `verificationType: "numeric"`, it contains a 6-digit code; for `"invisible"`, a clickable link.

**To verify with a code:**

```
GET /auth/verify-sign-up?email=user@example.com&token=123456
```

**To resend the verification email/code:**

```
POST /auth/request-confirmation-email
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "verificationType": "numeric"
}
```

**Verification response:**

```json
{
  "verified": true,
  "emailVerified": true,
  "phoneVerified": false
}
```

### Special Case: Passwordless Tenants

When a tenant has `passwordless` mode enabled:
- `password` and `confirmation` fields become **optional**
- A random password is generated server-side
- The user signs in via OTP code or magic token instead
- Email behavior is controlled by `passwordless.sendEmailWhenRegister` tenant config

---

## Flow B: Create New Tenant (Tenant Onboarding)

Multi-step flow that creates both a tenant and its first admin user.

```
Step 1: Register owner    POST /auth/signup/tenant
        ↓
Step 2: Verify email      GET /auth/verify-sign-up
        ↓
Step 3: Complete profile   PATCH /auth/signup/tenant/finish
        ↓
Step 4: Set billing plan   PATCH /billing/{companyId}/plan
        ↓
Step 5: Set payment card   PATCH /billing/{companyId}/credit-card
```

### Step 1: Register Tenant Owner

Creates a dormant user on the system tenant. The user is inactive until Step 3.

```
POST /auth/signup/tenant
Content-Type: application/json
```

**Minimal Request:**

```json
{
  "email": "owner@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd"
}
```

**Complete Request:**

```json
{
  "email": "owner@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd",
  "verificationType": "numeric",
  "name": "John Doe",
  "phone": "+5511999998888"
}
```

**Response (201):** `SignInResponse` — contains a **temporary JWT**. Store this token; it's required for Step 3.

> **Important:** This is a temporary token scoped to the system tenant. The user is inactive and cannot access any tenant resources until Step 3 completes.

### Step 2: Verify Email

Using `verificationType: "numeric"` (recommended for better UX):

```
GET /auth/verify-sign-up?email=owner@example.com&token=123456
```

**Resend code:**

```
POST /auth/request-confirmation-email
Content-Type: application/json
```

```json
{
  "email": "owner@example.com",
  "verificationType": "numeric"
}
```

**Response:** `{ "verified": true, "emailVerified": true, "phoneVerified": false }`

### Step 3: Complete Profile & Create Tenant

Uses the temporary JWT from Step 1.

```
PATCH /auth/signup/tenant/finish
Authorization: Bearer {temporaryToken}
Content-Type: application/json
```

```json
{
  "tenantName": "My Company",
  "ownerName": "John Doe",
  "phone": "+5511999998888"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `tenantName` | string | Yes | Min 3 chars | Company/organization name |
| `ownerName` | string | Yes | Min 5 chars | Owner's full name |
| `phone` | string | Yes | Valid phone | Owner's phone number |

**Response (200):**

```json
{
  "tenant": {
    "id": "uuid-new-tenant-id",
    "name": "My Company"
  },
  "tenantSignIn": {
    "token": "new-access-token-scoped-to-new-tenant...",
    "refreshToken": "new-refresh-token...",
    "data": {
      "sub": "uuid-user-id",
      "tenantId": "uuid-new-tenant-id",
      "roles": ["admin", "user"],
      "type": "user"
    }
  }
}
```

> **Important:** After this step, discard the temporary token and use the new `tenantSignIn.token`. The user is now an active admin of the new tenant.

**Side effects:**
- New tenant entity created
- User migrated from system tenant to new tenant
- User roles set to `[admin, user]`
- User marked as active
- Default tenant configuration dispatched (async)

### Step 4: Set Billing Plan

```
PATCH /billing/{companyId}/plan
Authorization: Bearer {newAccessToken}
Content-Type: application/json
```

```json
{
  "planId": "uuid-plan-id"
}
```

To get available plans:

```
GET /billing/plans
Authorization: Bearer {newAccessToken}
```

### Step 5: Set Payment Card (Stripe)

First tokenize the card via Stripe.js, then send the token to the API:

```
PATCH /billing/{companyId}/credit-card
Authorization: Bearer {newAccessToken}
Content-Type: application/json
```

```json
{
  "tokenId": "tok_1234567890",
  "cardId": "card_1234567890",
  "cardLastNumbers": "4242",
  "expiryMonth": 12,
  "expiryYear": 2028,
  "brand": "Visa"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenId` | string | Yes | Stripe token ID from `stripe.createToken()` |
| `cardId` | string | Yes | Stripe card ID from token response |
| `cardLastNumbers` | string | Yes | Last 4 digits of card |
| `expiryMonth` | number | Yes | Card expiration month |
| `expiryYear` | number | Yes | Card expiration year |
| `brand` | string | Yes | Card brand (Visa, Mastercard, etc.) |

---

## Email Notifications

| Event | Template | Trigger |
|-------|----------|---------|
| Signup confirmation (link) | `signup.html` | `verificationType: "invisible"` |
| Signup confirmation (code) | `signup-code.html` | `verificationType: "numeric"` |
| Invite confirmation | `invite.html` | User created with admin role |
| Complete profile | `default.html` | Default verification type |

Confirmation email URLs include: `?token={token}&code={code}&expire={timestamp}&email={email}&tenantId={tenantId}`.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Validation Error | Invalid email, weak password, or missing required fields | Check field validations |
| 400 | Invalid referral code | `referrer` code doesn't match any user in tenant | Verify referral code |
| 409 | Email already exists | User already registered in this tenant | Direct to sign-in |
| 422 | Unprocessable Entity | `confirmation` doesn't match `password` | Ensure both fields are identical |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Temporary token from Step 1 used for API calls | This token is for Step 3 only. It's scoped to the system tenant and the user is inactive |
| 2 | Skipping email verification before Step 3 | While technically possible, the user's `emailVerified` will be `false`, which may block features |
| 3 | Using `invisible` verification in mobile apps | Use `numeric` instead — deep links to email verification URLs are unreliable in mobile |
| 4 | Password rules failing on passwordless tenants | Check tenant config first. Passwordless tenants auto-generate passwords |
| 5 | `callbackUrl` rejected | The URL host must be in the tenant's allowed hosts list. Contact admin to whitelist your domain |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Sign-In | Sign in after registration | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) |
| Password Reset | If user forgets password | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) |
| OAuth | Alternative registration via Google/Apple | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) |
