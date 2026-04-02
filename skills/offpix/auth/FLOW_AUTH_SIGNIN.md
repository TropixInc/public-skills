---
id: FLOW_AUTH_SIGNIN
title: "Auth - Sign-In Flows"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - signin
  - login
depends_on:
  - AUTH_API_REFERENCE
---

# Sign-In Flows

## Overview

W3Block supports 5 sign-in methods: password, OTP code, magic token, tenant API key, and Google/Apple OAuth. All methods return the same `SignInResponse` with access and refresh tokens. This document covers the password, code, and magic token flows. OAuth flows are documented separately.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| User account | Registered user in a tenant | [Sign-Up flow](./FLOW_AUTH_SIGNUP.md) |
| `tenantId` | UUID of the target tenant | Tenant lookup or environment config |
| Base URL | Identity service URL | `https://pixwayid.w3block.io` (prod). A staging environment is also available — contact W3Block for details. |

## Entities & Relationships

```
Tenant (1) ──→ (N) User ──→ (N) UserCode (OTP/magic tokens)
                     │
                     └──→ (N) AccessLog (audit trail)
```

A single email can exist across multiple tenants. When `tenantId` is not provided, the API resolves the correct account — or returns a 409 conflict if multiple exist.

---

## Flow A: Password Sign-In

The most common flow. Optionally preceded by a tenant lookup.

### Step 1 (Optional): Resolve Tenant

If the user might have accounts in multiple tenants, resolve first:

```
GET /auth/user-tenants?email=user@example.com
```

**Response:**

```json
{
  "email": "user@example.com",
  "tenants": [
    { "id": "uuid-tenant-1", "name": "Company A" },
    { "id": "uuid-tenant-2", "name": "Company B" }
  ]
}
```

- If **1 tenant**: use its `id` as `tenantId` automatically.
- If **2+ tenants**: present a selection UI, then use the chosen `id`.
- If **0 tenants / 404**: email not registered.

### Step 2: Sign In

```
POST /auth/signin
Content-Type: application/json
```

**Minimal Request:**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd"
}
```

**Complete Request (with tenant):**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "tenantId": "uuid-tenant-id"
}
```

**Response (200):**

```json
{
  "token": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "data": {
    "sub": "uuid-user-id",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "verified": true,
    "emailVerified": true,
    "phoneVerified": false,
    "type": "user"
  }
}
```

### Step 3: Store Tokens

Store `token` (access) and `refreshToken` securely. The access token is used as `Authorization: Bearer {token}` for all authenticated requests.

**Token details:**
- Algorithm: RS256 (asymmetric — verify via `/auth/jwks.json`)
- Access token expiry: configured per-deployment (typically 15 minutes)
- Refresh token expiry: configured per-deployment (longer-lived)
- Refresh token has `typ: "refresh"` in the JWT header

---

## Flow B: OTP Code Sign-In

Two-step flow: request a 6-digit code, then sign in with it.

### Step 1: Request Code

```
POST /auth/signin/request-code
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id"
}
```

> **Note:** `tenantId` is **required** for this endpoint, unlike password sign-in.

**Response (201):** Empty body. A 6-digit code is sent to the user's email.

### Step 2: Sign In With Code

```
POST /auth/signin/code
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "code": "123456",
  "tenantId": "uuid-tenant-id"
}
```

**Response (200):** `SignInResponse` (same as password sign-in)

**Side effect:** If the user's email was not yet verified, it is automatically marked as verified on successful code sign-in.

---

## Flow C: Magic Token Sign-In

Used for admin-initiated sign-in (e.g., impersonation, integration onboarding). Requires an admin or integration JWT to generate the token.

### Step 1: Generate Magic Token (Admin)

```
POST /auth/signin/generate-magic-token
Authorization: Bearer {adminToken}
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "expirationInMinutes": 60
}
```

**Response (201):**

```json
{
  "token": "uuid-magic-token",
  "expiresAt": "2026-03-31T13:00:00.000Z"
}
```

### Step 2: Sign In With Magic Token

```
POST /auth/signin/magic-token
Content-Type: application/json
```

```json
{
  "token": "uuid-magic-token",
  "tenantId": "uuid-tenant-id"
}
```

**Response (200):** `SignInResponse`

---

## Flow D: Tenant API Key Sign-In

For server-to-server integrations. Returns a **tenant JWT** with `TenantRoleEnum` roles instead of a user JWT.

```
POST /auth/signin/tenant
Content-Type: application/json
```

```json
{
  "key": "tenant-api-key",
  "secret": "tenant-api-secret",
  "tenantId": "uuid-tenant-id"
}
```

**Response (200):** `SignInResponse` with `data.type = "tenant"` and `data.roles` containing `TenantRoleEnum` values.

---

## Token Refresh

When the access token approaches expiry, exchange the refresh token for a new pair:

```
POST /auth/refresh-token
Content-Type: application/json
```

```json
{
  "refreshToken": "eyJhbG..."
}
```

**Response (200):**

```json
{
  "token": "new-access-token...",
  "refreshToken": "new-refresh-token..."
}
```

**Important:** The old refresh token is blacklisted with a 30-second delay. Implement token refresh proactively (e.g., at 50% of access token lifetime) rather than waiting for 401 errors.

---

## Logout

Blacklists the current access token:

```
POST /auth/logout
Authorization: Bearer {accessToken}
```

**Response (201):** Empty body.

After logout, the access token is immediately invalid. Any subsequent request using it will receive a 401.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 401 | Unauthorized | Invalid credentials, expired token, or blacklisted token | Check credentials or refresh the token |
| 404 | Not Found | Email not registered in any tenant | Direct user to sign-up |
| 409 | Conflict | Email exists in multiple tenants, no `tenantId` | Use `/auth/user-tenants` to resolve, then include `tenantId` |
| 429 | Too Many Requests | Rate limit exceeded | Back off and retry (check `Retry-After` header) |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | 409 on sign-in without `tenantId` | Always call `/auth/user-tenants` first if you don't know the tenant. If exactly one result, use it automatically |
| 2 | Token refresh fails silently | Watch for `RefreshAccessTokenError` in the refresh response. Force sign-out and re-authenticate |
| 3 | OTP code expired | Codes have a short TTL. Implement a resend button with cooldown (recommended: 60 seconds) |
| 4 | `tenantId` required for `/request-code` but optional elsewhere | This is intentional — OTP codes are tenant-scoped. Always include `tenantId` for code-based flows |
| 5 | Password validation differs per tenant | Tenants with `passwordless` mode don't enforce password rules. Check tenant config if building multi-tenant UIs |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Sign-Up | Create a user account before signing in | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) |
| Password Reset | Reset password before signing in | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) |
| Google/Apple OAuth | Alternative sign-in methods | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) |
| Token Refresh | Maintain session after sign-in | This document (Token Refresh section) |
