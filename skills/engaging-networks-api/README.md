# Engaging Networks REST API Skill

A Claude Code skill that guides Claude through building, debugging, optimizing, and deploying integrations with the Engaging Networks REST API v6.0.0 for nonprofit fundraising data access.

**Regions:** US | US2 | CA | EU | AU
**Languages:** TypeScript, Python
**Endpoints:** 44 paths across 8 service groups

---

## What This Skill Does

When you ask Claude to work with the Engaging Networks API, this skill provides structured workflows, verified endpoint references, type-safe client patterns, and safety guardrails for write operations.

| Service Group | Description | Endpoints |
|---------------|-------------|-----------|
| Authentication | Session token management | 3 |
| Account | Audit log access | 1 |
| Page Services | Campaign pages, submissions, surveys | 5 |
| Page Components | Templates, code blocks, widgets, form blocks, etc. | 17 |
| Supporter Services | CRUD, queries, fields, import formats | 17 |
| Origin Source | Acquisition channel tracking | 4 |
| Export Job Services | Bulk data exports | 3 |
| Marketing Automation | Automated journeys and stats | 4 |

---

## Installation

**From the Orkestre marketplace:**

```
/plugin marketplace add grassriots/orkestre-engaging-networks
/plugin install ork-engaging-networks@orkestre-engaging-networks
```

**Auto-install in your project** (add to `.claude/settings.json`):

```json
{
  "extraKnownMarketplaces": {
    "orkestre-engaging-networks": {
      "source": { "source": "github", "repo": "grassriots/orkestre-engaging-networks" }
    }
  },
  "enabledPlugins": {
    "ork-engaging-networks@orkestre-engaging-networks": true
  }
}
```

**Local testing:**

```
/plugin marketplace add /path/to/orkestre-engaging-networks
```

---

## Prerequisites

1. **EN API User token** -- Obtain from EN Admin > Settings > API Users
2. **IP whitelisting** -- Your server's IP must be whitelisted in the API User settings (authentication will fail with 401 otherwise)
3. **Environment variables:**
   ```bash
   # .env
   EN_API_USER_TOKEN=your-uuid-token-here
   EN_REGION=ca   # us, us2, ca, eu, or au
   ```
4. **For TypeScript integrations:** Node.js 18+ and TypeScript 5+
5. **For Python integrations:** Python 3.8+

---

## Quick Start

Ask Claude anything related to the Engaging Networks API and the skill will activate:

```
"Build a TypeScript integration with Engaging Networks to sync our donor data"
```

The skill will:
1. Ask clarifying questions (language, region, integration goal)
2. Route to the appropriate workflow
3. Generate production-ready code with authentication, retry logic, and type safety

**Verify your setup manually:**

```bash
# 1. Authenticate
SESSION=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"ens-auth-token":"YOUR_API_USER_TOKEN"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate)
TOKEN=$(echo $SESSION | jq -r '."ens-auth-token"')

# 2. List donation pages
curl -s -H "ens-auth-token: $TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/page?type=nd&status=live" | jq '.[0]'

# 3. Query recent supporters
curl -s -H "ens-auth-token: $TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/supporter/query?type=latestModified&daysBack=1&rows=3" | jq '.pagination'
```

---

## Example Workflows

### Build a new integration

> "I need to build a TypeScript integration with EN to sync donor data to our database. We're on the CA region."

The skill walks you through project setup, creates an authenticated client with session token management, adds endpoints for pages, supporters, and transactions, and generates a runnable entry point.

### Add an endpoint to an existing integration

> "Add recurring transaction tracking to my EN client"

The skill reads your existing code, adds the correct types and methods for `GET /supporter/{id}/transactions/recurring`, and creates a test script.

### Debug API issues

> "Getting 401 errors from EN API on our production server"

The skill diagnoses IP whitelisting issues, token expiry, wrong region, and generates a diagnostic script that checks each possible cause.

### Optimize API usage

> "Our EN integration is hitting rate limits"

The skill implements caching strategies, request batching, and shows how to use export jobs for bulk data instead of per-supporter polling.

### Export bulk data

> "Set up a bulk data export from EN using the Monthly Donors query"

The skill creates the full export job lifecycle: initiate `POST /exportjob`, poll `GET /exportjob/{id}`, download `GET /exportjob/{id}/download`.

### Work with supporters

> "Query supporters modified in the last week and get their transactions"

The skill uses `GET /supporter/query?type=latestModified&daysBack=7` with pagination, then fetches transactions per supporter via `GET /supporter/{id}/transactions`.

---

## API Coverage

### Read-Only Endpoints (safe to execute freely)

- `POST /authenticate` -- Get session token
- `GET /authenticate/{token}` -- Validate token
- `GET /page`, `GET /page/{id}` -- List and view pages
- `GET /page/{id}/survey` -- Survey responses
- `GET /page/component/*` -- All 17 component endpoints
- `GET /supporter`, `GET /supporter/{id}` -- Look up supporters
- `GET /supporter/query` -- Query supporters (latestModified, latestCreated, suppressed, profile, search)
- `GET /supporter/fields`, `GET /supporter/questions` -- Schema discovery
- `GET /supporter/{id}/transactions` -- Transaction history
- `GET /supporter/{id}/transactions/recurring` -- Recurring donations
- `GET /supporter/{id}/originsource` -- Acquisition source
- `POST /exportjob`, `GET /exportjob/{id}`, `GET /exportjob/{id}/download` -- Bulk exports
- `GET /ma`, `GET /ma/{id}`, `GET /ma/{id}/stats` -- Marketing automation

### Write Endpoints (require human approval)

- `POST /supporter` -- Create or update supporter
- `PUT /supporter/{id}` -- Modify supporter
- `PUT /supporter/bulk` -- Bulk suppression update
- `POST /page/{id}/process` -- Submit donation, petition, event registration
- `POST/PUT /supporter/{id}/originsource` -- Set/update origin source
- `PUT /supporter/{id}/transactions/recurring/{txnId}` -- Modify recurring donation
- `POST/PUT /supporter/importformat` -- Manage import formats
- `POST /ma/{id}/supporter/bulk` -- Add supporters to automation

### Destructive Endpoints (require approval + irreversibility warning)

- `DELETE /supporter/{id}` -- Permanently delete supporter
- `DELETE /supporter/{id}/originsource` -- Remove origin source
- `DELETE /supporter/importformat/{id}` -- Remove import format

---

## Security & Write Safety

This skill implements a three-layer safety model to protect production nonprofit donor data:

### Layer 1: Risk Classification

Every endpoint in `references/endpoints.md` is tagged **READ**, **WRITE**, or **DESTRUCTIVE**. Write and destructive endpoints include blockquote warnings.

### Layer 2: Confirmation Gates

The skill instructs Claude to **never execute write or destructive API calls without human approval**. Before any write operation, Claude will:

1. Show you the exact HTTP method, endpoint, and request body
2. Explain in plain language what the call will do
3. For destructive calls, warn that the action is irreversible
4. Wait for your explicit confirmation

### Layer 3: Dry-Run in Generated Code

All write methods in generated TypeScript clients include a `dryRun` option:

```typescript
// Preview what would be sent (no data modified)
await client.updateSupporter(212200, { 'First Name': 'Jane' }, { dryRun: true });
// Output:
// === DRY RUN (no data will be sent) ===
// PUT https://ca.engagingnetworks.app/ens/service/supporter/212200
// Request body: { "First Name": "Jane" }
// =====================================

// Execute after reviewing
await client.updateSupporter(212200, { 'First Name': 'Jane' });
```

---

## Authentication

The EN API uses **session-based authentication** with two token types:

| Token Type | Source | Lifetime | Used For |
|------------|--------|----------|----------|
| API User Token | EN Admin > Settings > API Users | Permanent (rotate every 90 days) | `POST /authenticate` only |
| Session Token | Response from `/authenticate` | ~1 hour (check `expires` field) | All other API calls |

**Correct authentication body format:**

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"ens-auth-token":"YOUR_API_USER_TOKEN"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate
```

**Regional base URLs:**

| Region | Base URL |
|--------|----------|
| US | `https://us.engagingnetworks.app/ens/service` |
| US2 | `https://us2.engagingnetworks.app/ens/service` |
| CA | `https://ca.engagingnetworks.app/ens/service` |
| EU | `https://eu.engagingnetworks.app/ens/service` |
| AU | `https://au.engagingnetworks.app/ens/service` |

Card payment processing uses the `-vault` host variant (e.g., `ca-vault.engagingnetworks.app`).

---

## Skill Architecture

```
engaging-networks-api/
├── SKILL.md                          # Main skill definition (intake, routing, safety rules)
├── README.md                         # This file
├── references/
│   ├── endpoints.md                  # All 44 endpoints with risk tags and examples
│   ├── authentication.md             # Auth flow, token types, security best practices
│   ├── typescript-client.md          # Full TypeScript client with dryRun pattern
│   ├── common-patterns.md            # 11 real-world integration patterns
│   ├── error-handling.md             # HTTP status codes, retry logic, graceful degradation
│   └── rate-limits.md                # Limits, backoff, caching, optimization
├── workflows/
│   ├── build-integration.md          # Create new integration from scratch
│   ├── add-endpoint.md               # Add endpoint to existing integration
│   ├── debug-api-issues.md           # Troubleshoot connection/auth/rate issues
│   ├── optimize-api-usage.md         # Reduce API calls, implement caching
│   ├── test-integration.md           # Unit and integration testing
│   └── deploy-integration.md         # Production deployment (Vercel, Lambda, Docker)
└── .audit/
    ├── 2026-03-25-api-verification.md   # Live API test results
    └── 2026-03-25-subagent-tests.md     # Workflow validation results
```

**How progressive disclosure works:**

1. **Always loaded** (~100 words): Skill name and description from YAML frontmatter
2. **Loaded when skill triggers** (<200 lines): SKILL.md body with intake, routing, safety rules
3. **Loaded on demand** (unlimited): Reference files and workflows, read only when needed

---

## Verified & Tested

This skill has been verified against the live Engaging Networks API and tested with subagent workflow runs.

### Live API Verification (2026-03-25)

- **31 read-only endpoints tested** against the CA region
- **25 passed**, 4 blocked by API User permissions, 3 bugs found and fixed
- Tested with real Oxfam Canada account data (239 donation pages, 48K+ supporters)

### Subagent Workflow Tests (2026-03-25)

| Test Case | Workflow | Result |
|-----------|----------|--------|
| Build TypeScript Integration | build-integration.md | PASS |
| Add Supporter Endpoints | add-endpoint.md | PASS |
| Debug 401 Auth Errors | debug-api-issues.md | PASS |
| Export Job Workflow | add-endpoint.md (export) | PASS |
| Recurring Transactions | add-endpoint.md (recurring) | PASS |

### Bugs Found & Fixed: 8 total

- Authentication body format corrected (JSON object, not quoted string)
- `daysBack` parameter documented as required for supporter queries
- Transaction detail endpoint clarified (uses gateway `transactionId`, not internal `id`)
- Pagination `start` corrected to 1-based
- Page detail `template` field documented
- Transaction type codes documented (list `type` vs detail `transactionType`)
- Recurring transaction paths fixed (`/transactions/recurring`)
- Search filter syntax clarified (uses field `property` tags, not display names)

Full reports: `.audit/2026-03-25-api-verification.md` and `.audit/2026-03-25-subagent-tests.md`

---

## Troubleshooting

### 401 Unauthorized on authentication

**Most likely cause:** Your server's IP address is not whitelisted.

1. Get your IP: `curl ifconfig.me`
2. Add it in EN Admin > Settings > API Users > [Your User] > IP Whitelist
3. Also check: token format (UUID), no extra spaces in `.env`, correct region

### 401 on subsequent API calls

Session token expired. The skill generates clients with automatic re-authentication (1 minute before expiry).

### 429 Too Many Requests

Rate limit exceeded (5,000 requests/hour). The skill generates clients with exponential backoff and jitter. For bulk operations, use export jobs instead of per-supporter polling.

### Wrong region

Check your EN admin panel URL. If it starts with `ca.engagingnetworks.app`, use `EN_REGION=ca`.

### Debug workflow

Ask Claude: *"My EN API integration isn't working, help me debug it"* -- the skill routes to `workflows/debug-api-issues.md` which generates a diagnostic script checking token, IP, connectivity, and permissions.

---

## Rate Limits

| Limit | Value | Scope |
|-------|-------|-------|
| API requests | 5,000/hour | Per API User |
| Page processing | 200/5 minutes | Per IP address |
| Pagination max | 100 rows | Per request |
| Supporter query daysBack | 1-32 days | Per query |

---

## License

Part of the [Orkestre Plugins](https://github.com/grassriots/orkestre-engaging-networks) marketplace. See repository for license details.
