---
id: FLOW_TOKENIZATION_BULK_IMPORT
title: "Tokenization - Bulk Import/Export (XLSX)"
module: offpix/tokenization
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - xlsx
  - import
  - export
  - bulk
depends_on:
  - TOKENIZATION_API_REFERENCE
  - FLOW_TOKENIZATION_COLLECTION_LIFECYCLE
---

# Token Edition Bulk Import/Export

## Overview

This flow covers bulk management of token editions via Excel (XLSX) spreadsheets. It includes exporting an edition template with the correct column structure, importing edition data from a filled-in spreadsheet, handling import errors, retrying failed operations, and performing async exports of large edition sets. Bulk import is primarily used when `similarTokens` is false and each edition needs individual metadata (name, description, RFID, custom fields).

## Prerequisites

| Requirement | Description | How to obtain |
|-------------|-------------|---------------|
| Bearer token | JWT with Admin role | [Sign-In flow](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | Company UUID | Auth flow / environment config |
| Collection in DRAFT | Token collection with editions synced | [Collection Lifecycle](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Synced editions | `sync-draft-tokens` must have completed | [Collection Lifecycle](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |

---

## Flow: Export Template

Download an Excel template pre-populated with the collection's edition structure and metadata fields.

### Step 1: Export the XLSX Template

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-collections/{id}/export/xlsx` | Bearer (Admin) |

No query parameters required.

**Response:** Binary XLSX file download (Content-Type: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`)

**Template Structure:**

The exported file contains one row per edition with the following columns:

| Column | Type | Description |
|--------|------|-------------|
| `editionNumber` | number | Sequential edition number (read-only, do not modify) |
| `name` | string | Edition display name |
| `description` | string | Edition description |
| `rfid` | string | RFID tag value (must be globally unique) |
| Custom fields... | varies | Additional columns based on the subcategory template |

**Example template content:**

| editionNumber | name | description | rfid | rarity | color |
|---------------|------|-------------|------|--------|-------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| ... | | | | | |
| 100 | | | | | |

**Notes:**
- The `editionNumber` column is pre-filled and must not be changed
- Custom columns (e.g., `rarity`, `color`) are derived from the subcategory template fields
- If editions already have data (from a previous import or manual edit), the existing values are included in the export
- Use this template as the starting point for bulk import to ensure correct column structure

---

## Flow: Import Editions

Upload a filled-in Excel file to update edition data in bulk.

### Step 1: Prepare the Spreadsheet

Fill in the exported template with edition data:

**Example filled content:**

| editionNumber | name | description | rfid | rarity | color |
|---------------|------|-------------|------|--------|-------|
| 1 | Golden Phoenix #1 | A rare golden phoenix | RFID-GP-001 | legendary | gold |
| 2 | Silver Dragon #2 | A majestic silver dragon | RFID-SD-002 | epic | silver |
| 3 | Bronze Griffin #3 | A sturdy bronze griffin | RFID-BG-003 | rare | bronze |

**Preparation checklist:**
- Do not modify the `editionNumber` column
- Ensure all RFIDs are unique within the file and across the platform
- Fill in required fields defined by the subcategory template
- Save the file in `.xlsx` format (not `.xls` or `.csv`)
- Keep file size reasonable (large files with thousands of rows may timeout)

### Step 2: Upload the File

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| POST | `/{companyId}/token-collections/{id}/import/xlsx` | Bearer (Admin) |

**Request:** `multipart/form-data`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | file | Yes | The XLSX file to import |

**cURL Example:**

```bash
curl -X POST \
  "https://api.w3block.io/{companyId}/token-collections/{collectionId}/import/xlsx" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@editions.xlsx"
```

**Response (200):**

```json
{
  "jobId": "job-uuid"
}
```

### Step 3: Poll for Job Completion

The import is processed asynchronously. Poll the job status endpoint for completion.

**Poll Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/jobs/{jobId}` | Bearer (Admin) |

**Response (in progress):**

```json
{
  "id": "job-uuid",
  "status": "processing",
  "progress": 35,
  "createdAt": "2026-04-02T12:00:00.000Z"
}
```

**Response (completed successfully):**

```json
{
  "id": "job-uuid",
  "status": "completed",
  "progress": 100,
  "result": {
    "totalRows": 100,
    "updated": 98,
    "errors": 2
  },
  "createdAt": "2026-04-02T12:00:00.000Z",
  "completedAt": "2026-04-02T12:02:00.000Z"
}
```

**Response (failed):**

```json
{
  "id": "job-uuid",
  "status": "failed",
  "progress": 0,
  "error": "Invalid file format",
  "createdAt": "2026-04-02T12:00:00.000Z",
  "completedAt": "2026-04-02T12:00:05.000Z"
}
```

**Notes:**
- Poll at reasonable intervals (every 2-5 seconds for small files, every 10-15 seconds for large files)
- The `result.errors` count indicates how many rows had validation issues
- Even if some rows fail, the valid rows are still processed
- The collection must be in DRAFT status for import to work

---

## Flow: Handle Import Errors

When some editions fail validation during import, they are marked with error statuses. This flow covers identifying and fixing those errors.

### Step 1: List Editions with Errors

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions` | Bearer (Admin) |

**Query to find error editions:**

```
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=importError&page=1&limit=50
```

Also check for draft errors:

```
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=draftError&page=1&limit=50
```

**Response (200):**

```json
{
  "items": [
    {
      "id": "edition-uuid-1",
      "editionNumber": 45,
      "tokenCollectionId": "collection-uuid",
      "status": "importError",
      "name": "Asset #45",
      "rfid": "RFID-DUPLICATE",
      "errorFields": {
        "rfid": "RFID_HAS_ALREADY_BEEN_USED"
      },
      "createdAt": "2026-04-02T12:00:00.000Z",
      "updatedAt": "2026-04-02T12:02:00.000Z"
    },
    {
      "id": "edition-uuid-2",
      "editionNumber": 78,
      "tokenCollectionId": "collection-uuid",
      "status": "importError",
      "name": "",
      "errorFields": {
        "name": "IS_REQUIRED",
        "image": "INVALID_URL"
      },
      "createdAt": "2026-04-02T12:00:00.000Z",
      "updatedAt": "2026-04-02T12:02:00.000Z"
    }
  ],
  "meta": {
    "totalItems": 2,
    "itemCount": 2,
    "itemsPerPage": 50,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

### Step 2: Fix Errors

There are two approaches to fix import errors:

**Option A: Fix individually via API**

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/{id}` | Bearer (Admin) |

**Request Body:**

```json
{
  "name": "Fixed Asset #45",
  "rfid": "RFID-UNIQUE-045",
  "tokenData": {
    "image": "https://example.com/valid-image.png"
  }
}
```

**Response (200):** Returns the updated TokenEditionEntity. The status may change from `importError` to `draft` once all error fields are resolved.

**Option B: Fix in spreadsheet and re-import**

1. Export a new template (includes current data and error states)
2. Fix the problematic rows in the spreadsheet
3. Re-import the corrected file
4. The re-import overwrites existing edition data for matching edition numbers

### Step 3: Verify All Errors Resolved

Before publishing, confirm no editions remain in error states:

```
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=importError&limit=1
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=draftError&limit=1
```

Both queries should return `totalItems: 0`.

**Common error field values:**

| Error Code | Field | Meaning |
|------------|-------|---------|
| `IS_REQUIRED` | Any required field | The field is required but was empty or missing |
| `INVALID_URL` | image, mainImage | The URL format is invalid |
| `INVALID_STRING` | name, description | The value is not a valid string |
| `INVALID_NUMBER` | numeric fields | The value is not a valid number |
| `INVALID_DATE` | date fields | The date format is invalid |
| `INVALID_YEAR` | year fields | The year is outside acceptable range |
| `RFID_HAS_ALREADY_BEEN_USED` | rfid | The RFID is already assigned to another active edition |
| `RFID_HAS_ITEMS_DUPLICATED_ON_LIST` | rfid | Duplicate RFID values within the same import file |
| `INVALID_2D_DIMENSIONS` | dimension fields | 2D dimension format is incorrect |
| `INVALID_3D_DIMENSIONS` | dimension fields | 3D dimension format is incorrect |
| `INVALID_BOOLEAN` | boolean fields | Value is not a valid boolean |
| `INVALID_ARRAY` | array fields | Value is not a valid array |
| `INVALID_TYPE` | any typed field | Value type does not match expected type |

---

## Flow: Retry Failed Operations

After fixing errors, retry all failed editions in a collection at once.

### Retry Bulk

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| PATCH | `/{companyId}/token-editions/retry-bulk-by-collection` | Bearer (Admin) |

**Request Body:**

```json
{
  "tokenCollectionId": "collection-uuid"
}
```

**Response (200):**

```json
{
  "retried": 5
}
```

**Notes:**
- Retries editions with error statuses: `importError`, `draftError`, `burnFailure`, `transferFailure`
- Resets editions to the appropriate previous state and re-processes them
- For import/draft errors: resets to `draft` and re-validates
- For burn/transfer failures: resets to `minted` and re-submits the blockchain transaction
- The `retried` count indicates how many editions were affected

---

## Flow: Async Export

For large collections, use the async export endpoint to generate an Excel file without timeout risks.

### Request Async Export

**Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/{companyId}/token-editions/xls` | Bearer (Admin) |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenCollectionId` | uuid | Yes | Collection to export |

**Example:**

```
GET /{companyId}/token-editions/xls?tokenCollectionId={collectionId}
```

**Response (200):**

```json
{
  "jobId": "job-uuid"
}
```

### Poll for Export Completion

**Poll Endpoint:**

| Method | Path | Auth |
|--------|------|------|
| GET | `/jobs/{jobId}` | Bearer (Admin) |

**Response (completed):**

```json
{
  "id": "job-uuid",
  "status": "completed",
  "progress": 100,
  "result": {
    "downloadUrl": "https://storage.w3block.io/exports/editions-export-uuid.xlsx",
    "expiresAt": "2026-04-03T12:00:00.000Z"
  },
  "createdAt": "2026-04-02T12:00:00.000Z",
  "completedAt": "2026-04-02T12:01:00.000Z"
}
```

**Notes:**
- The download URL is temporary and expires (typically within 24 hours)
- Use this endpoint for collections with hundreds or thousands of editions where the synchronous export might timeout
- The exported file follows the same column structure as the template export

---

## Import/Export Workflow Summary

```
1. Create collection draft
   POST /{companyId}/token-collections

2. Set quantity and sync editions
   PUT /{companyId}/token-collections/{id}  (set quantity)
   PATCH /{companyId}/token-collections/{id}/sync-draft-tokens  (generate editions)
   GET /jobs/{jobId}  (poll until complete)

3. Export template
   GET /{companyId}/token-collections/{id}/export/xlsx

4. Fill in spreadsheet with edition data
   (offline step)

5. Import filled spreadsheet
   POST /{companyId}/token-collections/{id}/import/xlsx
   GET /jobs/{jobId}  (poll until complete)

6. Check for and fix errors
   GET /{companyId}/token-editions?tokenCollectionId={id}&status=importError
   PATCH /{companyId}/token-editions/{editionId}  (fix each)
   -- or --
   Re-export, fix in spreadsheet, re-import

7. Verify no errors remain
   GET /{companyId}/token-editions?tokenCollectionId={id}&status=importError&limit=1
   GET /{companyId}/token-editions?tokenCollectionId={id}&status=draftError&limit=1

8. Publish collection
   PATCH /{companyId}/token-collections/publish/{id}
```

---

## Error Handling

| Status | Error | Cause | Resolution |
|--------|-------|-------|------------|
| 400 | Invalid file format | File is not a valid XLSX | Save the file as `.xlsx` format (not `.xls` or `.csv`) |
| 400 | TokenCollectionNotDraftException | Collection is already published | Import is only allowed on DRAFT collections |
| 400 | Edition number mismatch | Spreadsheet contains edition numbers outside the collection range | Ensure edition numbers match the synced editions (1 to quantity) |
| 400 | JobRunningException | Another import or sync job is still running | Wait for the current job to complete |
| 400 | File too large | The uploaded file exceeds size limits | Split into multiple smaller files or reduce data volume |
| 408 | Request timeout | Synchronous export timed out for large collection | Use the async export endpoint (`/token-editions/xls`) instead |

## Common Pitfalls

| # | Problem | Solution |
|---|---------|----------|
| 1 | Large file import times out | The import itself is async (returns jobId), but the upload might timeout for very large files. Keep file sizes reasonable (under 10MB) |
| 2 | RFID conflicts in import | Ensure all RFIDs in the spreadsheet are unique both within the file and across the platform. Check with `check-rfid` for critical values |
| 3 | Character encoding issues | Save the Excel file with UTF-8 encoding. Special characters may be corrupted with other encodings |
| 4 | Missing editionNumber column | The `editionNumber` column must be present and match the synced editions. Do not delete or reorder this column |
| 5 | Modifying editionNumber values | Edition numbers are assigned during sync and must not be changed. The import matches rows by editionNumber |
| 6 | Import on published collection | Import is only allowed when the collection is in DRAFT status. Use the metadata update endpoint for published collections |
| 7 | Re-import does not clear previous data | Re-importing overwrites only the fields present in the spreadsheet. Empty cells do not clear existing values -- explicitly set them to empty strings if needed |
| 8 | Template columns change after subcategory update | If the subcategory template is changed, re-export the template to get the updated column structure |

## Related Flows

| Flow | Relationship | Document |
|------|-------------|----------|
| Collection Lifecycle | Required: collection must be created and editions synced first | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Edition Operations | After import: manage individual editions, mint, transfer | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| API Reference | Full endpoint and DTO details | [TOKENIZATION_API_REFERENCE](./TOKENIZATION_API_REFERENCE.md) |
| Auth Sign-In | Required for Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
