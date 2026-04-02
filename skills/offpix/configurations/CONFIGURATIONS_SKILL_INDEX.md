---
id: CONFIGURATIONS_SKILL_INDEX
title: "Configurations Skill Index"
module: offpix/configurations
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
---

# Configurations Skill Index

Documentation for the W3Block Configurations module. Covers dynamic form management (contexts, tenant contexts, and form fields/inputs) used for KYC, signup, surveys, and custom data collection.

**Service:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [CONFIGURATIONS_API_REFERENCE](./CONFIGURATIONS_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, enums, entity relationships | Implemented | API reference at any time |
| 2 | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) | 1.0.0 | Create, update, list, and configure forms (contexts) | Implemented | Build form management features |
| 3 | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) | 1.0.0 | Add, update, and retrieve form fields (19 input types) | Implemented | Add or manage form fields |
| 4 | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) | 1.0.0 | Toggle signup registration on/off, configure signup form | Implemented | Enable/disable user registration |

---

## Quick Guide

### For a basic form implementation:

```
1. Read: FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md   → Create a form (context)
2. Read: FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md    → Add fields to the form
3. Read: FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md   → Enable signup (if applicable)
4. Consult: CONFIGURATIONS_API_REFERENCE.md         → For enums, DTOs, edge cases
```

### Minimal API-first implementation (3 calls):

```
1. POST /tenants/{id}/admin/tenant-context    → Create form + tenant config
2. POST /tenants/{id}/tenant-input            → Add fields (repeat per field)
3. GET  /tenants/{id}/tenant-input/slug/{s}   → Render form to users (public)
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Slug validation fails | Must match `^[a-z0-9]+(?:-[a-z0-9]+)*$` (lowercase, no spaces/underscores) |
| 2 | Wrong PATCH endpoint | Admin PATCH updates context + config; regular PATCH only updates `active` and `data` |
| 3 | Adding mandatory field triggers user reviews | Plan form structure before creating inputs to avoid mass notification |
| 4 | `attributeName` collision | Always set custom `attributeName` when using multiple fields of the same type |
| 5 | `type` immutable after creation | Delete and recreate the input if you need a different type |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Create a new KYC/survey form | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Add fields to a form | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Enable/disable user signup | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) |
| Render a form to end users | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) (public slug endpoint) |
| List available form types and field types | [CONFIGURATIONS_API_REFERENCE](./CONFIGURATIONS_API_REFERENCE.md) (enums section) |
| Configure KYC approval workflow | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) (data schema) |
| Build multi-step forms | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) (step field) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Context Lifecycle | Input Mgmt | Signup Activation |
|----------|:-------:|:-----------------:|:----------:|:-----------------:|
| POST /contexts | X | | | |
| GET /contexts | X | | | |
| PATCH /contexts/:id | X | | | |
| DELETE /contexts/:id | X | | | |
| POST /tenants/:id/tenant-context | X | X | | |
| GET /tenants/:id/tenant-context | X | X | | X |
| GET /tenants/:id/tenant-context/:id | X | X | | |
| GET /tenants/:id/tenant-context/slug/:slug | X | X | | |
| PATCH /tenants/:id/tenant-context/:id | X | X | | X |
| GET /tenants/:id/tenant-context/:id/approvers | X | X | | |
| POST /tenants/:id/admin/tenant-context | X | X | | X |
| PATCH /tenants/:id/admin/tenant-context/:id | X | X | | |
| GET /tenants/:id/admin/tenant-context/:id/inputs | X | | X | |
| POST /tenants/:id/tenant-input | X | | X | |
| PATCH /tenants/:id/tenant-input/:id | X | | X | |
| GET /tenants/:id/tenant-input/slug/:slug | X | | X | |
| GET /tenants/:id/tenant-input | X | | X | |
| GET /tenants/:id/tenant-input/:id | X | | X | |
| GET /tenant-contexts/activate-signup/:id | | | | X |
| GET /tenant-contexts/deactivate-signup/:id | | | | X |
| GET /data-types/:id | X | | | X |
