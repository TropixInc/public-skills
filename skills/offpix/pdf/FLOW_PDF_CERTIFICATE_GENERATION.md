---
id: FLOW_PDF_CERTIFICATE_GENERATION
title: "PDF - Certificate Generation (By Token ID, By Query, Preview)"
module: offpix/pdf
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - pdf
  - certificate
  - metadata
  - blockchain
  - template
depends_on:
  - PDF_API_REFERENCE
---

# PDF Certificate Generation

## Overview

This flow covers generating PDF certificates for blockchain tokens. Certificates are created by fetching token metadata from the KEY service and rendering it into an HTML template via Handlebars and Puppeteer. The PDF service supports three certificate operations: generating by exact token ID, generating by metadata query (finding the first matching token), and previewing a custom template layout.

> **Internal service:** The PDF service does not require authentication. It is an internal service meant to be called from backend services or developer tooling, not directly from end-user frontends. Secure access at the network level.

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Contract address | Ethereum contract address where the token lives | From token creation / Contracts module |
| Chain ID | Blockchain network identifier (e.g., `137` for Polygon) | From token creation / Contracts module |
| Token ID | Specific token ID within the contract (for direct lookup) | From minting / tokenization flow |
| KEY service | The KEY metadata service must be accessible at the configured `KEY_URL` | Infrastructure / environment config |

---

## Flow: Generate Certificate by Token ID

The most common flow. Given a specific contract address, chain ID, and token ID, generate a PDF certificate.

### Step 1: Request the Certificate

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/certification/{address}/{chainId}/{tokenId}` | None |

**Example Request (download):**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/1"
```

**Example Request (inline preview):**

```bash
curl "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/1?preview=true"
```

### Step 2: Internal Processing (automatic)

The service performs the following steps automatically:

1. **Fetch metadata** from the KEY service:
   ```
   GET {KEY_URL}/metadata/address/0xAbC123.../137/1
   ```

2. **Extract template variables** from the metadata response:
   - `information.title` -- the token's display title
   - `information.description` -- the token's description text
   - `information.mainImage` -- URL to the token's primary image

3. **Handle video URLs**: If `mainImage` is a Cloudinary video URL (e.g., `.mp4`, `.mov`), the service automatically replaces the extension with `.png` to use the video thumbnail instead. This prevents broken images in the PDF.

4. **Build the template context**:
   ```json
   {
     "information": {
       "title": "My NFT Certificate",
       "description": "A unique digital collectible.",
       "mainImage": "https://res.cloudinary.com/.../image.png"
     },
     "certificateLink": "https://app.w3block.io/dash/certificate/0xAbC123.../137/1",
     "token": {
       "address": "0xAbC123def456789012345678901234567890ABCD",
       "chainId": 137,
       "tokenId": 1
     },
     "edition": {
       "number": 1,
       "total": 100
     }
   }
   ```

5. **Render HTML** using Handlebars with the certificate template and context data.

6. **Generate PDF** using Puppeteer (headless Chromium) and return the PDF stream.

### Step 3: Handle the Response

**Success (200):**

The response body is a raw PDF binary stream with `Content-Type: application/pdf`.

- **Without `preview` query param:** The response includes `Content-Disposition: attachment` header, prompting a file download in browsers.
- **With `preview` query param:** The response is inline, allowing browsers to display the PDF directly (e.g., in an iframe).

**Error Responses:**

| Status | Cause | Action |
|--------|-------|--------|
| 404 | Token not found at the given address/chainId/tokenId | Verify the token exists on-chain. Check that the contract address, chain ID, and token ID are correct |
| 500 | PDF generation failed (template error, Puppeteer issue, KEY service down) | Check service logs. Verify KEY service is accessible. Validate template syntax |
| 408 | Generation exceeded 30-second timeout | Simplify the template. Reduce image count and resolution |

---

## Flow: Generate Certificate by Metadata Query

Find a token by metadata attributes and generate a certificate for the first match. Useful when the exact token ID is not known but metadata properties are.

### Step 1: Request with Metadata Filters

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/certification/{address}/{chainId}/q` | None |

**Example Request:**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/q?rarity=legendary&color=gold"
```

All query parameters except `preview` are treated as metadata filter key-value pairs. The service passes them to the KEY metadata endpoint to find matching tokens.

### Step 2: Internal Processing (automatic)

1. **Query metadata** from the KEY service with the filter parameters
2. **Select the first matching token** from the results
3. If **no match is found**, the service returns a `404 NotFoundException`
4. For the matched token, the service follows the same rendering pipeline as the token ID flow (Steps 2-3 above)

### Step 3: Handle the Response

Same response format as the token ID flow. The PDF is returned as a binary stream.

**Example with inline preview:**

```bash
curl "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/q?rarity=legendary&preview=true"
```

**Error Responses:**

| Status | Cause | Action |
|--------|-------|--------|
| 404 | No token matches the provided metadata filters | Verify filter parameters match existing token metadata. Check available metadata keys and values |
| 500 | PDF generation failed | Check service logs and template syntax |
| 408 | Generation exceeded 30-second timeout | Simplify the template |

---

## Flow: Preview a Custom Certificate Template

Test a custom HTML template without needing a real token. The service escapes all Handlebars delimiters so variable names appear as literal text in the preview PDF.

### Step 1: Submit the Template

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| POST | `/certification/preview` | None |

**Minimal Request:**

```json
{
  "pdfTemplate": "<html><head><style>body { font-family: Arial; padding: 40px; } h1 { color: #333; }</style></head><body><h1>{{information.title}}</h1><p>{{information.description}}</p></body></html>"
}
```

**Complete Request (with full certificate layout):**

```json
{
  "pdfTemplate": "<html><head><style>body { font-family: 'Helvetica Neue', sans-serif; margin: 0; padding: 40px; } .certificate { border: 2px solid #c9a54e; padding: 60px; text-align: center; } .title { font-size: 28px; color: #1a1a1a; margin-bottom: 20px; } .image { max-width: 400px; border-radius: 8px; } .details { margin-top: 30px; font-size: 14px; color: #666; } .edition { font-size: 16px; color: #c9a54e; margin-top: 10px; }</style></head><body><div class='certificate'><h1 class='title'>{{information.title}}</h1><img class='image' src='{{information.mainImage}}' /><p>{{information.description}}</p><div class='details'><p>Contract: {{token.address}}</p><p>Chain: {{token.chainId}} | Token: {{token.tokenId}}</p></div><p class='edition'>Edition {{edition.number}} of {{edition.total}}</p><a href='{{certificateLink}}'>View Certificate</a></div></body></html>"
}
```

### Step 2: Internal Processing (automatic)

1. **Escape Handlebars delimiters**: All `{{` are replaced with `\{{` so that variable names render as literal text in the preview
2. **Render HTML** with the escaped template (no data context is injected)
3. **Generate PDF** using Puppeteer
4. Return the PDF stream

### Step 3: Review the Preview

The returned PDF shows the template layout with Handlebars variable names displayed as-is (e.g., `{{information.title}}` appears literally in the PDF). This lets template designers verify:

- Page layout and spacing
- CSS styling and fonts
- Element positioning
- Where dynamic data will appear

**Notes:**
- Images referenced by Handlebars variables (e.g., `{{information.mainImage}}`) will show as broken images in the preview since no data is injected
- To test with actual images, hardcode image URLs in the template during preview, then replace with Handlebars variables for production

---

## Template Variables Reference

Full list of variables available in the Handlebars context during certificate generation:

| Variable Path | Type | Description | Example Value |
|---------------|------|-------------|---------------|
| `information.title` | string | Token metadata title | `"Genesis Collection #42"` |
| `information.description` | string | Token metadata description | `"A unique digital artwork..."` |
| `information.mainImage` | string | Token primary image URL (video URLs auto-converted to PNG) | `"https://res.cloudinary.com/.../image.png"` |
| `certificateLink` | string | URL to the frontend certificate page | `"https://app.w3block.io/dash/certificate/0xAbC.../137/1"` |
| `token.address` | string | Contract address | `"0xAbC123...ABCD"` |
| `token.chainId` | number | Blockchain chain ID | `137` |
| `token.tokenId` | number | Token ID | `1` |
| `edition` | object | Edition metadata from the KEY service | `{ "number": 1, "total": 100 }` |

---

## Integration with Frontend

### Certificate Page URL Pattern

The Offpix frontend displays certificates at:

```
/dash/certificate/{contractAddress}/{chainId}/{tokenId}
```

### Download Pattern (JavaScript/TypeScript)

```typescript
import axios from 'axios';

async function downloadCertificate(
  address: string,
  chainId: number,
  tokenId: number
): Promise<void> {
  const response = await axios.get(
    `https://pdf.w3block.io/certification/${address}/${chainId}/${tokenId}`,
    { responseType: 'arraybuffer' }
  );

  const blob = new Blob([response.data], { type: 'application/pdf' });
  const url = window.URL.createObjectURL(blob);

  const link = document.createElement('a');
  link.href = url;
  link.download = `certificate-${tokenId}.pdf`;
  link.click();

  window.URL.revokeObjectURL(url);
}
```

### Preview Pattern (iframe)

```html
<iframe
  src="https://pdf.w3block.io/certification/0xAbC123.../137/1?preview=true"
  width="100%"
  height="800px"
  style="border: none;"
></iframe>
```

> **Security note:** Since the PDF service is internal, frontend applications should proxy certificate requests through their own backend rather than calling the PDF service directly from the browser.

---

## Error Handling

### Common Error Scenarios

| Scenario | Error | Resolution |
|----------|-------|------------|
| Token does not exist on-chain | 404 NotFoundException | Verify the token was minted. Check contract address, chain ID, and token ID |
| KEY metadata service is unreachable | 500 InternalServerErrorException | Check KEY service health. Verify `KEY_URL` configuration |
| Template contains invalid Handlebars syntax | 500 InternalServerErrorException | Validate template syntax. Check for unclosed `{{#if}}` or `{{#each}}` blocks |
| PDF generation exceeds 30 seconds | 408 RequestTimeoutException | Reduce template complexity. Minimize external resource loading. Compress images |
| Metadata has a video URL for mainImage | No error (auto-handled) | The service auto-converts Cloudinary video URLs to PNG thumbnails. Non-Cloudinary video URLs may render as broken images |

### Retry Strategy

The PDF service does not implement internal retries. Consuming services should implement their own retry logic with exponential backoff for transient failures (500, 408).

```typescript
async function getCertificateWithRetry(
  url: string,
  maxRetries: number = 3
): Promise<ArrayBuffer> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await axios.get(url, { responseType: 'arraybuffer' });
      return response.data;
    } catch (error) {
      if (attempt === maxRetries) throw error;
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

## Common Pitfalls

| # | Pitfall | Detail |
|---|---------|--------|
| 1 | Video URL in mainImage produces broken image | Only Cloudinary video URLs are auto-converted. If your video is hosted elsewhere, the PDF will contain a broken image. Host on Cloudinary or provide a direct image URL in the metadata |
| 2 | Template syntax errors fail silently | Handlebars may render empty strings for undefined variables rather than throwing errors. Use `{{#if variable}}` guards to handle missing data |
| 3 | Preview escapes all Handlebars | The preview endpoint is not a "render with sample data" feature. It escapes all `{{` delimiters. To test with real data, use the token ID endpoint with a test token |
| 4 | Large images cause timeout | Images embedded in the template are loaded by Puppeteer during rendering. Large images (especially high-DPI) can push rendering past the 30-second timeout |
| 5 | External fonts may not load | Puppeteer runs in a sandboxed environment. External font URLs (Google Fonts, etc.) may fail to load. Inline fonts as base64 or use system fonts |

---

## Related Flows

| Flow | Module | Relationship |
|------|--------|-------------|
| Token Creation / Minting | [Contracts](../contracts/CONTRACTS_SKILL_INDEX.md) | Tokens must exist on-chain before certificates can be generated |
| Token Metadata | [Tokenization](../tokenization/TOKENIZATION_SKILL_INDEX.md) | Certificate data (title, description, image) comes from token metadata |
| Pass Generation | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) | Alternative PDF type for pass/ticket use cases |
