---
id: AUTH_SKILL_INDEX
title: "Auth Skill Index"
module: offpix/auth
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
---

# Auth Skill Index

Documentation for the W3Block Identity Service (PixwayID) authentication module. Covers user registration, sign-in (5 methods), password reset, OAuth, token management, and tenant onboarding.

**Service:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [AUTH_API_REFERENCE](./AUTH_API_REFERENCE.md) | 1.0.0 | All auth endpoints, DTOs, enums, schemas | Implemented | API reference at any time |
| 2 | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) | 1.0.0 | Password, OTP code, magic token, tenant API key sign-in | Implemented | Implement any sign-in method |
| 3 | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) | 1.0.0 | User registration + tenant onboarding (multi-step) | Implemented | Implement registration flow |
| 4 | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) | 1.0.0 | Request reset email + set new password | Implemented | Implement password recovery |
| 5 | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) | 1.0.0 | Google & Apple OAuth (redirect + direct token) | Implemented | Implement social sign-in |

---

## Quick Guide

### For a basic auth implementation:

```
1. Read: FLOW_AUTH_SIGNUP.md              → Implement user registration
2. Read: FLOW_AUTH_SIGNIN.md              → Implement sign-in (pick your method)
3. Read: FLOW_AUTH_PASSWORD_RESET.md      → Implement password recovery
4. Consult: AUTH_API_REFERENCE.md         → For schemas, enums, and edge cases
```

### Minimal API-first implementation (3 calls):

```
1. POST /auth/signup          → Register user
2. POST /auth/signin          → Sign in (get JWT)
3. POST /auth/refresh-token   → Keep session alive
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | 409 on sign-in without `tenantId` | Call `GET /auth/user-tenants?email=X` first to resolve tenant |
| 2 | Password validation inconsistent | Tenants with `passwordless` mode have different rules. Check tenant config |
| 3 | Temporary token from tenant signup used for API calls | Only use it for `PATCH /auth/signup/tenant/finish`. Get a real JWT after that |
| 4 | Token refresh fails silently | Monitor for `RefreshAccessTokenError` and force re-authentication |
| 5 | OTP code requires `tenantId` but password sign-in doesn't | Design UX to collect tenant before showing OTP option |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Sign in with email + password | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Flow A |
| Sign in with OTP code | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Flow B |
| Sign in with magic link (admin-generated) | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Flow C |
| Authenticate server-to-server (API key) | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Flow D |
| Register a user in existing tenant | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) — Flow A |
| Create a new tenant + admin user | [FLOW_AUTH_SIGNUP](./FLOW_AUTH_SIGNUP.md) — Flow B |
| Reset a forgotten password | [FLOW_AUTH_PASSWORD_RESET](./FLOW_AUTH_PASSWORD_RESET.md) |
| Add Google sign-in | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) — Google |
| Add Apple sign-in | [FLOW_AUTH_OAUTH](./FLOW_AUTH_OAUTH.md) — Apple |
| Refresh an expiring token | [FLOW_AUTH_SIGNIN](./FLOW_AUTH_SIGNIN.md) — Token Refresh |
| Verify a JWT signature externally | [AUTH_API_REFERENCE](./AUTH_API_REFERENCE.md) — JWKS endpoint |

---

## Matrix: Endpoints x Documents

| Endpoint | Method | Sign-In | Sign-Up | Password Reset | OAuth | API Ref |
|----------|--------|:-------:|:-------:|:--------------:|:-----:|:-------:|
| `/auth/signin` | POST | **X** | | | | **X** |
| `/auth/signin/code` | POST | **X** | | | | **X** |
| `/auth/signin/request-code` | POST | **X** | | | | **X** |
| `/auth/signin/magic-token` | POST | **X** | | | | **X** |
| `/auth/signin/generate-magic-token` | POST | **X** | | | | **X** |
| `/auth/signin/tenant` | POST | **X** | | | | **X** |
| `/auth/signup` | POST | | **X** | | | **X** |
| `/auth/signup/tenant` | POST | | **X** | | | **X** |
| `/auth/signup/tenant/finish` | PATCH | | **X** | | | **X** |
| `/auth/verify-sign-up` | GET | | **X** | | | **X** |
| `/auth/request-confirmation-email` | POST | | **X** | | | **X** |
| `/auth/request-password-reset` | POST | | | **X** | | **X** |
| `/auth/reset-password` | POST | | | **X** | | **X** |
| `/auth/refresh-token` | POST | **X** | | | | **X** |
| `/auth/logout` | POST | **X** | | | | **X** |
| `/auth/user-tenants` | GET | **X** | | **X** | | **X** |
| `/auth/{tenantId}/signin/google` | GET | | | | **X** | **X** |
| `/auth/{tenantId}/signin/google/code` | GET | | | | **X** | **X** |
| `/auth/signin/google` | POST | | | | **X** | **X** |
| `/auth/{tenantId}/signin/apple` | GET | | | | **X** | **X** |
| `/auth/{tenantId}/signin/apple/code` | POST | | | | **X** | **X** |
| `/auth/signin/apple` | POST | | | | **X** | **X** |
| `/auth/jwks.json` | GET | | | | | **X** |
| `/billing/plans` | GET | | **X** | | | **X** |
| `/billing/{id}/plan` | PATCH | | **X** | | | **X** |
| `/billing/{id}/credit-card` | PATCH | | **X** | | | **X** |
| `/billing/{id}/state` | GET | | **X** | | | **X** |

---

## Flow Diagram

```
                         ┌─────────────────────────────────────────────────┐
                         │              W3BLOCK AUTH MODULE                 │
                         │                                                 │
  ┌──────────────┐       │  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
  │  New User    │──────→│  │ Sign-Up  │───→│  Verify  │───→│ Sign-In  │  │
  └──────────────┘       │  │ (A or B) │    │  Email   │    │          │  │
                         │  └──────────┘    └──────────┘    └────┬─────┘  │
  ┌──────────────┐       │                                       │        │
  │ Existing User│──────→│  ┌──────────────────────────────┐     │        │
  └──────────────┘       │  │ Sign-In (5 methods)          │     │        │
                         │  │ • Password                   │─────┤        │
                         │  │ • OTP Code                   │     │        │
  ┌──────────────┐       │  │ • Magic Token                │     │        │
  │ Forgot Pass  │──────→│  │ • Tenant API Key             │     │        │
  └──────────────┘       │  │ • Google / Apple OAuth       │     │        │
                         │  └──────────────────────────────┘     │        │
                         │                                       ▼        │
  ┌──────────────┐       │  ┌──────────┐               ┌──────────────┐  │
  │ Password     │──────→│  │ Request  │──→ email ──→  │ Reset + Auto │  │
  │ Reset        │       │  │ Reset    │               │ Sign-In      │  │
  └──────────────┘       │  └──────────┘               └──────────────┘  │
                         │                                       │        │
                         │                                       ▼        │
                         │                              ┌──────────────┐  │
                         │                              │  JWT Token   │  │
                         │                              │  + Refresh   │  │
                         │                              └──────────────┘  │
                         └─────────────────────────────────────────────────┘
```
