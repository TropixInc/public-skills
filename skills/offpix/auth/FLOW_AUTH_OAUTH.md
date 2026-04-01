---
id: FLOW_AUTH_OAUTH
title: "Auth - Google & Apple OAuth Flows"
module: offpix/auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - auth
  - oauth
  - google
  - apple
depends_on:
  - AUTH_API_REFERENCE
  - FLOW_AUTH_SIGNIN
---

# Google & Apple OAuth Flows

## Overview

W3Block supports Google and Apple OAuth as alternative sign-in methods. Both support two integration patterns: **redirect flow** (server-side code exchange) and **direct token flow** (client-side credential submission). Both auto-create user accounts on first sign-in.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `tenantId` | Target tenant UUID | Environment config |
| OAuth configured | Google/Apple OAuth must be configured for the tenant | Admin panel or tenant config API |
| Base URL | Identity service URL | `https://pixwayid.w3block.io` (prod) |

---

## Google OAuth

### Flow A: Redirect (Authorization Code)

Best for web applications. The backend handles the OAuth exchange.

#### Step 1: Redirect to Google

Navigate the user's browser to:

```
GET /auth/{tenantId}/signin/google
```

The backend redirects to Google's consent screen with the correct client ID and scopes configured for the tenant.

#### Step 2: Handle Callback

After the user consents, Google redirects back to your app with a `code` query parameter. Exchange it:

```
GET /auth/{tenantId}/signin/google/code?code={authorizationCode}
```

Optional query parameter: `referrer={referralCode}`

**Response (200):**

```json
{
  "token": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "data": {
    "sub": "uuid-user-id",
    "tenantId": "uuid-tenant-id",
    "email": "user@gmail.com",
    "name": "John Doe",
    "roles": ["user"],
    "type": "user"
  },
  "isNewUser": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `isNewUser` | boolean | `true` if this is the user's first sign-in (account was just created) |

**Use `isNewUser`** to redirect new users to a profile completion or onboarding page.

### Flow B: Direct Token (SPA / Mobile)

For apps that obtain the Google ID token client-side (e.g., via Google Sign-In SDK).

```
POST /auth/signin/google
Content-Type: application/json
```

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id",
  "referrer": "REFERRAL_CODE"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `credential` | string | Yes | Google ID token JWT |
| `tenantId` | UUID | Yes | Target tenant |
| `referrer` | string | No | Referral code |

**Response (200):** Same as redirect flow.

---

## Apple OAuth

### Flow A: Redirect (Authorization Code)

#### Step 1: Redirect to Apple

Navigate the user's browser to:

```
GET /auth/{tenantId}/signin/apple
```

The backend redirects to Apple's consent screen. Apple OAuth configuration is loaded dynamically per tenant.

#### Step 2: Handle Callback

After consent, Apple redirects back. Exchange the code:

```
POST /auth/{tenantId}/signin/apple/code
```

**Response (200):**

```json
{
  "token": "eyJhbG...",
  "refreshToken": "eyJhbG...",
  "data": { ... },
  "isNewUser": true,
  "isPrivateEmail": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `isNewUser` | boolean | `true` if account was just created |
| `isPrivateEmail` | boolean | `true` if the user chose Apple's private relay email |

### Flow B: Direct Token (Mobile)

For apps using Apple Sign-In SDK:

```
POST /auth/signin/apple
Content-Type: application/json
```

```json
{
  "credential": "eyJhbGciOiJSUzI1NiIs...",
  "tenantId": "uuid-tenant-id"
}
```

Accepts either `credential` or `id_token` field — both expect an Apple ID token JWT.

**Response (200):** Same as redirect flow.

---

## Auto-Registration Behavior

Both Google and Apple OAuth auto-create user accounts on first sign-in:

1. Backend verifies the OAuth token/code
2. Extracts email and profile info from the token
3. Checks if a user with this email exists in the target tenant
4. **If exists:** Links the OAuth provider to the existing account and signs in
5. **If not exists:** Creates a new user with the OAuth profile info and signs in

The conflict resolution behavior (step 4) is configurable per-tenant via `thirdPartyAuthConflictResolution`:

| Value | Behavior |
|-------|----------|
| `ALWAYS_UPDATE` | Always update user profile with OAuth data |
| `UPDATE_WHEN_JUST_HAVE_PROVIDER_AUTH` | Only update if user has no password set |
| `IGNORE` | Never update existing profile data |

---

## Apple Private Relay Email

When `isPrivateEmail: true`, the user chose to hide their real email. Apple provides a private relay address like `abc123@privaterelay.appleid.com`. Handle this by:

- Storing the relay email as the user's email (it forwards to their real inbox)
- Not displaying it as a "real" email in your UI
- Optionally prompting the user to provide their real email in profile settings

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Invalid credential | OAuth token is expired, malformed, or from wrong client | Re-authenticate with the OAuth provider |
| 400 | Invalid tenant config | OAuth not configured for this tenant | Configure Google/Apple OAuth in tenant settings |
| 409 | Email conflict | Email already exists with a different auth method | Depends on `thirdPartyAuthConflictResolution` config |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Google OAuth not working for a tenant | Verify the tenant has Google OAuth client ID/secret configured |
| 2 | Apple OAuth requires per-tenant config | Apple requires a private key per app. Each tenant may need its own Apple Developer configuration |
| 3 | `isNewUser` is false on first Google sign-in | If the user already registered with email+password, Google OAuth links to the existing account |
| 4 | Redirect URL mismatch | The callback URL must be registered in Google/Apple developer console |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Sign-In | Alternative to password sign-in | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) |
| Sign-Up | OAuth auto-creates accounts | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) |
