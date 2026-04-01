---
id: FLOW_SETTINGS_TENANT_CONFIGURATION
title: "Settings - Tenant Configuration"
module: offpix/settings
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - settings
  - tenant
  - configuration
  - auth-providers
  - kyc
  - passwordless
  - email
depends_on:
  - SETTINGS_SKILL_INDEX
  - SETTINGS_API_REFERENCE
---

# Tenant Configuration

## Overview

This flow covers tenant profile management and configuration of integrations and features: authentication providers (Google OAuth, Apple OAuth, passwordless), KYC settings, email service, push notifications (OneSignal), WhatsApp messaging (Twilio), fraud detection (ClearSale), and automatic account exclusion. It also covers feature-gating via billing plan checks.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `tenantId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token (Admin role) | ID service authentication |
| Provider credentials | API keys/secrets for OAuth, email, etc. | External provider dashboards |

## Entities & Relationships

```
TenantEntity (1) -----> (1) TenantConfigurationsEntity
     |
     +-----> (1) TenantPlanEntity -----> (1) BillingPlanEntity
                                               |
                                               +-- enabledFeatures: BillingFeature[]
```

**Key concepts:**
- Each tenant has exactly one TenantConfigurationsEntity (created on first POST)
- Configuration fields are independent -- you can update one without affecting others
- Feature availability depends on the tenant's billing plan (`enabledFeatures`)
- TenantEntity holds the profile (name, document, country); TenantConfigurationsEntity holds integrations

---

## Flow: Tenant Profile Management

### Step 1: Get Tenant Details

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/tenant/{tenantId}` | Bearer token (Admin) | - |

**Response (200):**
```json
{
  "id": "tenant-uuid",
  "name": "My Company LLC",
  "document": "12345678000190",
  "countryCode": "BR",
  "wallets": [
    {
      "address": "0x1234...abcd",
      "type": "operator"
    }
  ],
  "clientId": "client-uuid",
  "info": {
    "website": "https://example.com",
    "industry": "retail"
  },
  "roles": ["admin"],
  "operatorAddress": "0x1234...abcd",
  "setupDone": true
}
```

**Notes:**
- `wallets` contains blockchain wallets associated with the tenant
- `setupDone` indicates whether the tenant has completed initial onboarding
- `operatorAddress` is the default blockchain operator for transactions

### Step 2: Update Tenant Profile

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| PUT | `/tenant/profile/{tenantId}` | Bearer token (Admin) | application/json |

**Minimal Request:**
```json
{
  "name": "My Company"            // required
}
```

**Complete Request:**
```json
{
  "name": "My Company LLC",       // required
  "document": "12345678000190",   // optional -- business registration (CNPJ, EIN, etc.)
  "countryCode": "BR",           // optional -- ISO 3166-1 alpha-2
  "info": {                       // optional -- freeform metadata
    "website": "https://example.com",
    "industry": "retail",
    "contactEmail": "admin@example.com"
  }
}
```

**Response (200):** Updated TenantEntity.

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Tenant display name |
| `document` | string | No | Business registration number |
| `countryCode` | string | No | ISO country code |
| `info` | object | No | Freeform metadata object |

**Notes:**
- This is a PUT (full replace), not a PATCH -- send all fields you want to keep
- `wallets`, `roles`, and `operatorAddress` are not modifiable through this endpoint

---

## Flow: Tenant Configuration (Integrations)

### Step 3: Get Current Configuration

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/tenant/configurations/{tenantId}` | Bearer token (Admin) | - |

**Response (200):**
```json
{
  "tenantId": "tenant-uuid",
  "kyc": null,
  "signUp": null,
  "passwordless": { "enabled": false },
  "googleOAuth": null,
  "appleOAuth": null,
  "oneSignal": null,
  "twilioWhatsApp": null,
  "clearSale": null,
  "emailService": null,
  "automaticAccountExclusion": false
}
```

**Notes:**
- `null` fields mean the integration is not configured
- Always fetch current config before updating to understand the current state

### Step 4: Update Configuration

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/tenant/configurations/{tenantId}` | Bearer token (Admin) | application/json |

This endpoint uses upsert behavior: creates the configuration on first call, updates on subsequent calls.

#### 4a: Enable Passwordless Authentication

**Minimal Request:**
```json
{
  "passwordless": {
    "enabled": true
  }
}
```

#### 4b: Configure Google OAuth

**Request:**
```json
{
  "googleOAuth": {
    "clientId": "123456789-abc.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxxxxxxxxx",
    "enabled": true
  }
}
```

#### 4c: Configure Apple OAuth

**Request:**
```json
{
  "appleOAuth": {
    "clientId": "com.example.app",
    "teamId": "ABC123DEF4",
    "keyId": "KEY123",
    "enabled": true
  }
}
```

#### 4d: Configure KYC

**Request:**
```json
{
  "kyc": {
    "provider": "clear_sale",
    "enabled": true,
    "requiredForPurchase": false
  }
}
```

#### 4e: Configure Email Service

**Request:**
```json
{
  "emailService": {
    "provider": "sendgrid",
    "apiKey": "SG.xxxxxxxxxxxx",
    "fromEmail": "noreply@example.com",
    "fromName": "My Company"
  }
}
```

#### 4f: Configure Push Notifications (OneSignal)

**Request:**
```json
{
  "oneSignal": {
    "appId": "onesignal-app-uuid",
    "apiKey": "onesignal-rest-api-key"
  }
}
```

#### 4g: Configure WhatsApp (Twilio)

**Request:**
```json
{
  "twilioWhatsApp": {
    "accountSid": "ACxxxxxxxxxxxx",
    "authToken": "auth-token-here",
    "fromNumber": "+15551234567"
  }
}
```

#### 4h: Configure Fraud Detection (ClearSale)

**Request:**
```json
{
  "clearSale": {
    "enabled": true,
    "apiKey": "clearsale-api-key"
  }
}
```

#### 4i: Enable Automatic Account Exclusion

**Request:**
```json
{
  "automaticAccountExclusion": true
}
```

#### Complete Configuration (all fields):

```json
{
  "kyc": {
    "provider": "clear_sale",
    "enabled": true,
    "requiredForPurchase": false
  },
  "signUp": {
    "requireEmailVerification": true,
    "allowedDomains": ["example.com"]
  },
  "passwordless": {
    "enabled": true
  },
  "googleOAuth": {
    "clientId": "123456789-abc.apps.googleusercontent.com",
    "clientSecret": "GOCSPX-xxxxxxxxxxxx",
    "enabled": true
  },
  "appleOAuth": {
    "clientId": "com.example.app",
    "teamId": "ABC123DEF4",
    "keyId": "KEY123",
    "enabled": true
  },
  "oneSignal": {
    "appId": "onesignal-app-uuid",
    "apiKey": "onesignal-rest-api-key"
  },
  "twilioWhatsApp": {
    "accountSid": "ACxxxxxxxxxxxx",
    "authToken": "auth-token-here",
    "fromNumber": "+15551234567"
  },
  "clearSale": {
    "enabled": true,
    "apiKey": "clearsale-api-key"
  },
  "emailService": {
    "provider": "sendgrid",
    "apiKey": "SG.xxxxxxxxxxxx",
    "fromEmail": "noreply@example.com",
    "fromName": "My Company"
  },
  "automaticAccountExclusion": false
}
```

**Response (201):** Created/updated TenantConfigurationsEntity.

**Notes:**
- You can send a partial payload with only the fields you want to update
- JSONB fields (`kyc`, `signUp`) are stored as-is -- the backend does not merge nested objects
- Setting a field to `null` disables that integration
- Sensitive fields (API keys, secrets) are stored encrypted

---

## Flow: Feature Gating

### Step 5: Check if a Feature is Enabled

Use this endpoint to gate UI elements or operations based on the tenant's billing plan.

**Endpoint:**

| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| GET | `/billing/{tenantId}/is-feature-enabled/{feature}` | Bearer token (Admin/User/Operator) | - |

**Example Request:**
```
GET /billing/tenant-uuid/is-feature-enabled/COMMERCE
```

**Response (200):**
```json
{
  "enabled": true
}
```

**Common Feature Checks:**

| Feature | Use Case |
|---------|----------|
| `COMMERCE` | Show/hide storefront and product management |
| `LOYALTY` | Show/hide loyalty program features |
| `KYC` | Enable/disable KYC verification flows |
| `MINT_ERC_721` | Allow NFT minting |
| `MINT_ERC_20` | Allow ERC20 minting |
| `WHITE_LABEL` | Enable white-label branding options |
| `CUSTOM_DOMAIN` | Show custom domain settings |
| `FULL_TOKEN_PASS` | Enable advanced token pass features |
| `CUSTODIAL_WALLET` | Enable custodial wallet management |
| `EXTERNAL_ECOMMERCE_INTEGRATION` | Show external e-commerce integration options |

**Notes:**
- This endpoint is available to Admin, User, and Operator roles -- not just admins
- The check is against `BillingPlanEntity.features.enabledFeatures` for the tenant's current plan
- Custom limits on TenantPlanEntity do not affect feature availability
- Use this as a guard before showing feature-specific UI or making feature-specific API calls

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | VALIDATION_ERROR | Invalid field values (bad JSON, wrong types) | Validate payload structure before sending |
| 401 | UNAUTHORIZED | Missing or expired token | Re-authenticate |
| 403 | FORBIDDEN | Non-admin trying to update config/profile | Ensure Admin role |
| 404 | TENANT_NOT_FOUND | Invalid `tenantId` | Verify tenant exists |
| 404 | CONFIGURATION_NOT_FOUND | GET config before any POST (no config exists yet) | Create config first with POST |
| 422 | INVALID_FEATURE | Unknown feature name in is-feature-enabled | Check BillingFeature enum values |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | PUT profile erases fields not included | PUT is a full replace; always include all fields you want to keep |
| 2 | OAuth not working after config update | Verify `enabled: true` is set alongside the credentials |
| 3 | Feature check returns false for a configured feature | Feature availability depends on the billing plan, not on configuration. Upgrade the plan if needed |
| 4 | KYC config saved but KYC flow not appearing | Check that the `KYC` billing feature is enabled on the plan AND that `kyc.enabled` is `true` in config |
| 5 | GET configurations returns 404 | No configuration exists yet; POST first to create one |
| 6 | Partial config update overwrites JSONB fields | JSONB fields (kyc, signUp) are replaced entirely; send the complete object, not just changed subfields |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Authentication | Required for all endpoints; OAuth config affects auth flows | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Billing Management | Plan determines available features | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| KYC Submission | KYC config controls verification flows | [FLOW_CONTACTS_KYC_SUBMISSION](../contacts/FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Configurations | Context/form config complements tenant config | [CONFIGURATIONS_SKILL_INDEX](../configurations/CONFIGURATIONS_SKILL_INDEX.md) |
