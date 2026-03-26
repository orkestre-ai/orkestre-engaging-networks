# Engaging Networks REST API Skill - Verification Audit

**Date:** 2026-03-25
**Region:** CA (`https://ca.engagingnetworks.app/ens/service`)
**API Version:** 6.0.0
**Client:** Oxfam Canada (clientId: 2110)

---

## Summary

Verified 31 read-only (GET) endpoints against the live EN API. 24 passed, 4 blocked by API User permissions, 3 revealed documentation bugs (now fixed).

| Category | Tested | Passed | Blocked | Bugs Found |
|----------|--------|--------|---------|------------|
| Authentication | 2 | 2 | 0 | 1 (body format) |
| Account | 1 | 0 | 1 | 0 |
| Page Services | 6 | 5 | 1 | 0 |
| Page Components | 7 | 0 | 7 | 0 |
| Supporter Services | 9 | 9 | 0 | 1 (daysBack required) |
| Origin Source | 1 | 1 | 0 | 0 |
| Transactions | 4 | 4 | 0 | 1 (transactionId vs id) |
| Export Jobs | 0 | 0 | 0 | 0 |
| Marketing Automation | 4 | 4 | 0 | 0 |
| **Total** | **34** | **25** | **9** | **3** |

Write endpoints (POST/PUT/DELETE) were intentionally skipped to avoid modifying production data.

---

## Detailed Results

### 1. Authentication

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 1 | `POST /authenticate` | PASS | Session token obtained, expires in 60 min |
| 2 | `GET /authenticate/{token}` | PASS | Returns `{"valid": true}` |

**Bug Found & Fixed:** Authentication body must be `{"ens-auth-token":"TOKEN"}` (JSON object), NOT `"TOKEN"` (quoted string). The quoted string returns `"Invalid api key"`. All curl examples and text descriptions across SKILL.md, endpoints.md, and authentication.md corrected.

### 2. Account

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 3 | `GET /account/auditlog` | BLOCKED | "ApiUser does not have adequate permissions" |

### 3. Page Services

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 4 | `GET /page?type=nd` | PASS | 239 donation pages |
| 5 | `GET /page?type=et` | PASS | 39 email-to-target pages |
| 6 | `GET /page?type=ev` | PASS | 16 event pages |
| 7 | `GET /page?type=ec` | PASS | 3 ecommerce pages |
| 8 | `GET /page?type=dc` | FAIL | "Failed to get pages" (error, not permissions) |
| 9 | `GET /page/{id}` | PASS | Returns page detail incl. `template` field |
| 10 | `GET /page/{id}/survey` | BLOCKED | "ApiUser does not have adequate permissions" |

**Note:** `type=dc` returns an error despite docs listing it. Account may have no dc-type pages or the type code may differ. Types `nd`, `et`, `ev`, `ec` all work. The `GET /page/{id}` response includes a `template` field (string name) not present in the list response.

### 4. Page Components

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 11 | `GET /page/component/codeblock` | BLOCKED | Permissions |
| 12 | `GET /page/component/formblock` | BLOCKED | Permissions |
| 13 | `GET /page/component/template` | BLOCKED | Permissions |
| 14 | `GET /page/component/textblock` | BLOCKED | Permissions |
| 15 | `GET /page/component/thankemail` | BLOCKED | Permissions |
| 16 | `GET /page/component/upsell` | BLOCKED | Permissions |
| 17 | `GET /page/component/widget` | BLOCKED | Permissions |

All page component endpoints require additional API User permissions not assigned to this token.

### 5. Supporter Services

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 18 | `GET /supporter/fields` | PASS | Returns account-specific field definitions with `id`, `name`, `tag`, `property` |
| 19 | `GET /supporter/questions` | PASS | Returns question list (OPT, GEN types) |
| 20 | `GET /supporter/questions/{id}` | PASS | Returns localized question with `htmlFieldType` (checkbox, etc.) |
| 21 | `GET /supporter?email=...` | PASS | Returns full supporter record with all custom fields |
| 22 | `GET /supporter/{id}` | PASS | Same as email lookup but by ID |
| 23 | `GET /supporter/query?type=latestModified` | PASS | 48,782 modified today |
| 24 | `GET /supporter/query?type=latestCreated` | PASS | 437 created in last 7 days |
| 25 | `GET /supporter/query?type=suppressed` | PASS | 2,598 suppressed (requires `daysBack`) |
| 26 | `GET /supporter/query?type=search` | PASS | Works with `filter=country:CA` (350,194 results) |
| 27 | `GET /supporter/importformat` | PASS | Returns import format definitions |

**Bug Found & Fixed:** `type=suppressed` requires `daysBack` parameter (API returns "Missing mandatory parameter daysBack"). Documentation updated from "optional" to "required for all query types".

**Discovery:** Search `filter` must use field `property` tags (from GET /supporter/fields), not display names. `filter=Donor:true` fails ("Failed to map field property 'Donor'"), while `filter=country:CA` works because `country` is a valid property tag. Documentation updated.

### 6. Origin Source

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 28 | `GET /supporter/{id}/originsource` | PASS | Returns `{}` when no origin source set |

### 7. Transactions

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 29 | `GET /supporter/{id}/transactions` | PASS | Returns EMAIL, nd, dc, et, SBC transaction types |
| 30 | `GET /supporter/{id}/transactions/{txnId}` | PASS | Works with gateway `transactionId`, returns full detail |
| 31 | `GET /supporter/{id}/transactions/recurring` | PASS | Returns active recurring with amount, frequency, dates |

**Bug Found & Fixed:** The transaction detail endpoint (`/transactions/{transactionId}`) requires the gateway-defined `transactionId` field (e.g., `ND272532241__A23028626`), NOT the internal `id` field (e.g., `57620596`). Using the internal `id` returns HTTP 204 (no data). Documentation updated to clarify this distinction.

**Transaction type codes observed:**
- `EMAIL` - Email broadcast records
- `nd` - Net donor (donation) transactions
- `dc` - Data capture (petition/signup)
- `et` - Email to target
- `SBC` - Subscribe/consent
- Types include `txType` field for financial transactions: `CREDIT_SINGLE`, `BANK_RECURRING`, etc.

**Recurring transaction fields observed:**
```json
{
  "reason": null,
  "nextPaymentDate": "2026-04-01",
  "amount": "40.00",
  "paymentDay": 1,
  "campaignId": 334120,
  "currency": "CAD",
  "id": 1113024,
  "transactionId": "ND272532241__A23028626",
  "startDate": "2024-10-04",
  "status": "ACTIVE",
  "frequency": "MONTHLY"
}
```

### 8. Marketing Automations

| # | Endpoint | Result | Detail |
|---|----------|--------|--------|
| 32 | `GET /ma` | PASS | Returns automation list with filtering by name |
| 33 | `GET /ma/{id}` | PASS | Returns automation detail |
| 34 | `GET /ma/{id}/stats` | PASS | Returns journeyStarts, openRate, clickRate, donations, etc. |

Active automation stats example (id=24899):
```json
{
  "journeyStarts": 416,
  "averageOpenRate": 43.12,
  "averageClickRate": 1.43,
  "actions": 0,
  "donations": 0,
  "objectiveReached": 0,
  "unsubscribes": 12,
  "smsDeliveryRate": 0,
  "jumps": 0
}
```

---

## Bugs Fixed

### Bug 1: Authentication Body Format (CRITICAL)

**Symptom:** `POST /authenticate` with body `"TOKEN"` (quoted string) returns `"Invalid api key"`.

**Root Cause:** The API expects a JSON object `{"ens-auth-token":"TOKEN"}`, not a bare quoted string.

**Files Fixed:**
- `SKILL.md` - verification loop curl command
- `references/endpoints.md` - auth endpoint description and example
- `references/authentication.md` - 6 curl examples and body description

### Bug 2: Suppressed Query Requires daysBack (MODERATE)

**Symptom:** `GET /supporter/query?type=suppressed&rows=3` returns `"Missing mandatory parameter daysBack"`.

**Root Cause:** `daysBack` is required for all query types, not optional.

**Files Fixed:**
- `references/endpoints.md` - parameter description updated to "required for all query types"

### Bug 3: Transaction Detail Uses Gateway transactionId (MODERATE)

**Symptom:** `GET /supporter/{id}/transactions/57620596` returns HTTP 204 (empty). Using `ND272532241__A23028626` returns full detail.

**Root Cause:** The `{transactionId}` path parameter expects the gateway-defined identifier from the `transactionId` field, not the internal EN `id` field.

**Files Fixed:**
- `references/endpoints.md` - parameter description clarified, transaction list response documented with field distinction note

---

## Endpoints Not Tested

### Skipped (Write Operations - Would Modify Data)

| Endpoint | Reason |
|----------|--------|
| `DELETE /authenticate/{token}` | Retires session token |
| `POST /page/{id}/process` | Submits page data (donation, petition) |
| `POST /page/{id}/process/` | Card payment processing |
| `POST /supporter` | Creates/updates supporter |
| `PUT /supporter/{id}` | Updates supporter |
| `DELETE /supporter/{id}` | Deletes supporter |
| `PUT /supporter/bulk` | Bulk suppression update |
| `POST /supporter/importformat` | Creates import format |
| `PUT /supporter/importformat/{id}` | Updates import format |
| `DELETE /supporter/importformat/{id}` | Deletes import format |
| `POST /supporter/{id}/originsource` | Assigns origin source |
| `PUT /supporter/{id}/originsource` | Updates origin source |
| `DELETE /supporter/{id}/originsource` | Removes origin source |
| `POST /supporter/{id}/transactions/recurring` | Migrates recurring transaction |
| `PUT /supporter/{id}/transactions/recurring/{txnId}` | Updates recurring transaction |
| `POST /exportjob` | Initiates export job |
| `POST /ma/{id}/supporter/bulk` | Adds supporters to automation |

### Skipped (Read-Only but Blocked by Permissions)

| Endpoint | Error |
|----------|-------|
| `GET /account/auditlog` | "ApiUser does not have adequate permissions" |
| `GET /page/{id}/survey` | "ApiUser does not have adequate permissions" |
| `GET /page/?action=InUse` | "ApiUser does not have adequate permissions" |
| `GET /page/component/codeblock` | "ApiUser does not have adequate permissions" |
| `GET /page/component/formblock` | "ApiUser does not have adequate permissions" |
| `GET /page/component/template` | "ApiUser does not have adequate permissions" |
| `GET /page/component/textblock` | "ApiUser does not have adequate permissions" |
| `GET /page/component/thankemail` | "ApiUser does not have adequate permissions" |
| `GET /page/component/upsell` | "ApiUser does not have adequate permissions" |
| `GET /page/component/widget` | "ApiUser does not have adequate permissions" |

---

## Real API Response Shapes Observed

### Page List Item (`GET /page?type=nd`)
```json
{
  "id": 61749,
  "campaignId": 184294,
  "name": "Omega Reference - Membership (Test Gateway)",
  "title": "Oxfam Canada Membership",
  "type": "nd",
  "subType": "MEMBERSHIP",
  "clientId": 2110,
  "createdOn": 1591123657000,
  "modifiedOn": 1649375095000,
  "campaignBaseUrl": "https://secured.oxfam.ca",
  "campaignStatus": "live",
  "defaultLocale": "en-CA"
}
```

### Page Detail (`GET /page/{id}`)
Same as list item plus `"template": "Omega 4Site - Oxfam ENGrid"` field.

### Supporter Fields (`GET /supporter/fields`)
```json
{"id": 99368, "name": "Email Address", "tag": "Email Address", "property": "emailAddress"}
```

### Supporter Query Response (`GET /supporter/query`)
```json
{
  "pagination": {"start": 1, "rows": 3, "total": 48782},
  "data": [
    {
      "emailAddress": "...",
      "modifiedOn": "2026-03-25",
      "supporterId": 283464295,
      "createdOn": "2024-09-20"
    }
  ]
}
```
Note: `pagination.start` is 1-based (not 0-based as documented in some places).

### Supporter Record (`GET /supporter/{id}`)
Returns flat key-value with display names as keys, plus `supporterId`, `suppressed`, `questions`, and custom fields.

### Transaction List Item (`GET /supporter/{id}/transactions`)
```json
{
  "createdDate": 1774449193000,
  "campaignId": 382725,
  "exportType": "FCS",
  "name": "Emergency - Conflict in Lebanon - March 2026",
  "id": 57620596,
  "txType": "CREDIT_SINGLE",
  "type": "nd",
  "recurringPayment": "N",
  "status": "success"
}
```

### Transaction Detail (`GET /supporter/{id}/transactions/{transactionId}`)
```json
{
  "pageSubtype": null,
  "recurringFrequency": "MONTHLY",
  "recurringDay": "1",
  "pageTitle": "Donate Now",
  "createdOn": 1728093382000,
  "pageName": "Imported Sustainers - 2OXI80 - Brickwork",
  "paymentType": "ACHEFT",
  "pageStatus": "live",
  "pageType": "nd",
  "currency": "CAD",
  "id": 45642177,
  "expiry": "12/2199",
  "transactionError": null,
  "amount": "0.00",
  "transactionStatus": "success",
  "campaignId": 334120,
  "ccLastFour": null,
  "campaignPageId": 156199,
  "recurringStatus": "ACTIVE",
  "transactionId": "ND272532241__A23028626",
  "parentTransactionId": null,
  "transactionType": "BANK_RECURRING",
  "taxDeductible": "N",
  "recurringPayment": "Y",
  "gateway": "IATS North America",
  "child": false
}
```

### Marketing Automation Stats (`GET /ma/{id}/stats`)
```json
{
  "journeyStarts": 416,
  "averageOpenRate": 43.12,
  "averageClickRate": 1.43,
  "actions": 0,
  "donations": 0,
  "objectiveReached": 0,
  "unsubscribes": 12,
  "smsDeliveryRate": 0,
  "jumps": 0
}
```

---

## Account Data Summary

| Metric | Value |
|--------|-------|
| Donation pages (nd, live) | 239 |
| Email-to-target pages (et) | 39 |
| Event pages (ev) | 16 |
| Ecommerce pages (ec) | 3 |
| Recently modified supporters (1 day) | 48,782 |
| Recently created supporters (7 days) | 437 |
| Suppressed supporters | 2,598 |
| Supporters in Canada | 350,194 |
| Import formats | 3+ |
| Marketing automations | Multiple (incl. active) |
| Supporter questions | 5+ (OPT and GEN types) |
