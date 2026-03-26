<overview>
Complete reference for Engaging Networks REST API v6.0.0 endpoints. Base URL structure: `https://{region}.engagingnetworks.app/ens/service`

**8 Service Groups, 44 Paths:**
1. Authentication (3 endpoints)
2. Account (1 endpoint)
3. Page Services (5 endpoints)
4. Page Components (17 endpoints)
5. Supporter Services (17 endpoints)
6. Origin Source (4 endpoints)
7. Export Job Services (3 endpoints)
8. Marketing Automation (4 endpoints)
</overview>

<risk_classification>
## Endpoint Risk Classification

Endpoints that modify data are tagged with a risk level:

- **READ** (default, no tag): Safe to execute freely. All GET requests plus POST /authenticate and POST /exportjob.
- **WRITE**: Creates or modifies data. Show the full request payload to the user and get explicit confirmation before executing.
- **DESTRUCTIVE**: Deletes data irreversibly. Show payload + warn about irreversibility + get explicit confirmation.
</risk_classification>

<regional_instances>
## Base URLs by Region

| Region | Base URL |
|--------|----------|
| Canada / Europe | `https://ca.engagingnetworks.app/ens/service` |
| United States | `https://us.engagingnetworks.app/ens/service` |
| United States 2 | `https://us2.engagingnetworks.app/ens/service` |

Check your EN admin panel URL to determine your region.

**Card payment processing** uses a `-vault` variant of the host (e.g., `https://ca-vault.engagingnetworks.app/ens/service`) for tokenized credit card submissions.
</regional_instances>

---

<authentication_endpoints>
## 1. Authentication

### POST /authenticate

Authenticates an API User and returns a session token required for all other API calls.

**Request Body:** `application/json` - a JSON object with `ens-auth-token` key

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"ens-auth-token":"YOUR_API_USER_TOKEN"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate
```

**Response 200:**
```json
{
  "expires": "2024-10-17T15:30:00Z",
  "ens-auth-token": "session-token-uuid-here"
}
```

**Response 401:** `{"message": "...", "messageId": "..."}`

---

### GET /authenticate/{ens-auth-token}

Check whether a session token is still valid.

**Path Parameters:**
- `ens-auth-token` (string, required): The session token to validate

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/authenticate/SESSION_TOKEN"
```

**Response 200:**
```json
{
  "valid": true,
  "ens-auth-token": "session-token-uuid-here"
}
```

---

### DELETE /authenticate/{ens-auth-token}

Retire a session token before it expires.

**Path Parameters:**
- `ens-auth-token` (string, required): The session token to retire

```bash
curl -X DELETE \
  "https://ca.engagingnetworks.app/ens/service/authenticate/SESSION_TOKEN"
```

**Response 200:** Token retired successfully.
</authentication_endpoints>

---

<account_endpoints>
## 2. Account

### GET /account/auditlog

Retrieve a filtered list of audit log entries for the account.

**Query Parameters:**
- `type` (string, optional): Comma-separated list of entry types
  - Values: `LOGIN`, `LOGOUT`, `ADD`, `UPDATE`, `DELETE`, `AUDIT`, `PAGE_AUDIT`, `DUP_RESET`, `VALIDATE`
- `entityType` (string, optional): Comma-separated list of entity types
  - Values: `CLIENT`, `OPT_IN`, `SANDBOX`, `USER`, `CAMPAIGNER`, `CMP_PAGE`, `CMP_DESIGN`, `CMP_DEF`, `GATEWAY`, `ACCT_EMAIL`, and more
- `entityId` (string, optional): Comma-separated list of entity IDs
- `startDate` (string, optional): ISO 8601 date (YYYY-MM-DD)
- `endDate` (string, optional): ISO 8601 date (YYYY-MM-DD)
- `sortField` (string, optional): Field to sort by (currently: `createdOn`)
- `sortDirection` (string, optional): `asc` or `desc`
- `userId` (string, optional): Comma-separated list of user IDs

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/account/auditlog?type=LOGIN,LOGOUT&startDate=2024-01-01&endDate=2024-12-31"
```

**Response 200:** CSV format list of audit log entries.
**Response 401:** `{"message": "..."}`
</account_endpoints>

---

<page_service_endpoints>
## 3. Page Services

### GET /page

List pages filtered by type.

**Query Parameters:**
- `type` (string, **required**): The page type to filter by
- `status` (string, optional): Filter by status
  - Values: `new`, `live`, `close`, `tested`, `block`, `delete`

**Response 200:** Array of page objects
```typescript
interface ENPage {
  id: number;
  campaignId: number;
  name: string;
  title: string;
  type: string;
  subType: string;
  clientId: number;
  createdOn: number;       // epoch timestamp
  modifiedOn: number;      // epoch timestamp
  campaignBaseUrl: string;
  campaignStatus: string;
  defaultLocale: string;
  trackingParameters?: string;
}
```

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page?type=dc&status=live"
```

**Page types (common):** `dc` (data capture/petition), `nd` (net donor/donation), `et` (email to target), `ec` (ecommerce), `ev` (event), `p2p` (peer to peer), `pitm` (premium in the mail), `sv` (survey)

---

### GET /page/{id}

Retrieve details about a specific page. Returns the same fields as the list response plus an additional `template` field with the page template name.

**Path Parameters:**
- `id` (integer, required): Page ID

**Response 200:**
```json
{
  "id": 12345,
  "campaignId": 67890,
  "name": "Emergency Relief Fund",
  "type": "nd",
  "subType": "standard",
  "clientId": 100,
  "createdOn": 1705312200000,
  "modifiedOn": 1728569520000,
  "campaignBaseUrl": "https://example.org/page/12345/donate/1",
  "campaignStatus": "live",
  "defaultLocale": "en-US",
  "trackingParameters": "",
  "template": "My Page Template"
}
```

**Note:** The `template` field (string, template name) is only present in the detail response, not in list results.

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/12345"
```

---

### POST /page/{id}/process -- WRITE

> **Requires human approval.** This submits real data to an EN page and may trigger financial transactions, auto-responder emails, and CRM updates.

Process a page submission (donation, petition signature, event registration, etc.).

Use this endpoint for submissions that do NOT include raw credit card numbers. For credit card payments, use the `-vault` endpoint variant instead.

**Path Parameters:**
- `id` (integer, required): Page ID

**Request Body:** `application/json`

```typescript
interface PageProcessRequest {
  demo?: boolean;                    // true = test mode, no real charges
  suppressAutoResponder?: boolean;   // true = skip thank-you email
  standardFieldNames?: boolean;      // true = use field codes, false = display names
  trackingId?: string;               // External tracking reference
  appealCode?: string;               // Campaign appeal code
  locale?: string;                   // e.g., "en-US", "fr-CA", "de-DE"
  txn1?: string; txn2?: string;      // Custom transaction fields (txn1-txn10)
  // ... txn3 through txn10
  supporter: {
    emailAddress: string;            // Required (or "Email Address" if standardFieldNames=false)
    title?: string;
    firstName?: string;
    lastName?: string;
    address1?: string;
    city?: string;
    region?: string;
    country?: string;
    postcode?: string;
    phoneNumber?: string;
    questions?: Record<string, string>;  // e.g., {"Opt-in": "Y"}
  };
  transaction?: {
    donationAmt: number;
    paymenttype: string;         // "visa", "mc", "amex", "paypal", etc.
    paycurrency: string;         // "USD", "CAD", "GBP", "EUR"
    recurrpay?: "Y" | "N";
    recurrfreq?: "MONTHLY" | "QUARTERLY" | "SEMI_ANNUAL" | "ANNUAL";
    recurrday?: number;          // Day of month for recurring
    recurrstart?: string;        // ISO date
    recurrend?: string;          // ISO date
    // ACH/bank fields
    bankAccountNumber?: string;
    bankRoutingNumber?: string;
    bankAccountType?: string;
  };
  product?: {
    productVariantId: number;
    productSku: string;
  };
  event?: {
    additionalAmount?: number;
    discount?: string;
    tickets: Array<{
      // Attendee ticket details
    }>;
  };
  membership?: {
    promoCode?: string;
    membershipVariantId: number;
    members: Array<{}>;
  };
  contactMessage?: {              // For email-to-target pages
    subject: string;
    message: string;
    messageFormat: string;        // "plain" or "html"
  };
}
```

**Response 200:**
```json
{
  "status": "success",
  "id": 98765,
  "supporterId": 212200,
  "supporterEmailAddress": "otto.nv@example.com",
  "type": "nd",
  "transactionId": "txn_abc123",
  "error": "",
  "amount": 25.00,
  "currency": "CAD",
  "recurringPayment": false,
  "paymentType": "visa",
  "recurringFrequency": "",
  "recurringDay": 0,
  "createdOn": 1728569520000
}
```

```bash
# Simple petition/data capture
curl -X POST \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "suppressAutoResponder": true,
    "standardFieldNames": true,
    "supporter": {
      "emailAddress": "supporter@example.com",
      "firstName": "Jane",
      "lastName": "Doe",
      "questions": {"Opt-in": "Y"}
    }
  }' \
  "https://ca.engagingnetworks.app/ens/service/page/12345/process"
```

---

### POST /page/{id}/process/ (Card Payment Variant) -- WRITE

> **Requires human approval.** This processes a real credit card payment. Use `demo: true` for testing.

Process a page submission containing credit card data. This endpoint **must** use the `-vault` host variant for PCI-compliant tokenization.

**Host:** `https://{region}-vault.engagingnetworks.app/ens/service`

**Path Parameters:**
- `id` (integer, required): Page ID

**Request Body:** Same structure as POST /page/{id}/process, but the `transaction` object includes credit card fields:

```typescript
transaction: {
  donationAmt: number;
  paymenttype: string;
  paycurrency: string;
  ccnumber: string;          // Full card number (tokenized by vault)
  ccexpire: string;          // MM/YY
  ccvv: string;              // CVV
  // ... other transaction fields
}
```

**Response 200:** Same as POST /page/{id}/process

```bash
curl -X POST \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "demo": true,
    "standardFieldNames": true,
    "supporter": {
      "emailAddress": "otto.nv@example.com",
      "firstName": "Otto",
      "lastName": "Normalverbraucher"
    },
    "transaction": {
      "donationAmt": 25,
      "paymenttype": "visa",
      "paycurrency": "CAD",
      "ccnumber": "4111111111111111",
      "ccexpire": "12/25",
      "ccvv": "123",
      "recurrpay": "N"
    }
  }' \
  "https://ca-vault.engagingnetworks.app/ens/service/page/12345/process/"
```

---

### GET /page/{id}/survey

Retrieve survey responses for a survey page.

**Path Parameters:**
- `id` (integer, required): Survey page ID

**Query Parameters:**
- `startDate` (string, optional): ISO 8601 date (YYYY-MM-DD)
- `endDate` (string, optional): ISO 8601 date (YYYY-MM-DD)
- `exportGroupName` (string, optional): Name of export group for additional supporter fields

**Response 200:**
```json
{
  "responses": [...]
}
```

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/12345/survey?startDate=2024-01-01&endDate=2024-12-31"
```
</page_service_endpoints>

---

<page_component_endpoints>
## 4. Page Components

Page components are reusable building blocks that form part of EN page structures. All component list endpoints share the same pagination parameters and response shape.

**Common Pagination Parameters (all list endpoints):**
- `rows` (integer, optional): Number of rows per page
- `start` (integer, optional): Start listing from row N
- `page` (integer, optional): Page number of records
- `folderId` (integer, optional): Dashboard folder ID (0 = Home folder)

**Common Response Shape (all list endpoints):**
```typescript
interface ComponentListItem {
  id: number;
  name: string;
  createdOn: number;    // epoch timestamp
  modifiedOn: number;   // epoch timestamp
  ownedBy: number;      // user ID
  folderId: number;     // dashboard folder ID
}
```

### GET /page/ (List Pages Using a Component)

Find pages where a specific component is used.

**Query Parameters:**
- `action` (string, optional): Must be `InUse`
- `componentId` (integer, optional): The component ID to search for

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/?action=InUse&componentId=5678"
```

**Response 200:** Array of `{campaignId, name, id}`

---

### Code Blocks

Code blocks contain JavaScript or CSS snippets used by pages.

**GET /page/component/codeblock** - List all code blocks
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/codeblock?rows=50&start=0"
```

**GET /page/component/codeblock/{id}** - View a specific code block
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/codeblock/123"
```

---

### Form Blocks

Form blocks contain supporter fields and questions for data collection.

**GET /page/component/formblock** - List all form blocks
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/formblock?rows=50"
```

**GET /page/component/formblock/{id}** - View a specific form block
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/formblock/456"
```

---

### Web Pages (Landing Pages)

Landing pages used for redirects, error conditions, and campaign stop-off points.

**GET /page/component/page** - List all landing pages
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/page?rows=50"
```

---

### HTML Templates

Templates define the look and feel of pages.

**GET /page/component/template** - List all templates
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/template?rows=50"
```

**GET /page/component/template/{id}** - View a template (returns all localized versions)
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/template/789"
```

---

### Text Blocks

Formatted content shared between multiple pages.

**GET /page/component/textblock** - List all text blocks
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/textblock?rows=50"
```

**GET /page/component/textblock/{id}** - View a specific text block
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/textblock/321"
```

---

### Thank You Emails (Auto-Responders)

Auto-responder emails sent after page submission. Can include personalized supporter and transaction data.

**GET /page/component/thankemail** - List all thank-you emails
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/thankemail?rows=50"
```

**GET /page/component/thankemail/{id}** - View a specific thank-you email
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/thankemail/654"
```

---

### Upsell Lightboxes

Prompt donors to convert single donations to recurring or increase donation amounts.

**GET /page/component/upsell** - List all upsell lightboxes
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/upsell?rows=50"
```

---

### Display Widgets

Small graphical displays showing campaign data (petition signatures, donation thermometers, etc.).

**GET /page/component/widget** - List all widgets
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/widget?rows=50"
```

**GET /page/component/widget/{id}** - View widget details
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page/component/widget/987"
```

Widget types: `PROGRESSBAR`, `THERMOMETER`, `IMAGEFILL`

---

### Supporter Questions

Questions that can be included in forms (opt-ins, preferences, survey questions).

**GET /supporter/questions** - List all available questions
```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/questions"
```

**Response 200:**
```json
[
  {"id": 1, "questionId": 100, "name": "Opt-in Email", "type": "OPT"},
  {"id": 2, "questionId": 101, "name": "Comments", "type": "GEN"}
]
```

**GET /supporter/questions/{id}** - View question details including HTML field configuration

**Response 200:**
```json
[
  {
    "id": 1,
    "questionId": 100,
    "name": "Opt-in Email",
    "type": "OPT",
    "locale": "en-US",
    "label": "I'd like to receive email updates",
    "htmlFieldType": "checkbox",
    "content": ""
  }
]
```

HTML field types: `calendar`, `password`, `hidden`, `text`, `textarea`, `radio`, `checkbox`, `select`, `imgselect`, `telephone`, `range`
</page_component_endpoints>

---

<supporter_endpoints>
## 5. Supporter Services

### GET /supporter

Look up a supporter by email address.

**Query Parameters:**
- `email` (string, **required**): Supporter's email address
- `includeQuestions` (boolean, optional): Include question responses
- `includeMemberships` (boolean, optional): Include membership information

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter?email=otto.nv@example.com&includeQuestions=true&includeMemberships=true"
```

**Response 200:**
```json
{
  "supporterId": 212200,
  "suppressed": false,
  "First Name": "Otto",
  "Last Name": "Normalverbraucher",
  "Email Address": "otto.nv@example.com",
  "questions": {"Opt-in Email": "Y"},
  "memberships": []
}
```

---

### GET /supporter/{supporterId}

Get full supporter details by ID.

**Path Parameters:**
- `supporterId` (integer, required): Supporter's unique database ID

**Query Parameters:**
- `includeQuestions` (boolean, optional): Include question responses
- `includeMemberships` (boolean, optional): Include membership information

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200?includeQuestions=true&includeMemberships=true"
```

**Response 200:**
```json
{
  "supporterId": 212200,
  "suppressed": false,
  "questions": {...},
  "memberships": [...]
}
```

---

### POST /supporter -- WRITE

> **Requires human approval.** Creates a new supporter record or updates an existing one if the email already exists.

Create a new supporter or update an existing one (matched by email address).

**Request Body:** `application/json`
```json
{
  "Email Address": "otto.nv@example.com",
  "First Name": "Otto",
  "Last Name": "Normalverbraucher",
  "questions": {
    "Opt-in Email": "Y",
    "Comments": "Signed up via API"
  }
}
```

**Response 200:** `{"id": 212200}`
**Response 400:** `{"message": "...", "messageId": "..."}`

Field names in POST use **display names** (e.g., "Email Address", "First Name") by default. Use `GET /supporter/fields` to discover available field names for your account.

---

### PUT /supporter/{supporterId} -- WRITE

> **Requires human approval.** Modifies an existing supporter's data.

Update a supporter record by ID.

**Path Parameters:**
- `supporterId` (integer, required): Supporter ID

**Request Body:** `application/json` - Only include fields to update

```json
// Change email
{"Email Address": "newemail@example.com"}

// Update contact preferences
{"questions": {"Opt-in Email": "Y", "Opt-in SMS": "N"}}

// Update address
{"Address 1": "42-2630 Hegal Place", "City": "Alexandria", "Region": "VA", "Country": "US"}
```

**Supporter fields available for PUT:**
`Email Address`, `Title`, `First Name`, `Middle Name`, `Last Name`, `Address 1`, `Address 2`, `Address 3`, `Postcode`, `City`, `Region`, `Country`, `Appeal Code`, `Phone Number`, `Anonymous Donor`, `Second Email Address`, `Second Address 1`, `Second Address 2`, `Second City`, `Second Region`, `Second Country`, `Second Phone Number`, `Billing Address 1`, `Billing Address 2`, `Billing Postcode`, `Billing City`, `Billing Region`, `Billing Country`, `Supporter Organization`, `Supporter Chapter`, `Supporter Language`, `Supporter Suffix`, `Supporter Birthday`, `Send Offset`, `questions`

**Response 200:** `{"id": 212200}`

---

### DELETE /supporter/{supporterId} -- DESTRUCTIVE

> **Requires human approval. This action is irreversible.** The supporter record will be permanently deleted from the system.

Permanently remove a supporter record.

**Path Parameters:**
- `supporterId` (integer, required): Supporter ID

```bash
curl -X DELETE \
  -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200"
```

**Response 200:** `{"id": 212200}`

---

### PUT /supporter/bulk -- WRITE

> **Requires human approval.** Modifies suppression status for multiple supporters at once.

Bulk update suppression status for multiple supporters.

**Request Body:** `application/json`
```json
{
  "field": "suppressed",
  "value": true,
  "campaigners": [212200, 212201, 212202]
}
```

**Response 200:** Number of supporters affected.

---

### GET /supporter/fields

List available supporter database fields for your account. Field names are account-specific and needed for POST/PUT operations.

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/fields"
```

**Response 200:**
```json
[
  {"id": 1, "name": "Email Address", "tag": "emailAddress", "property": "core"},
  {"id": 2, "name": "First Name", "tag": "firstName", "property": "core"},
  {"id": 3, "name": "Last Name", "tag": "lastName", "property": "core"}
]
```

---

### GET /supporter/query

Query supporters with various filters. The primary endpoint for bulk supporter data retrieval.

**Query Parameters:**
- `type` (string, required): Query type
  - `latestCreated` - Recently created supporters
  - `latestModified` - Recently modified supporters
  - `suppressed` - Currently suppressed supporters
  - `profile` - Supporters matching an EN profile (requires `profileId`)
  - `search` - Filter-based search (requires `filter`)
- `daysBack` (integer, **required for all query types**): Days of data to return (1-32)
- `rows` (integer, optional): Max rows per page (default: 20, max: 100)
- `start` (integer, optional): Starting row for pagination (1-based, default: 1)
- `exportGroup` (string, optional): Export group name for additional fields (max 100 chars)
- `profileId` (integer, optional): Profile ID (required when `type=profile`)
- `filter` (string, optional): Search filter (required when `type=search`)
  - Syntax: `fieldName:value` where `fieldName` is the field's `property` tag (from GET /supporter/fields), not the display name
  - Operators: `:` (equals), `:>` (greater than), `:<` (less than), `:*` (contains)

**Response 200:**
```json
{
  "pagination": {
    "start": 1,
    "rows": 100,
    "total": 4523
  },
  "data": [
    {
      "supporterId": 212200,
      "emailAddress": "otto.nv@example.com",
      "createdOn": "2022-01-10",
      "modifiedOn": "2023-05-02",
      "scores": [],
      "summary": {}
    }
  ]
}
```

**Example Requests:**
```bash
# Latest modified (last 7 days)
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/query?type=latestModified&daysBack=7&rows=100"

# Search by province
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/query?type=search&filter=province:ON&rows=100"

# Search by language (contains)
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/query?type=search&filter=language:*French*&rows=100"

# Supporters in a specific profile
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/query?type=profile&profileId=1551&rows=100"

# With export group for additional fields
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/query?type=latestModified&daysBack=7&rows=100&exportGroup=Basic%20Address%20Fields"
```

**Filter Syntax Examples:**

Filter field names must use the `property` tag from `GET /supporter/fields` (e.g., `country`, `firstName`, `emailAddress`), NOT display names (e.g., "Country", "First Name", "Email Address").

```
country:CA                     # Country equals CA (property tag: country)
province:ON                    # Province equals Ontario
language:Fran\u00e7ais              # Language equals Fran\u00e7ais
createdOn:>2024-01-01         # Created after date
emailAddress:*@gmail.com      # Email contains pattern (property tag: emailAddress)
firstName:J*                   # First name starts with J
```

**Pagination Pattern:**
```typescript
async function getAllSupporters(queryType: string, daysBack: number): Promise<Supporter[]> {
  const all: Supporter[] = [];
  let start = 1;  // 1-based pagination
  const rows = 100;

  while (true) {
    const response = await client.querySupporters({ type: queryType, daysBack, rows, start });
    all.push(...response.data);

    if (response.data.length < rows || all.length >= response.pagination.total) {
      break;
    }
    start += rows;
  }
  return all;
}
```

---

### Import Formats

Import formats define the field mapping for bulk supporter imports.

**GET /supporter/importformat** - List all import formats

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/importformat"
```

**Response 200:**
```json
[
  {"id": 1, "name": "CRM Import Format", "fields": [...]}
]
```

**POST /supporter/importformat** - Create a new import format -- WRITE

```bash
curl -X POST \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CRM_Contact_Import",
    "format": [
      {"name": "Email Address", "label": "Constituent Email"},
      {"name": "First Name", "label": "Constituent Given Name"},
      {"name": "Last Name", "label": "Constituent Family Name"}
    ]
  }' \
  "https://ca.engagingnetworks.app/ens/service/supporter/importformat"
```

**Response 200:** `{"id": 42}`

**GET /supporter/importformat/{id}** - View a specific import format

**PUT /supporter/importformat/{id}** - Update an import format (same body as POST) -- WRITE

**DELETE /supporter/importformat/{id}** - Delete an import format -- DESTRUCTIVE

**Response 200:** `{"id": 42}`
</supporter_endpoints>

---

<origin_source_endpoints>
## 6. Origin Source

Track how supporters were originally acquired. Origin source codes must be pre-configured in the EN dashboard under Page > Components > Origin Source.

### GET /supporter/{supporterId}/originsource

Retrieve the origin source associated with a supporter.

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200/originsource"
```

**Response 200:**
```json
{
  "code": "FB_AD_2024Q4",
  "category": "socialmedia_facebook"
}
```

**Category values (partial list):** `advocacy`, `fundraising`, `email`, `event`, `socialmedia`, `socialmedia_facebook`, `socialmedia_twitter`, `socialmedia_instagram`, `paid_ads`, `search_engine`, `search_engine_google`, `organic`, `offline`, `data_api`, `tell_a_friend`, `forwarded_email`, `external_system`, and many more.

---

### POST /supporter/{supporterId}/originsource -- WRITE

> **Requires human approval.** Assigns an acquisition source to a supporter record.

Assign an origin source to a supporter (only if they don't already have one).

```bash
curl -X POST \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code": "FB_AD_2024Q4", "category": "socialmedia_facebook"}' \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200/originsource"
```

**Response 200:** `{"id": 212200}`

---

### PUT /supporter/{supporterId}/originsource -- WRITE

> **Requires human approval.** Overwrites the existing origin source for this supporter.

Update (overwrite) the origin source for a supporter.

```bash
curl -X PUT \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"code": "GOOGLE_ADS_2024", "category": "search_engine_google"}' \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200/originsource"
```

**Response 200:** `{"id": 212200}`

---

### DELETE /supporter/{supporterId}/originsource -- DESTRUCTIVE

> **Requires human approval. This action is irreversible.** The origin source association will be permanently removed.

Remove the origin source from a supporter.

```bash
curl -X DELETE \
  -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200/originsource"
```

**Response 200:** `{"id": 212200}`
</origin_source_endpoints>

---

<transaction_endpoints>
## Supporter Transactions

### GET /supporter/{supporterId}/transactions

List all transactions for a supporter. Returns email broadcasts, donations, petition signatures, event registrations, etc.

**Path Parameters:**
- `supporterId` (integer, required): Supporter ID

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200/transactions"
```

**Response fields include:** `id` (internal EN ID), `transactionId` (gateway-defined, for use in detail endpoint), `type`, `subType`, `name`, `createdDate` (epoch), `campaignId`, `status`, `amount`, `recurringPayment`.

**Transaction `type` values** (page/activity type):
- `EMAIL` - Email broadcast delivery record
- `nd` - Net donor (donation)
- `dc` - Data capture (petition, signup)
- `et` - Email to target (advocacy)
- `ev` - Event registration
- `ec` - Ecommerce purchase
- `SBC` - Subscribe/consent record
- `p2p` - Peer-to-peer fundraising

**Transaction `subType` values** (varies by type): `MEMBERSHIP`, `UNSUB`, etc.

**Important:** To get details for a specific transaction, use the `transactionId` field (gateway ID), not the `id` field.

---

### GET /supporter/{supporterId}/transactions/{transactionId}

Get detailed information about a specific transaction.

**Path Parameters:**
- `supporterId` (integer, required): Supporter ID
- `transactionId` (string, required): The gateway-defined transaction identifier (e.g., `ND272532241__A23028626`). This is the `transactionId` field from the transaction list response, NOT the internal `id` field. Using the internal `id` will return 204 with no data.

**Response 200:**
```json
{
  "pageSubType": "standard",
  "recurringFrequency": "MONTHLY",
  "recurringDay": 15,
  "pageTitle": "Monthly Giving",
  "createdOn": 1728569520000,
  "campaignPageId": 12345,
  "campaignId": 67890,
  "ccLastFour": "1234",
  "currency": "CAD",
  "id": 98765,
  "expiry": "12/25",
  "transactionError": "",
  "amount": 25.00,
  "transactionStatus": "success",
  "paymentDay": 15,
  "recurringStatus": "ACTIVE",
  "transactionId": "txn_abc123",
  "parentTransactionId": "",
  "transactionType": "nd",
  "taxDeductible": "Y",
  "recurringPayment": true,
  "gateway": "stripe",
  "child": []
}
```

**Response 204:** No additional information available for this transaction.

**`transactionType` values** (financial instrument type, distinct from the list endpoint's `type` field):
- `CREDIT_SINGLE` - One-time credit card payment
- `CREDIT_RECURRING` - Recurring credit card payment
- `BANK_SINGLE` - One-time bank/ACH payment
- `BANK_RECURRING` - Recurring bank/ACH/EFT payment
- `PAYPAL_SINGLE` - One-time PayPal payment
- `PAYPAL_RECURRING` - Recurring PayPal payment

**`transactionStatus` values:** `success`, `reject`, `refund`, `change`
**`recurringStatus` values:** `ACTIVE`, `PAUSED`, `SUSPENDED`, `CANCELED`, `FIRST_RETRY`, `FINAL_RETRY`

---

### GET /supporter/{supporterId}/transactions/recurring

List a supporter's recurring transactions.

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/212200/transactions/recurring"
```

**Response 200:**
```json
[
  {
    "reason": "",
    "amount": 25.00,
    "paymentDay": 15,
    "campaignId": 67890,
    "currency": "CAD",
    "id": 98765,
    "transactionId": "txn_abc123",
    "startDate": "2024-01-15",
    "nextPaymentDate": "2024-11-15",
    "status": "ACTIVE",
    "frequency": "MONTHLY"
  }
]
```

---

### POST /supporter/{supporterId}/transactions/recurring -- WRITE

> **Requires human approval.** Creates a new recurring payment schedule for a supporter.

Migrate an existing recurring transaction to EN (using a gateway token from the original payment processor).

**Request Body:** `application/json`
```json
{
  "campaignId": 67890,
  "type": "nd",
  "paymentType": "visa",
  "gatewayToken": "tok_abc123",
  "currency": "CAD",
  "amount": 25.00,
  "expiry": "12/25",
  "recurringFrequency": "MONTHLY",
  "recurringDay": 15,
  "recurringStartDate": "2024-01-15",
  "appealCode": "MONTHLY2024",
  "other1": "", "other2": "", "other3": "", "other4": ""
}
```

**Response 200:** `{"id": 98765}` (initial transaction created with amount 0)
**Response 400:** `{"message": "...", "messageId": "...", "developerMessage": "..."}`

---

### PUT /supporter/{supporterId}/transactions/recurring/{transactionId} -- WRITE

> **Requires human approval.** Modifies an active recurring donation (amount, frequency, status, payment date).

Update a recurring transaction (change amount, frequency, status, etc.).

**Path Parameters:**
- `supporterId` (integer, required): Supporter ID
- `transactionId` (string, required): Gateway transaction ID

**Request Body:** `application/json` - Only include fields to update

```json
// Cancel a recurring donation
{"status": "CANCELED", "reason": "Requested by supporter"}

// Change amount
{"amount": 50}

// Change payment day
{"paymentDay": 1}

// Change next payment date
{"nextPaymentDate": "2024-01-01"}

// Pause recurring
{"status": "PAUSED"}
```

**Updatable fields:** `amount`, `frequency`, `paymentDay`, `endDate`, `status`, `reason`, `nextPaymentDate`

**Recurring frequency values:** `MONTHLY`, `QUARTERLY`, `SEMI_ANNUAL`, `ANNUAL`
</transaction_endpoints>

---

<export_job_endpoints>
## 7. Export Job Services

Export jobs run pre-configured queries from the EN dashboard and produce downloadable CSV/Excel files.

### POST /exportjob

Initiate a new export job. Queries must be pre-configured in the dashboard.

**Request Body:** `application/json`
```json
{
  "displayUserDataInTransactionExport": false,
  "applyCustomReferenceNames": false,
  "fileType": "csv",
  "queryName": "Monthly Donors Report",
  "format": "",
  "fieldGroup": ""
}
```

- `fileType`: `csv`, `standardcsv`, `xls`, `xlsx`
- `queryName`: Must match an existing query name in the EN dashboard

**Response 200:**
```json
{
  "id": 456,
  "createdOn": 1728569520000,
  "status": "pending"
}
```

```bash
curl -X POST \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"fileType": "csv", "queryName": "Monthly Donors Report"}' \
  "https://ca.engagingnetworks.app/ens/service/exportjob"
```

---

### GET /exportjob/{id}

Check the status of an export job.

**Path Parameters:**
- `id` (integer, required): Job ID

**Response 200:**
```json
{
  "completedOn": 1728569580000,
  "createdOn": 1728569520000,
  "url": "https://...",
  "status": "completed"
}
```

**Status values:** `pending`, `completed`

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/exportjob/456"
```

---

### GET /exportjob/{id}/download

Download the results of a completed export job as a file.

**Path Parameters:**
- `id` (integer, required): Job ID

**Response 200:** CSV file content (`text/csv`)

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  -o export_results.csv \
  "https://ca.engagingnetworks.app/ens/service/exportjob/456/download"
```

**Export Job Workflow:**
```typescript
async function runExportJob(queryName: string): Promise<string> {
  // 1. Initiate
  const job = await client.post('/exportjob', { fileType: 'csv', queryName });

  // 2. Poll for completion
  let status = 'pending';
  while (status === 'pending') {
    await sleep(5000);  // 5 second intervals
    const check = await client.get(`/exportjob/${job.id}`);
    status = check.status;
  }

  // 3. Download
  const csv = await client.get(`/exportjob/${job.id}/download`);
  return csv;
}
```
</export_job_endpoints>

---

<marketing_automation_endpoints>
## 8. Marketing Automation

### GET /ma

List marketing automations in the account.

**Query Parameters:**
- `folderId` (integer, optional): Dashboard folder ID (0 = Home)
- `name` (string, optional): Filter by name (partial match)

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/ma?name=Welcome"
```

**Response 200:**
```json
[
  {
    "id": 100,
    "clientId": 50,
    "name": "Welcome Series",
    "status": "active",
    "folderId": 0,
    "ownedBy": 1,
    "createdOn": 1705312200000,
    "modifiedOn": 1728569520000
  }
]
```

---

### GET /ma/{id}

Get details about a specific marketing automation.

**Path Parameters:**
- `id` (integer, required): Automation ID

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/ma/100"
```

**Response 200:** Same shape as list item above.

---

### GET /ma/{id}/stats

Retrieve statistics for a marketing automation over a time range.

**Path Parameters:**
- `id` (integer, required): Automation ID

**Query Parameters:**
- `startMonth` (string, optional): First month (YYYYMM format)
- `endMonth` (string, optional): Last month (YYYYMM format)

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/ma/100/stats?startMonth=202401&endMonth=202412"
```

**Response 200:**
```json
{
  "journeyStarts": 5000,
  "averageOpenRate": 42.5,
  "averageClickRate": 8.3,
  "actions": 12500,
  "donations": 350,
  "objectiveReached": 2800,
  "unsubscribes": 45,
  "smsDeliveryRate": 98.2,
  "jumps": 150
}
```

---

### POST /ma/{id}/supporter/bulk -- WRITE

> **Requires human approval.** Adds supporters to a marketing automation journey. They will begin receiving automated communications.

Add supporters in bulk to an active marketing automation. Supporters skip the query qualification step but exclusion filters still apply.

**Path Parameters:**
- `id` (integer, required): Automation ID (must be active)

**Request Body:** `application/json` - Array of supporter email addresses or IDs

```bash
curl -X POST \
  -H "ens-auth-token: SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '["supporter1@example.com", "supporter2@example.com"]' \
  "https://ca.engagingnetworks.app/ens/service/ma/100/supporter/bulk"
```

**Response 200:** Number of supporters added.
**Response 400:** `{"message": "...", "messageId": "..."}`
</marketing_automation_endpoints>
