---
id: PDF_API_REFERENCE
title: "PDF - API Reference"
module: offpix/pdf
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - pdf
  - certificate
  - pass
  - puppeteer
  - handlebars
  - api-reference
---

# PDF API Reference

Complete endpoint reference for the W3Block PDF module (w3block-pdf). This is an **internal service** built with NestJS, Puppeteer, and Handlebars that generates PDF documents from HTML templates. It serves two primary use cases: blockchain token certificates and pass/ticket PDFs with QR codes.

> **Internal service:** All endpoints are unauthenticated. The service is designed to run behind a firewall or VPN and should never be exposed directly to the public internet. Developers building custom applications can call these endpoints from their backend services.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://pdf.w3block.io` |
| Swagger | https://pdf.w3block.io/docs |

## Authentication

**None required.** This is an internal service. All endpoints are open and do not require any authentication headers. Access control is enforced at the network/infrastructure level (firewall, VPN, service mesh).

---

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `KEY_URL` | Base URL for the KEY metadata service (used to fetch token metadata) | `https://api.w3block.io` |
| `GLOBAL_TIMEOUT` | Request timeout in milliseconds | `30000` (30 seconds) |

> **Timeout:** All requests are subject to a global 30-second timeout. PDF generation for complex templates with many images may exceed this limit. Optimize templates to stay within the timeout window.

---

## Endpoints

### Certificate Endpoints

#### GET /certification/{address}/{chainId}/{tokenId}

Generate a PDF certificate for a specific blockchain token by its token ID.

| Method | Path | Auth |
|--------|------|------|
| GET | `/certification/{address}/{chainId}/{tokenId}` | None |

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `address` | string | Yes | Ethereum contract address (hex, e.g., `0x1234...abcd`) |
| `chainId` | number | Yes | Blockchain chain ID (e.g., `137` for Polygon, `80001` for Mumbai) |
| `tokenId` | number | Yes | Token ID within the contract |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `preview` | any | No | If present (any value), returns PDF inline for browser display instead of as a download attachment |

**Response:**

| Status | Content-Type | Description |
|--------|-------------|-------------|
| 200 | `application/pdf` | PDF stream. Without `preview`, includes `Content-Disposition: attachment` header |
| 404 | `application/json` | Token not found at the given address/chainId/tokenId |
| 500 | `application/json` | PDF generation failed (template error, Puppeteer crash) |
| 408 | `application/json` | Request exceeded 30-second timeout |

**Internal Flow:**

1. Service calls KEY metadata endpoint: `GET {KEY_URL}/metadata/address/{address}/{chainId}/{tokenId}`
2. Extracts `information` (title, description, mainImage), `certificateLink`, token details, and edition data
3. If `mainImage` is a Cloudinary video URL, converts it to a PNG thumbnail (replaces file extension)
4. Renders the certificate HTML template with Handlebars using the metadata context
5. Uses Puppeteer to convert rendered HTML to PDF
6. Returns the PDF stream

**Example Request:**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0x1234567890abcdef1234567890abcdef12345678/137/42"
```

**Example Request (inline preview):**

```bash
curl "https://pdf.w3block.io/certification/0x1234567890abcdef1234567890abcdef12345678/137/42?preview=true"
```

---

#### GET /certification/{address}/{chainId}/q?

Generate a PDF certificate by querying token metadata with custom filters. Returns a certificate for the first matching token.

| Method | Path | Auth |
|--------|------|------|
| GET | `/certification/{address}/{chainId}/q` | None |

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `address` | string | Yes | Ethereum contract address (hex) |
| `chainId` | number | Yes | Blockchain chain ID |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `preview` | any | No | If present, returns PDF inline for browser display |
| (custom filters) | any | No | Any additional query parameters are treated as metadata filter key-value pairs |

**Response:**

| Status | Content-Type | Description |
|--------|-------------|-------------|
| 200 | `application/pdf` | PDF stream for the first matching token |
| 404 | `application/json` | No token matches the provided metadata filters |
| 500 | `application/json` | PDF generation failed |
| 408 | `application/json` | Request exceeded 30-second timeout |

**Internal Flow:**

1. Service calls KEY metadata endpoint with query parameters as filters
2. Returns the first matching token's metadata
3. If no match is found, throws `NotFoundException`
4. Generates certificate PDF from the matched token metadata

**Example Request:**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0x1234567890abcdef1234567890abcdef12345678/137/q?rarity=legendary&edition=1"
```

---

#### POST /certification/preview

Generate a preview PDF from a custom HTML template. Handlebars delimiters are escaped so variables render as literal text (e.g., `{{information.title}}` appears as-is in the PDF).

| Method | Path | Auth |
|--------|------|------|
| POST | `/certification/preview` | None |

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pdfTemplate` | string | Yes | Full HTML string with Handlebars syntax for the certificate template |

**Minimal Request:**

```json
{
  "pdfTemplate": "<html><body><h1>{{information.title}}</h1><p>{{information.description}}</p><img src='{{information.mainImage}}' /><p>Token: {{token.address}} #{{token.tokenId}}</p></body></html>"
}
```

**Response:**

| Status | Content-Type | Description |
|--------|-------------|-------------|
| 200 | `application/pdf` | PDF stream with escaped Handlebars variables shown as literal text |
| 400 | `application/json` | Invalid request body (missing `pdfTemplate`) |
| 500 | `application/json` | PDF generation failed |

**Notes:**
- The preview escapes all Handlebars delimiters (`{{` becomes `\{{`) so that variables are displayed as placeholder text rather than rendered with empty data
- Useful for template designers to verify layout and styling before deploying a template

**Example Request:**

```bash
curl -X POST "https://pdf.w3block.io/certification/preview" \
  -H "Content-Type: application/json" \
  -d '{"pdfTemplate": "<html><body><h1>{{information.title}}</h1></body></html>"}'  \
  -o preview.pdf
```

---

### Pass Endpoints

#### POST /pass/generateTickets

Generate a pass/ticket PDF containing token information, variant details, and benefits with QR codes.

| Method | Path | Auth |
|--------|------|------|
| POST | `/pass/generateTickets` | None |

**Request Body (PassDto):**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenName` | string | Yes | Display name of the token pass |
| `tokenEditionNumber` | string | Yes | Edition number or identifier |
| `tokenVariants` | TokenVariantsDto[] | Yes | Array of variant label/value pairs (can be empty `[]`) |
| `benefits` | BenefitsDto[] | Yes | Array of benefits to include in the pass |
| `pdfTemplate` | string | No | Custom HTML template with Handlebars syntax. If omitted, the built-in template is used |

**Minimal Request:**

```json
{
  "tokenName": "VIP Concert Pass",
  "tokenEditionNumber": "42",
  "tokenVariants": [],
  "benefits": [
    {
      "name": "General Admission",
      "description": "Access to the main event area",
      "rules": "Valid for one entry only. No re-entry allowed.",
      "startDate": "2026-06-15T18:00:00.000Z",
      "qrCode": "https://example.com/redeem/benefit-uuid-1234"
    }
  ]
}
```

**Complete Request:**

```json
{
  "tokenName": "VIP Concert Pass",
  "tokenEditionNumber": "42",
  "tokenVariants": [
    {
      "label": "Tier",
      "value": "Platinum"
    },
    {
      "label": "Seat",
      "value": "A12"
    }
  ],
  "benefits": [
    {
      "name": "General Admission",
      "description": "Access to the main event area with reserved seating.",
      "rules": "Valid for one entry only. No re-entry allowed. Must present QR code at gate.",
      "startDate": "2026-06-15T18:00:00.000Z",
      "endDate": "2026-06-15T23:59:00.000Z",
      "startCheckinTime": "2026-06-15T17:00:00.000Z",
      "endCheckinTime": "2026-06-15T20:00:00.000Z",
      "qrCode": "https://example.com/redeem/benefit-uuid-1234"
    },
    {
      "name": "Backstage Meet & Greet",
      "description": "Exclusive backstage access to meet the performers.",
      "rules": "Limited to 30 minutes. No photography allowed.",
      "startDate": "2026-06-15T22:00:00.000Z",
      "endDate": "2026-06-15T22:30:00.000Z",
      "startCheckinTime": "2026-06-15T21:45:00.000Z",
      "endCheckinTime": "2026-06-15T22:00:00.000Z",
      "qrCode": "https://example.com/redeem/benefit-uuid-5678"
    }
  ],
  "pdfTemplate": "<html><body><h1>{{tokenName}}</h1><p>Edition: {{tokenEditionNumber}}</p>{{#each benefits}}<div><h2>{{this.name}}</h2><p>{{this.description}}</p></div>{{/each}}</body></html>"
}
```

**Response:**

| Status | Content-Type | Description |
|--------|-------------|-------------|
| 200 | `application/pdf` | PDF stream with `Content-Disposition: attachment; filename=pass.pdf` |
| 400 | `application/json` | Invalid request body (missing required fields, malformed DTO) |
| 500 | `application/json` | PDF generation failed |
| 408 | `application/json` | Request exceeded 30-second timeout |

**Example Request:**

```bash
curl -X POST "https://pdf.w3block.io/pass/generateTickets" \
  -H "Content-Type: application/json" \
  -d '{
    "tokenName": "VIP Pass",
    "tokenEditionNumber": "1",
    "tokenVariants": [],
    "benefits": [{
      "name": "Entry",
      "description": "General admission",
      "rules": "Single use",
      "startDate": "2026-06-15T18:00:00.000Z",
      "qrCode": "https://example.com/redeem/abc123"
    }]
  }' \
  -o pass.pdf
```

---

### Health Endpoints

#### GET /health

Full health check returning memory usage metrics.

| Method | Path | Auth |
|--------|------|------|
| GET | `/health` | None |

**Response (200):**

```json
{
  "status": "ok",
  "info": {
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" }
  },
  "error": {},
  "details": {
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" }
  }
}
```

---

#### GET /health/liveness

Liveness probe for container orchestration (Kubernetes).

| Method | Path | Auth |
|--------|------|------|
| GET | `/health/liveness` | None |

**Response:**

| Status | Description |
|--------|-------------|
| 204 | Service is alive |
| 500 | Service is not responding |

---

#### GET /health/readiness

Readiness probe for container orchestration (Kubernetes).

| Method | Path | Auth |
|--------|------|------|
| GET | `/health/readiness` | None |

**Response:**

| Status | Description |
|--------|-------------|
| 204 | Service is ready to accept requests |
| 503 | Service is not ready (dependencies unavailable) |

---

#### GET /

Application info endpoint returning service metadata.

| Method | Path | Auth |
|--------|------|------|
| GET | `/` | None |

**Response (200):**

```json
{
  "name": "w3block-pdf",
  "version": "0.0.1",
  "node_env": "production"
}
```

---

## DTOs

### CertificationDto

Used internally when resolving certificate requests by token ID.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | Yes | Ethereum contract address (hex format, e.g., `0x1234...abcd`) |
| `chainId` | number | Yes | Blockchain chain ID (e.g., `137` for Polygon) |
| `tokenId` | number | Yes | Token ID within the contract |

### CertificationPreviewDto

Request body for the certificate preview endpoint.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pdfTemplate` | string | Yes | Full HTML string containing the certificate template with Handlebars syntax |

### PassDto

Request body for pass/ticket PDF generation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tokenName` | string | Yes | Display name of the token pass |
| `tokenEditionNumber` | string | Yes | Edition number or label (e.g., `"42"`, `"Limited #5"`) |
| `tokenVariants` | TokenVariantsDto[] | Yes | Array of variant label/value pairs. Pass `[]` if no variants |
| `benefits` | BenefitsDto[] | Yes | Array of benefits with QR codes |
| `pdfTemplate` | string | No | Custom HTML template. If omitted, the built-in default template is used |

### TokenVariantsDto

Describes a single variant attribute of a token pass.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | string | Yes | Variant attribute name (e.g., `"Color"`, `"Tier"`, `"Seat"`) |
| `value` | string | Yes | Variant attribute value (e.g., `"Blue"`, `"Platinum"`, `"A12"`) |

### BenefitsDto

Describes a single benefit entry within a pass, including its QR code data.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Benefit display name |
| `description` | string | Yes | Benefit description text |
| `rules` | string | Yes | Usage rules and restrictions |
| `startDate` | string | Yes | Benefit start date/time (ISO 8601) |
| `endDate` | string | No | Benefit end date/time (ISO 8601) |
| `startCheckinTime` | string | No | Check-in window start (ISO 8601) |
| `endCheckinTime` | string | No | Check-in window end (ISO 8601) |
| `qrCode` | string | Yes | Data to encode in the QR code (typically a redemption URL) |

---

## Template Syntax

The PDF service uses [Handlebars](https://handlebarsjs.com/) as its templating engine. Templates are full HTML documents that get rendered to PDF via Puppeteer.

### Basic Handlebars Usage

```html
<!-- Variable output -->
<h1>{{information.title}}</h1>
<p>{{information.description}}</p>
<img src="{{information.mainImage}}" />

<!-- Conditional rendering -->
{{#if information.description}}
  <p>{{information.description}}</p>
{{/if}}

<!-- Iterating over arrays -->
{{#each benefits}}
  <div>
    <h2>{{this.name}}</h2>
    <p>{{this.description}}</p>
  </div>
{{/each}}
```

### Custom Helpers

The PDF service registers two custom Handlebars helpers:

#### dateFormat

Formats a date string using a specified format pattern.

```handlebars
{{dateFormat startDate "DD/MM/YYYY"}}
{{dateFormat startDate "YYYY-MM-DD HH:mm"}}
{{dateFormat createdAt "MMM D, YYYY"}}
```

#### xIf

Extended conditional helper that supports comparison operators.

```handlebars
{{#xIf edition "==" "Limited"}}
  <span class="badge">Limited Edition</span>
{{/xIf}}

{{#xIf tokenId ">" 100}}
  <span>High token ID</span>
{{/xIf}}

{{#xIf status "!=" "expired"}}
  <span>Active</span>
{{/xIf}}

{{#xIf value "typeof" "string"}}
  <span>{{value}}</span>
{{/xIf}}
```

**Supported operators:** `==`, `===`, `!=`, `<`, `>`, `<=`, `>=`, `typeof`

### Certificate Template Variables

When generating certificates, the following variables are available in the Handlebars context:

| Variable | Type | Description |
|----------|------|-------------|
| `information.title` | string | Token metadata title |
| `information.description` | string | Token metadata description |
| `information.mainImage` | string | URL to the token's main image (video URLs auto-converted to PNG) |
| `certificateLink` | string | Full URL to the frontend certificate page |
| `token.address` | string | Contract address |
| `token.chainId` | number | Blockchain chain ID |
| `token.tokenId` | number | Token ID |
| `edition` | object | Edition details from the metadata service |

### Pass Template Variables

When generating pass/ticket PDFs, the following variables are available:

| Variable | Type | Description |
|----------|------|-------------|
| `tokenName` | string | Display name of the token pass |
| `tokenEditionNumber` | string | Edition number |
| `tokenVariants` | array | Array of `{ label, value }` objects |
| `tokenVariants[].label` | string | Variant attribute name |
| `tokenVariants[].value` | string | Variant attribute value |
| `benefits` | array | Array of benefit objects |
| `benefits[].name` | string | Benefit name |
| `benefits[].description` | string | Benefit description |
| `benefits[].rules` | string | Benefit rules |
| `benefits[].startDate` | string | Start date (ISO 8601) |
| `benefits[].endDate` | string | End date (ISO 8601, may be absent) |
| `benefits[].startCheckinTime` | string | Check-in start (ISO 8601, may be absent) |
| `benefits[].endCheckinTime` | string | Check-in end (ISO 8601, may be absent) |
| `benefits[].qrCode` | string | QR code data |

---

## Error Reference

| Status | Exception | Cause | Resolution |
|--------|-----------|-------|------------|
| 400 | `BadRequestException` | Invalid DTO: malformed Ethereum address, invalid chain ID, missing required fields | Verify request body matches the DTO schema. Ethereum addresses must be valid hex strings |
| 404 | `NotFoundException` | No token found matching the provided address/chainId/tokenId or metadata filter | Verify the token exists on-chain. Check address, chainId, and tokenId values. For query-based lookups, verify filter parameters |
| 408 | `RequestTimeoutException` | PDF generation exceeded the 30-second global timeout | Simplify the HTML template. Reduce image count and size. Avoid loading external resources in the template |
| 500 | `InternalServerErrorException` | PDF generation failed due to template rendering error, Puppeteer crash, or KEY service unavailability | Check template syntax for Handlebars errors. Verify the KEY service is accessible. Check service logs for Puppeteer errors |

---

## Video URL Handling

The certificate generation flow includes automatic video-to-thumbnail conversion for `mainImage` URLs. When the metadata service returns a video URL from Cloudinary, the PDF service converts it to a static PNG thumbnail:

| Original URL | Converted URL |
|-------------|---------------|
| `https://res.cloudinary.com/.../video.mp4` | `https://res.cloudinary.com/.../video.png` |
| `https://res.cloudinary.com/.../clip.mov` | `https://res.cloudinary.com/.../clip.png` |
| `https://example.com/image.png` | `https://example.com/image.png` (no change) |

This conversion only applies to Cloudinary-hosted video URLs. Non-Cloudinary URLs and non-video URLs are passed through unchanged.
