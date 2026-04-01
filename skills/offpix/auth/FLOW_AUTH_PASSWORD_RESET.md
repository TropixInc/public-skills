---
id: FLOW_AUTH_PASSWORD_RESET
title: "Auth - Password Reset Flow"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - password
  - reset
depends_on:
  - AUTH_API_REFERENCE
  - FLOW_AUTH_SIGNIN
---

# Password Reset Flow

## Overview

Two-step flow that sends a reset email with a time-limited token, then allows the user to set a new password and automatically sign in. The user does not need to be authenticated.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Registered email | User must have an existing account | Sign-up flow |
| `tenantId` | Tenant scope (optional, resolves via email) | Tenant lookup |

---

## Flow

```
Step 1: Request reset email    POST /auth/request-password-reset
        ↓ (user clicks email link)
Step 2: Set new password       POST /auth/reset-password
        ↓ (auto sign-in)
        JWT returned
```

### Step 1: Request Password Reset

Optionally resolve the tenant first (same as sign-in):

```
GET /auth/user-tenants?email=user@example.com
```

Then request the reset email:

```
POST /auth/request-password-reset
Content-Type: application/json
```

**Minimal Request:**

```json
{
  "email": "user@example.com"
}
```

**Complete Request (production example):**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "callbackUrl": "https://myapp.com/auth/reset-password"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | User email |
| `tenantId` | UUID | No | Tenant scope (recommended for multi-tenant) |
| `callbackUrl` | URL | No | Your password reset page URL |

**Response (201):** Empty body (email sent)

**Email link format:**

```
{callbackUrl}?email={email}&token={token};{expireTimestamp}
```

Example: `https://myapp.com/auth/reset-password?email=user@example.com&token=abc123;1711900000`

### Step 2: Set New Password

Parse `email` and `token` from the email link's query parameters.

**Client-side expiry check (optional but recommended):**

```javascript
// Token format: "actualToken;unixTimestamp"
const [actualToken, expireStr] = tokenParam.split(';');
const expireMs = parseInt(expireStr) * 1000;
if (Date.now() > expireMs) {
  // Token expired — show "link expired" UI
  // Redirect user to request a new reset email
}
```

**Set the new password:**

```
POST /auth/reset-password
Content-Type: application/json
```

```json
{
  "email": "user@example.com",
  "token": "abc123;1711900000",
  "password": "NewP@ssw0rd",
  "confirmation": "NewP@ssw0rd"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email | User email (from query param) |
| `token` | string | Yes | Verification token | Full token string including expiry (from query param) |
| `password` | string | Yes | 8-32 chars, mixed case + digit | New password |
| `confirmation` | string | Yes | Must match `password` | Password confirmation |

**Response (200):** `SignInResponse` — the user is **automatically signed in** after a successful password reset.

```json
{
  "token": "new-access-token...",
  "refreshToken": "new-refresh-token...",
  "data": {
    "sub": "uuid-user-id",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "roles": ["user"],
    "verified": true,
    "emailVerified": true,
    "type": "user"
  }
}
```

**Side effect:** If the user's email was not yet verified, it is automatically marked as verified.

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Invalid token | Token expired, already used, or malformed | Request a new reset email |
| 400 | Validation Error | Weak password or `confirmation` mismatch | Fix the password fields |
| 404 | User not found | Email not registered | Direct to sign-up |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Token contains semicolon and breaks URL parsing | Use `decodeURIComponent()` when reading from query params. The full string `token;timestamp` must be sent to the API |
| 2 | User clicks expired link | Check the timestamp client-side before calling the API. Show a "request new link" UI instead of an API error |
| 3 | `callbackUrl` not provided in Step 1 | The email won't contain a link. Always include `callbackUrl` pointing to your reset password page |
| 4 | Token works only once | After successful reset, the token is invalidated. A second `POST /auth/reset-password` with the same token will fail |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Sign-In | User can sign in after reset (or use the auto-signed-in JWT) | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) |
| Sign-Up | If user doesn't have an account | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) |
