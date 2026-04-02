---
id: PDF_SKILL_INDEX
title: "PDF Skill Index"
module: offpix/pdf
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# PDF Skill Index

Documentation for the W3Block PDF module (w3block-pdf). This is an **internal service** that generates PDF documents from HTML templates using NestJS, Puppeteer, and Handlebars. It produces PDF certificates for blockchain tokens (using on-chain metadata) and pass/ticket PDFs with QR codes for benefit redemption. While the service is not directly exposed to end users, its API is useful for developers building custom applications on top of the W3Block platform.

**Service:** PDF (w3block-pdf) | **Swagger:** https://pdf.w3block.io/docs

> **Internal service:** The PDF service runs without authentication on its endpoints. Access is controlled at the network/firewall level. Do not expose this service directly to the public internet.

---

## Documents

| # | Document | Version | Description | Status | When to use |
|---|----------|---------|-------------|--------|-------------|
| 1 | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) | 1.0.0 | All endpoints, DTOs, template syntax, configuration | Implemented | API reference at any time |
| 2 | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) | 1.0.0 | Generate certificates from token metadata, preview, query-based lookup | Implemented | Build certificate download/preview features |
| 3 | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) | 1.0.0 | Generate pass/ticket PDFs with benefits and QR codes | Implemented | Build pass/ticket PDF generation features |

---

## Quick Guide

### For certificate generation:

```
1. Read: FLOW_PDF_CERTIFICATE_GENERATION.md   -> Generate certificate by token ID or metadata query
2. Consult: PDF_API_REFERENCE.md              -> For DTOs, template variables, error codes
```

### For pass/ticket generation:

```
1. Read: FLOW_PDF_PASS_GENERATION.md          -> Generate pass PDF with benefits and QR codes
2. Consult: PDF_API_REFERENCE.md              -> For PassDto schema, template syntax, helpers
```

### Minimal API-first implementation:

```
1. Generate a certificate   -> GET /certification/{address}/{chainId}/{tokenId}
2. Generate a pass/ticket   -> POST /pass/generateTickets
3. Consult API details      -> PDF_API_REFERENCE.md
```

---

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | PDF generation times out after 30 seconds | Simplify the HTML template, reduce image sizes, avoid external resource loading in templates. The service enforces a global 30-second timeout |
| 2 | Video URLs in `mainImage` produce broken images | The service automatically converts Cloudinary video URLs to PNG thumbnails (replaces extension with `.png`). Ensure video assets are hosted on Cloudinary for this to work |
| 3 | Template preview renders Handlebars variables literally | The preview endpoint escapes `{{var}}` to `\{{var}}` to prevent rendering with empty data. This is expected behavior |
| 4 | Endpoints accessible without authentication | This is by design. The PDF service is an internal service with no auth layer. Secure it via network policies and firewall rules; never expose it directly to the public internet |

---

## Decision Table

| I want to... | Read this |
|--------------|-----------|
| Generate a PDF certificate for a specific token | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) |
| Find a token by metadata and generate its certificate | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) |
| Preview a custom certificate template | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) |
| Generate a pass/ticket PDF with QR codes | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) |
| Use a custom HTML template for passes | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) |
| Understand DTOs, Handlebars helpers, and template variables | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) |
| Check service health and configuration | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) |
| Understand error codes and timeouts | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) |

---

## Matrix: Endpoints x Documents

| Endpoint | API Ref | Certificate Generation | Pass Generation |
|----------|:-------:|:----------------------:|:---------------:|
| GET /certification/{address}/{chainId}/{tokenId} | X | X | |
| GET /certification/{address}/{chainId}/q? | X | X | |
| POST /certification/preview | X | X | |
| POST /pass/generateTickets | X | | X |
| GET /health | X | | |
| GET /health/liveness | X | | |
| GET /health/readiness | X | | |
| GET / | X | | |
