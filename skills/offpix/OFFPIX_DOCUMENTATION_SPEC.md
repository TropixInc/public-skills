---
id: OFFPIX_DOCUMENTATION_SPEC
title: "Offpix Documentation Process Spec"
module: offpix
version: "1.0.0"
type: setup
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - process
  - meta
  - documentation
---

# Offpix Documentation Process Spec

This document defines the process for generating W3Block API documentation aimed at **developers and AI-assisted users** who want to consume the W3Block backend to build their own dashboards and applications. Reference this spec every time you start a new documentation task.

---

## 1. Audience & Goal

**Who:** Developers and AI-assisted users building custom frontends on top of the W3Block platform. They have access to the Swagger docs but need guidance on how to orchestrate API calls into working flows.

**What they need:**
- Understand the domain (entities, relationships, business rules)
- Know the correct sequence of API calls for each flow
- Have real-world payloads they can copy and adapt
- Avoid common pitfalls that aren't obvious from Swagger alone

**What they already have:**
- Swagger documentation for all 6 services
- Basic understanding of REST APIs and authentication

**What this documentation is NOT:**
- A copy of the Swagger (don't duplicate endpoint listings without added value)
- Frontend-specific (no React components, hooks, or UI patterns — focus on the API)
- An internal dev guide (assume the reader has no access to the source code)

---

## 2. Source Material

### 2.1 Backend Services (Primary Source)

Use the backend services to understand business logic, validations, entity relationships, and flows that the Swagger doesn't expose.

| Service | Swagger | Framework |
|---------|---------|-----------|
| **ID** (Auth/Identity) | https://pixwayid.w3block.io/docs/ | NestJS + TypeORM |
| **KEY** (Main API / Registry) | https://api.w3block.io/docs/ | NestJS + TypeORM |
| **Commerce** | https://commerce.w3block.io/docs | NestJS + TypeORM |
| **Pass** | https://pass.w3block.io/docs | NestJS + TypeORM |
| **PDF** | https://pdf.w3block.io/docs | NestJS + Puppeteer |

> **Note:** The PDF service is internal and not exposed to end users. However, its Swagger can provide insights into document generation logic and template patterns.

**What to look for in NestJS backend codebases:**
- Module directories — understand domain boundaries and entity relationships
- Entity files — data models, column types, relations, enums
- Service files — business logic, validations, side effects, event chains
- Controller files — endpoint signatures, guards, decorators, request/response DTOs
- DTO files — request/response shapes with validation decorators
- Migrations — schema evolution, understand current state
- Enum files — status values, types, categories

### 2.2 Frontend Reference (Secondary Source — Names, Flows & Hierarchy)

The Offpix frontend serves as a **reference implementation** — proof that API sequences work in production. Beyond tracing calls, it is a critical source for **naming and flow hierarchy**. API field names are often non-obvious or technical; the frontend uses user-friendly labels and a clear navigation hierarchy (what comes before what, parent-child relationships, step ordering) that should be reflected in the documentation. Use it to:

- **Adopt frontend naming** when documenting fields and concepts — if the frontend calls an API field by a different, clearer name, prefer the frontend name in documentation and note the API field name in parentheses
- **Map flow hierarchy** — understand which screens/steps come before others, what the parent-child relationships are, and document flows in the same order the user experiences them
- Trace the actual order of API calls for a given flow
- Extract real payloads being sent (from hooks, mutations, SDK calls)
- Understand which fields are actually required vs. optional in practice
- Identify error handling patterns and edge cases

**Key areas to examine in the frontend:**
- API route enums — all 170+ API route patterns
- API client configuration
- Per-module hooks — data fetching patterns per domain
- Per-module API call implementations
- Auth flow implementation (e.g., NextAuth route handlers)

### 2.3 Existing Documentation (Tertiary Source)

Check the existing skill documentation for already-documented flows. Don't duplicate — reference and extend.

---

## 3. Research Process

Before writing any documentation, follow this sequence:

```
Step 1: Scope
  └─ User tells you which flow/domain to document
  └─ Check if it already exists in the existing skill documentation
  └─ Identify which backend service(s) are involved

Step 2: Backend Deep Dive
  └─ Read the relevant module(s) in the backend repo
  └─ Map entities, relationships, and enums
  └─ Trace the service logic: validations, side effects, event chains
  └─ Read DTOs to understand request/response contracts
  └─ Check for guards/decorators (auth requirements, roles, permissions)

Step 3: Swagger Verification
  └─ Cross-reference backend code with Swagger output
  └─ Note any discrepancies (Swagger can lag behind code)
  └─ Capture exact endpoint paths, methods, and status codes

Step 4: Frontend Trace
  └─ Find how the Offpix frontend consumes these endpoints
  └─ Extract the call sequence (which endpoint first, second, etc.)
  └─ Capture real payloads from hooks/mutations
  └─ Note error handling and edge cases

Step 5: Write
  └─ Follow the output template (Section 4)
  └─ Use progressive payloads (Section 5)
  └─ Apply quality checklist (Section 7)
```

---

## 4. Output Template

Every documentation file follows this structure. Scale each section to its complexity — a simple flow may have a short overview, a complex one may need diagrams.

### 4.1 YAML Frontmatter (Required)

```yaml
---
id: FLOW_OFFPIX_{DOMAIN}_{FLOW_NAME}
title: "{Domain} - {Flow Name}"
module: offpix
version: "1.0.0"
type: flow                    # flow | api-reference | index
status: draft                 # draft | review | implemented | deprecated
last_updated: "YYYY-MM-DD"
authors:
  - rafaelmhp
tags:
  - domain-tag
  - flow-tag
depends_on:
  - DEPENDENCY_SKILL_ID
---
```

### 4.2 Document Body

```markdown
# {Flow Title}

## Overview

Brief description of what this flow accomplishes, when to use it, and what
the end result is. 2-4 sentences max.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| `companyId` | Tenant UUID | Auth flow / environment config |
| Bearer token | JWT access token | ID service authentication |
| ... | ... | ... |

## Entities & Relationships

Describe the domain model relevant to this flow. Include:
- Entity names and their purpose
- Key fields and their types
- Relationships between entities (1:N, N:M, etc.)
- Relevant enums/status values

Use ASCII diagrams for complex relationships:
```
Entity A (1) ──→ (N) Entity B ──→ (N) Entity C
```

## Flow: {Flow Name}

### Step 1: {Action Description}

**Endpoint:**
| Method | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/path/{param}` | Bearer token | application/json |

**Minimal Request:**
```json
{
  "requiredField": "value"
}
```

**Complete Request (production example):**
```json
{
  "requiredField": "value",
  "optionalField": "value",
  "nestedObject": {
    "field": "value"
  }
}
```

**Response (200):**
```json
{
  "id": "uuid",
  "status": "created",
  "...": "..."
}
```

**Field Reference:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `requiredField` | string | Yes | ... |
| `optionalField` | string | No | ... |

**Notes:**
- Business rules, validations, or side effects worth knowing
- What happens internally after this call (events triggered, etc.)

### Step 2: {Next Action}
(repeat pattern)

## Error Handling

| Status | Error Code | Cause | Resolution |
|--------|-----------|-------|------------|
| 400 | VALIDATION_ERROR | Missing required field | Check required fields |
| 409 | CONFLICT | Duplicate entity | ... |
| ... | ... | ... | ... |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Description of gotcha | How to fix/avoid |
| 2 | ... | ... |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Auth | Required before this flow | [FLOW_SIGNUP](../auth/FLOW_SIGNUP.md) |
| ... | ... | ... |
```

### 4.3 API Reference Template

For `*_API_REFERENCE.md` files (one per domain):

```markdown
# {Domain} API Reference

## Base URLs

| Environment | Service | URL |
|-------------|---------|-----|
| Production | {Service} | https://{service}.w3block.io |
| Swagger | {Service} | https://{service}.w3block.io/docs |

## Authentication

All authenticated endpoints require:
```
Authorization: Bearer {accessToken}
```

## Enums

### {EnumName}
| Value | Description |
|-------|-------------|
| `value1` | ... |

## Endpoints

(Group by resource, list all endpoints with method, path, description, auth requirement)
```

### 4.4 Skill Index Template

For `OFFPIX_SKILL_INDEX.md`:

```markdown
# Offpix Skill Index

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [DOC_NAME](./path) | 1.0.0 | ... | Status | ... |

## Quick Guide

(Numbered steps for the most common implementation path)

## Common Pitfalls

(Top issues across all flows)

## Decision Table

(Map use cases to documents)

## Matrix: Endpoints x Documents

(Map endpoints to skills using X marks)
```

---

## 5. Payload Strategy: Progressive Examples

Every endpoint example MUST include two payload versions:

### Minimal (gets the job done)
- Only required fields
- Simplest valid values
- Annotate with: `// required` for each field

### Complete (production-grade)
- All fields the Offpix frontend actually sends
- Real-world values extracted from the frontend codebase
- Annotate optional fields with: `// optional — {purpose}`

**Example:**

```json
// Minimal
{
  "name": "My Token",           // required
  "contractId": "uuid-here"     // required
}

// Complete (based on Offpix production usage)
{
  "name": "My Token",           // required
  "contractId": "uuid-here",    // required
  "description": "A collectible token",  // optional — displayed on storefront
  "imageUrl": "https://...",             // optional — token thumbnail
  "metadata": {                          // optional — on-chain metadata
    "attributes": []
  }
}
```

---

## 6. Language & Naming Conventions

### Language
- **English** for all documentation text
- **W3Block domain terms preserved as-is:** `tenantId`, `companyId`, `tokenPass`, `deliverId`, `passShareCode`, `providersForSelection`, etc.
- Do NOT translate or paraphrase W3Block-specific identifiers

### File & Folder Structure
Each module gets its own subfolder mirroring the frontend module organization:

```
skills/offpix/
├── OFFPIX_DOCUMENTATION_SPEC.md        ← this file
├── OFFPIX_SKILL_INDEX.md               ← global index
├── auth/
│   ├── AUTH_SKILL_INDEX.md
│   ├── AUTH_API_REFERENCE.md
│   └── FLOW_AUTH_*.md
├── commerce/
│   ├── COMMERCE_SKILL_INDEX.md
│   ├── COMMERCE_API_REFERENCE.md
│   └── FLOW_COMMERCE_*.md
├── tokens/
│   └── ...
└── (one folder per module)
```

### File Naming
- Flow documents: `FLOW_{MODULE}_{FLOW_NAME}.md`
- API references: `{MODULE}_API_REFERENCE.md`
- Module index: `{MODULE}_SKILL_INDEX.md`
- Global index: `OFFPIX_SKILL_INDEX.md`

### Frontmatter
- `module: offpix` (always)
- `id:` matches filename without `.md`
- `version:` starts at `1.0.0`, follows SemVer

---

## 7. Quality Checklist

Before marking a document as complete, verify:

| # | Check | Description |
|---|-------|-------------|
| 1 | **Endpoints verified** | Every endpoint path tested against Swagger or backend controller |
| 2 | **Payloads validated** | Minimal payload includes all required fields; complete payload matches frontend usage |
| 3 | **Sequence correct** | Flow steps are in the right order; dependencies are explicit |
| 4 | **Auth documented** | Every endpoint states whether auth is required and what role/permission |
| 5 | **Errors covered** | Common error responses documented with causes and resolutions |
| 6 | **No Swagger duplication** | Document adds value beyond what Swagger already shows (sequence, business rules, gotchas) |
| 7 | **Cross-references** | Related flows linked; prerequisites reference existing docs |
| 8 | **Frontmatter complete** | All required YAML fields present and accurate |
| 9 | **Progressive payloads** | Both minimal and complete examples present |
| 10 | **Pitfalls documented** | At least 2-3 common mistakes identified from backend validations or frontend error handling |

---

## 8. Available Domains

Reference for which domains can be documented and their source mapping:

| Domain | Backend Service | Swagger | Key Entities |
|--------|----------------|---------|-------------|
| **Auth & Identity** | ID Service | pixwayid.w3block.io/docs/ | Users, Tenants, Sessions, Roles |
| **Commerce** | Commerce Service | commerce.w3block.io/docs | Products, Orders, Prices, Discounts, Payments |
| **Tokens & NFTs** | Registry Service | api.w3block.io/docs/ | Contracts, Collections, Editions, Wallets |
| **Tokenization** | Registry Service | api.w3block.io/docs/ | Dynamic tokenization flows |
| **Pass & Benefits** | Pass Service | pass.w3block.io/docs | TokenPasses, Benefits, Operators, Addresses |
| **Loyalty** | Registry Service | api.w3block.io/docs/ | Loyalty contracts, ERC20 transactions |
| **Contacts** | ID Service | pixwayid.w3block.io/docs/ | Contacts, Users, Imports |
| **KYC** | ID Service | pixwayid.w3block.io/docs/ | KYC documents, verification |
| **PDF** | PDF Service | pdf.w3block.io/docs | PDF templates, generation (internal only) |
| **Settings & Billing** | ID Service | pixwayid.w3block.io/docs/ | Plans, Balance, Billing, Emails |
| **Whitelist** | Registry Service | api.w3block.io/docs/ | Whitelist entries |
| **Configuration** | ID Service | pixwayid.w3block.io/docs/ | Tenant contexts, Menu config |

---

## 9. How to Use This Spec

When the user requests documentation for a flow:

1. **Read this spec** to load the process and templates
2. **Identify the domain** from the table in Section 8
3. **Follow the research process** in Section 3 (backend → swagger → frontend)
4. **Write using the output template** in Section 4
5. **Apply progressive payloads** per Section 5
6. **Validate against the quality checklist** in Section 7
7. **Save the file** in `skills/offpix/` with proper naming (Section 6)
8. **Update `OFFPIX_SKILL_INDEX.md`** with the new document entry

**Example user request:**
> "Document the token collection creation flow"

**Your response:**
1. Domain: Tokens & NFTs → Registry Service + `api.w3block.io/docs/`
2. Review the backend module for collection entities, services, DTOs
3. Trace the frontend token hooks for call sequence
4. Write `FLOW_OFFPIX_TOKENS_CREATE_COLLECTION.md`
5. Update `OFFPIX_SKILL_INDEX.md`
