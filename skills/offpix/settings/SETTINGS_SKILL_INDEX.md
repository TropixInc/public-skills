---
id: SETTINGS_SKILL_INDEX
title: "Settings & Billing Skill Index"
module: offpix/settings
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Settings & Billing Skill Index

Documentation for the W3Block Settings & Billing module. Covers billing plan selection, credit card management, billing cycle history, usage tracking and simulation, feature-gating, subscription cancellation, tenant profile management, and tenant configuration (auth providers, KYC, passwordless, email service, account exclusion).

**Service:** Pixway ID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [SETTINGS_API_REFERENCE](./SETTINGS_API_REFERENCE.md) | 1.0.0 | All 16 endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) | 1.0.0 | Plan selection, credit card, billing cycles, usage summary, simulation, cancellation | Implemented | Manage billing and subscriptions |
| 3 | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) | 1.0.0 | Tenant profile, configurations (auth providers, KYC, passwordless, email), feature checks | Implemented | Configure tenant settings |

---

## Quick Guide

### For billing implementation:

```
1. Read: FLOW_SETTINGS_BILLING_MANAGEMENT.md     -> Plan selection, payments, cycles
2. Consult: SETTINGS_API_REFERENCE.md             -> For enums, DTOs, edge cases
```

### For tenant configuration:

```
1. Read: FLOW_SETTINGS_TENANT_CONFIGURATION.md   -> Profile, auth providers, KYC config
2. Consult: SETTINGS_API_REFERENCE.md             -> For configuration schemas
```

### Minimal billing setup (3 calls):

```
1. GET    /billing/plans                            -> List available plans
2. PATCH  /billing/{tenantId}/credit-card           -> Set payment method
3. PATCH  /billing/{tenantId}/plan                  -> Select a plan
```

### Minimal tenant configuration (2 calls):

```
1. PUT    /tenant/profile/{tenantId}                -> Update tenant profile
2. POST   /tenant/configurations/{tenantId}         -> Set auth/KYC/email config
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Plan change rejected | Ensure a valid credit card is set before changing to a paid plan |
| 2 | Feature check returns false unexpectedly | Verify the tenant's plan includes the feature in `enabledFeatures` |
| 3 | Billing cycle dates wrong after plan change | Cycle dates reset when the plan changes; check `startCycleDate`/`endCycleDate` |
| 4 | Usage limits exceeded but no error | Some limits are soft (tracked only); check `hard*` fields for enforced limits |
| 5 | Configuration update seems to have no effect | Tenant configurations use JSONB; partial updates may require sending the full object |
| 6 | Cannot cancel subscription | Only Admin role can cancel; ensure proper authorization |
| 7 | Currency mismatch in billing | Check `BillingCurrencyExchangeRate` for correct USD conversion rates |
| 8 | Custom limits not applying | `customLimits` on `TenantPlan` are partial overrides; unset fields fall back to plan defaults |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| List available billing plans | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Set up a credit card for billing | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Change my tenant's plan | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| View billing cycle history | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Check usage and costs | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Simulate a billing event cost | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Cancel a subscription | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Register a usage event (integration) | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Trigger manual invoice/charge | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Check if a feature is enabled | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Update tenant profile | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Configure auth providers (Google, Apple) | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Enable passwordless authentication | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Configure KYC settings | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Set up email service | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Get tenant details | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Billing Mgmt | Tenant Config |
|----------|:-------:|:------------:|:-------------:|
| GET /billing/plans | X | X | |
| POST /billing/{tenantId}/register-billing-usage | X | X | |
| GET /billing/{tenantId}/is-feature-enabled/{feature} | X | | X |
| GET /billing/{tenantId}/state | X | X | |
| PATCH /billing/{tenantId}/credit-card | X | X | |
| PATCH /billing/{tenantId}/plan | X | X | |
| PATCH /billing/{tenantId}/cancel | X | X | |
| GET /billing/{tenantId}/cycles | X | X | |
| GET /billing/{tenantId}/billing-summary | X | X | |
| GET /billing/{tenantId}/simulate-billing-event-usage | X | X | |
| GET /billing/{tenantId}/billing-usages | X | X | |
| PATCH /billing/{tenantId}/invoice-and-charge | X | X | |
| POST /tenant/configurations/{tenantId} | X | | X |
| GET /tenant/configurations/{tenantId} | X | | X |
| PUT /tenant/profile/{tenantId} | X | | X |
| GET /tenant/{tenantId} | X | | X |
