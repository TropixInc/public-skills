---
id: AUTH_API_REFERENCE
title: "Auth - API Reference"
module: offpix/auth
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - identity
  - api-reference
---

# Auth API Reference

Complete endpoint reference for the W3Block Identity Service (PixwayID) authentication module.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://pixwayid.w3block.io` |
| Staging | `https://pixwayid.stg.w3block.io` |
| Swagger | https://pixwayid.w3block.io/docs/ |

## Authentication

Endpoints marked **Auth: Bearer** require:

```
Authorization: Bearer {accessToken}
```

Endpoints marked **Auth: None** are public. However, if an `Authorization` header is present on a public endpoint, the token is still validated — enabling optional authentication.

## Rate Limiting

All endpoints are rate-limited. Default: 120 requests/minute. Endpoints with stricter limits are noted individually.

---

## Enums

### UserRoleEnum

| Value | Description |
|-------|-------------|
| `superAdmin` | Platform-level super administrator |
| `admin` | Tenant administrator |
| `operator` | Tenant operator |
| `user` | Regular user |
| `loyaltyOperator` | Loyalty program operator |
| `commerceOrderReceiver` | Commerce order receiver |
| `kycApprover` | KYC approval operator |
| `keyErc20Receiver` | ERC-20 token receiver |

### TenantRoleEnum

| Value | Description |
|-------|-------------|
| `application` | Application-level access |
| `administrator` | Tenant admin role |
| `integration` | Integration/API key access |
| `contextApplication` | Context-scoped application |

### VerificationType

| Value | Description |
|-------|-------------|
| `numeric` | 6-digit numeric code sent via email |
| `invisible` | Token-based verification via email link |

### UserCodeType

| Value | Description |
|-------|-------------|
| `signin` | OTP code for sign-in |
| `magicSignInToken` | UUID magic token for sign-in |
| `loyalty` | Loyalty program code |
| `phone_verification` | Phone number verification |

### JwtType

| Value | Description |
|-------|-------------|
| `user` | User JWT (issued on user sign-in) |
| `tenant` | Tenant JWT (issued on API key sign-in) |

---

## Schemas

### SignInResponse

Returned by all sign-in endpoints on success.

```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6InJlZnJlc2gifQ...",
  "data": {
    "sub": "uuid-user-id",
    "iss": "uuid-tenant-id",
    "aud": ["uuid-tenant-id"],
    "exp": 1711900000,
    "iat": 1711899100,
    "type": "user",
    "tenantId": "uuid-tenant-id",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "verified": true,
    "emailVerified": true,
    "phoneVerified": false
  },
  "profile": {
    "id": "uuid-user-id",
    "email": "user@example.com",
    "name": "John Doe",
    "roles": ["user"],
    "verified": true
  }
}
```

### UserJwtPayload (decoded `token`)

| Field | Type | Description |
|-------|------|-------------|
| `sub` | UUID | User ID |
| `iss` | UUID | Tenant ID (issuer) |
| `aud` | UUID[] | Audience (tenant IDs) |
| `exp` | number | Expiration (unix timestamp) |
| `iat` | number | Issued at (unix timestamp) |
| `type` | `"user"` | Token type |
| `tenantId` | UUID | Tenant ID |
| `email` | string | User email |
| `name` | string? | User name |
| `roles` | UserRoleEnum[] | User roles |
| `verified` | boolean | Whether user is verified |
| `emailVerified` | boolean | Whether email is verified |
| `phoneVerified` | boolean | Whether phone is verified |

### TenantJwtPayload (decoded `token` for tenant sign-in)

Same as UserJwtPayload but with `type: "tenant"` and `roles: TenantRoleEnum[]`.

---

## Endpoints

### Sign-In

#### POST `/auth/signin`

Password-based user sign-in.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 60/min |
| **Content-Type** | application/json |

**Minimal Request:**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd"
}
```

**Complete Request:**

```json
{
  "email": "user@example.com",
  "password": "MyP@ssw0rd",
  "tenantId": "uuid-tenant-id"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email, max 50 chars | User email (case-insensitive) |
| `password` | string | Yes | 8-32 chars | User password |
| `tenantId` | UUID | No | Valid UUID | Tenant scope. If omitted and email exists in multiple tenants, returns 409 |

**Response (200):** `SignInResponse`

**Errors:**

| Status | Cause | Response |
|--------|-------|----------|
| 401 | Invalid credentials | `{ "statusCode": 401, "message": "Unauthorized" }` |
| 409 | Email exists in multiple tenants, no `tenantId` provided | `{ "statusCode": 409, "message": "Conflict", "tenants": [{ "id": "uuid", "name": "Tenant Name" }] }` |

---

#### POST `/auth/signin/code`

OTP code-based sign-in. Requires prior call to `/auth/signin/request-code`.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "email": "user@example.com",
  "code": "123456",
  "tenantId": "uuid-tenant-id"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email, max 50 chars | User email |
| `code` | string | Yes | Exactly 6 characters | OTP code received via email |
| `tenantId` | UUID | No | Valid UUID | Tenant scope |

**Response (200):** `SignInResponse`

**Side effect:** If email is not yet verified, it is automatically marked as verified on successful code sign-in.

---

#### POST `/auth/signin/request-code`

Sends a 6-digit OTP code to the user's email for code-based sign-in.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email | User email |
| `tenantId` | UUID | **Yes** | Valid UUID | Tenant scope (required unlike other endpoints) |

**Response (201):** Empty body (email sent)

---

#### POST `/auth/signin/magic-token`

Sign-in using a UUID magic token (generated by admin/integration).

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "token": "uuid-magic-token",
  "tenantId": "uuid-tenant-id"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `token` | UUID | Yes | Valid UUID | Magic sign-in token |
| `tenantId` | UUID | Yes | Valid UUID | Tenant scope |

**Response (200):** `SignInResponse`

---

#### POST `/auth/signin/generate-magic-token`

Generates a magic sign-in token for a user. Admin/integration only.

| | |
|---|---|
| **Auth** | Bearer (SuperAdmin, Integration) |
| **Content-Type** | application/json |

**Request:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "expirationInMinutes": 30
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email | Target user email |
| `tenantId` | UUID | Yes | Valid UUID | Tenant scope |
| `expirationInMinutes` | number | No | Integer, min 1 | Token TTL (default: 30 min) |

**Response (201):**

```json
{
  "token": "uuid-magic-token",
  "expiresAt": "2026-03-31T12:30:00.000Z"
}
```

---

#### POST `/auth/signin/tenant`

Sign-in using tenant API key and secret. Returns a **tenant JWT** (not a user JWT).

| | |
|---|---|
| **Auth** | None |
| **Content-Type** | application/json |

**Request:**

```json
{
  "key": "api-key-string",
  "secret": "api-secret-string",
  "tenantId": "uuid-tenant-id"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key` | string | Yes | Tenant API key |
| `secret` | string | Yes | Tenant API secret |
| `tenantId` | UUID | Yes | Tenant ID |

**Response (200):** `SignInResponse` (with `data.type = "tenant"` and `data.roles = TenantRoleEnum[]`)

---

### Google OAuth

#### GET `/auth/{tenantId}/signin/google`

Redirects to Google OAuth consent screen.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | UUID | Tenant ID (determines OAuth client config) |

**Response (302):** Redirect to `accounts.google.com`

---

#### GET `/auth/{tenantId}/signin/google/code`

Exchanges a Google authorization code for a W3Block JWT.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `code` | string | Yes | Google authorization code |
| `referrer` | string | No | Referral code |

**Response (200):** `SignInResponse` (with additional `isNewUser: boolean`)

---

#### POST `/auth/signin/google`

Direct sign-in using a Google ID token JWT (for mobile/SPA apps).

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id",
  "referrer": "REFERRAL_CODE"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `credential` | string | Yes | Google ID token (JWT) |
| `tenantId` | UUID | Yes | Tenant ID |
| `referrer` | string | No | Referral code |

**Response (200):** `SignInResponse` (with `isNewUser: boolean`)

---

### Apple OAuth

#### GET `/auth/{tenantId}/signin/apple`

Redirects to Apple OAuth consent screen.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |

**Path Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tenantId` | UUID | Tenant ID |

**Response (302):** Redirect to `appleid.apple.com`

---

#### POST `/auth/{tenantId}/signin/apple/code`

Exchanges an Apple authorization code for a W3Block JWT.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |

**Response (200):** `SignInResponse` (with `isNewUser: boolean`, `isPrivateEmail: boolean`)

---

#### POST `/auth/signin/apple`

Direct sign-in using an Apple ID token (for mobile/SPA apps).

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id"
}
```

Accepts either `credential` or `id_token` field (both are Apple ID token JWTs).

**Response (200):** `SignInResponse` (with `isNewUser: boolean`, `isPrivateEmail: boolean`)

---

### Sign-Up

#### POST `/auth/signup`

Registers a new user within an existing tenant.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 60/min |
| **Content-Type** | application/json |

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
  "callbackUrl": "https://myapp.com/auth/verify",
  "verificationType": "numeric",
  "referrer": "REFERRAL_CODE",
  "utmParams": {
    "utm_source": "google",
    "utm_medium": "cpc",
    "utm_campaign": "launch"
  }
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `tenantId` | UUID | Yes | Valid UUID | Target tenant |
| `email` | string | Yes | Valid email, max 50 chars | User email (lowercased) |
| `password` | string | Conditional | 8-32 chars, mixed case + digit | Required unless tenant is passwordless |
| `confirmation` | string | Conditional | Must match `password` | Password confirmation |
| `name` | string | No | 1-250 chars | User display name |
| `phone` | string | No | Valid phone number | User phone |
| `i18nLocale` | enum | No | `pt-BR` or `en` | Email language (default: `pt-BR`) |
| `callbackUrl` | URL | No | Valid URL, allowed host | Email verification redirect URL |
| `verificationType` | enum | No | `numeric` or `invisible` | Verification method (default: `invisible`) |
| `referrer` | string | No | — | Referral code |
| `utmParams` | object | No | — | UTM tracking parameters |

**Response (201):** `SignInResponse`

**Notes:**
- If tenant has `passwordless` mode enabled, `password` and `confirmation` are optional — a random password is generated server-side.
- If `verificationType` is `numeric`, a 6-digit code is sent instead of a link.
- If tenant config requires a referrer, the `referrer` field becomes mandatory.

---

#### POST `/auth/signup/tenant`

Step 1 of tenant creation. Creates a dormant user on the system tenant.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "email": "owner@example.com",
  "password": "MyP@ssw0rd",
  "confirmation": "MyP@ssw0rd",
  "verificationType": "numeric"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | Owner email |
| `password` | string | Yes | 8-32 chars, mixed case + digit |
| `confirmation` | string | Yes | Must match password |
| `verificationType` | enum | No | Default: `invisible` |
| `name` | string | No | Owner name |
| `phone` | string | No | Phone number |

**Response (201):** `SignInResponse` (temporary JWT — user is inactive until step 2)

---

#### PATCH `/auth/signup/tenant/finish`

Step 2 of tenant creation. Creates the tenant and activates the user.

| | |
|---|---|
| **Auth** | Bearer (temporary token from step 1) |
| **Rate Limit** | 10/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "tenantName": "My Company",
  "ownerName": "John Doe",
  "phone": "+5511999998888"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `tenantName` | string | Yes | Min 3 chars | Company/tenant name |
| `ownerName` | string | Yes | Min 5 chars | Owner full name |
| `phone` | string | Yes | Valid phone | Owner phone |

**Response (200):**

```json
{
  "tenant": {
    "id": "uuid-new-tenant-id",
    "name": "My Company"
  },
  "tenantSignIn": { /* SignInResponse - new JWT scoped to the new tenant */ }
}
```

**Side effects:**
- Creates a new `Tenant` entity
- Migrates the user from system tenant to new tenant
- Sets user roles to `[admin, user]`
- Marks user as active
- Dispatches tenant setup jobs (client creation, default config)

---

### Email Verification

#### GET `/auth/verify-sign-up`

Verifies a user's email using a verification token.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 120/min |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | string | Yes | User email |
| `token` | string | Yes | Verification token (from email link or 6-digit code) |

**Response (200):**

```json
{
  "verified": true,
  "emailVerified": true,
  "phoneVerified": false
}
```

---

#### POST `/auth/request-confirmation-email`

Re-sends the email verification email/code.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 60/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "callbackUrl": "https://myapp.com/auth/verify",
  "verificationType": "numeric"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | User email |
| `tenantId` | UUID | No | Tenant scope |
| `callbackUrl` | URL | No | Redirect URL for email link |
| `verificationType` | enum | No | `numeric` or `invisible` |

**Response (201):** Empty body (email sent)

---

### Password Reset

#### POST `/auth/request-password-reset`

Sends a password reset email to the user.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 60/min |
| **Content-Type** | application/json |

**Minimal Request:**

```json
{
  "email": "user@example.com"
}
```

**Complete Request:**

```json
{
  "email": "user@example.com",
  "tenantId": "uuid-tenant-id",
  "callbackUrl": "https://myapp.com/auth/reset-password",
  "verificationType": "invisible"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | User email |
| `tenantId` | UUID | No | Tenant scope |
| `callbackUrl` | URL | No | Password reset page URL |
| `verificationType` | enum | No | Default: `invisible` |

**Response (201):** Empty body (email sent)

**Note:** The email link contains `?email={email}&token={token};{expireTimestamp}`. Your password reset page should parse these parameters.

---

#### POST `/auth/reset-password`

Consumes the reset token and sets a new password. Returns a JWT (auto sign-in).

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 60/min |
| **Content-Type** | application/json |

**Request:**

```json
{
  "email": "user@example.com",
  "token": "verification-token-string",
  "password": "NewP@ssw0rd",
  "confirmation": "NewP@ssw0rd"
}
```

| Field | Type | Required | Validation | Description |
|-------|------|----------|------------|-------------|
| `email` | string | Yes | Valid email | User email |
| `token` | string | Yes | Verification token | Token from reset email |
| `password` | string | Yes | 8-32 chars, mixed case + digit | New password |
| `confirmation` | string | Yes | Must match `password` | Password confirmation |

**Response (200):** `SignInResponse` (user is automatically signed in)

**Side effect:** Email is marked as verified if it wasn't already.

---

### Token Management

#### POST `/auth/refresh-token`

Exchanges a refresh token for a new access + refresh token pair.

| | |
|---|---|
| **Auth** | None (refresh token in body) |
| **Content-Type** | application/json |

**Request:**

```json
{
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6InJlZnJlc2gifQ..."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `refreshToken` | string | Yes | Valid JWT refresh token |

**Response (200):**

```json
{
  "token": "new-access-token...",
  "refreshToken": "new-refresh-token..."
}
```

**Notes:**
- The old refresh token is blacklisted with a 30-second delay (allowing in-flight requests to complete).
- Refresh tokens have `typ: "refresh"` in the JWT header.
- The refresh token contains a `tokenHash` (SHA-256 of the access token it was issued with).

---

#### POST `/auth/logout`

Blacklists the current access token.

| | |
|---|---|
| **Auth** | Bearer |

**Request:** Empty body. The access token from the `Authorization` header is blacklisted.

**Response (201):** Empty body

**Notes:**
- Token is blacklisted via its SHA-256 hash in the cache.
- Blacklist TTL = remaining seconds until the token's natural expiry.

---

### Tenant Lookup

#### GET `/auth/user-tenants`

Lists tenants where a user has an account. Useful for multi-tenant email resolution.

| | |
|---|---|
| **Auth** | None |
| **Rate Limit** | 60/min |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | string | Yes | User email |

**Response (200):**

```json
{
  "email": "user@example.com",
  "tenants": [
    { "id": "uuid-tenant-1", "name": "Company A" },
    { "id": "uuid-tenant-2", "name": "Company B" }
  ]
}
```

---

### JWKS

#### GET `/auth/jwks.json`

Returns the RSA public key as a JWKS for token verification.

| | |
|---|---|
| **Auth** | None |
| **Cache** | CDN cached for 10 minutes |

**Response (200):**

```json
{
  "keys": [
    {
      "kty": "RSA",
      "n": "...",
      "e": "AQAB",
      "alg": "RS256",
      "use": "sig"
    }
  ]
}
```

**Use case:** External services can verify W3Block JWTs by fetching this endpoint and validating the RS256 signature.
