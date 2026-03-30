---
name: engaging-networks-api
description: Engaging Networks Public API integration for nonprofit data synchronization and reporting. Use when implementing Engaging Networks Public API calls, building campaign widgets, creating donor rolls, fetching supporter counts, retrieving donation summaries, or integrating EN data with external systems. Supports TypeScript and Python implementations. When Claude needs to fetch campaign data, donation statistics, supporter information, broadcast metrics, or event details from Engaging Networks.
---

<objective>
Enables Claude to build, debug, optimize, and deploy integrations with the Engaging Networks REST API for nonprofit fundraising data access.
</objective>

<essential_principles>

<authentication_overview>
The Engaging Networks REST API provides programmatic access to fundraising pages, supporter data, and campaign metrics for nonprofit organizations.

**Session-based authentication (two-step process):**
1. Obtain API User token from EN admin (Settings -> API Users)
2. POST to /authenticate with API User token to get session token
3. Use session token in `ens-auth-token` header for all subsequent requests
4. Session tokens expire (typically 1 hour) - re-authenticate when expired
5. Store API User token securely in environment variables
6. Never expose tokens in client-side code or logs
7. Ensure IP addresses are whitelisted in API User settings

**Two token types:**
- **API User Token:** Permanent UUID from EN Admin, used ONLY for POST /authenticate
- **Session Token:** Temporary token from /authenticate response, used for all other API calls
</authentication_overview>

<rate_limits_overview>
**Hard limits:**
- 5,000 requests per hour per API user (EN-specific)
- 200 page processing requests per 5 minutes per IP
- HTTP 429 response when exceeded

**Best practices:**
- Implement exponential backoff with jitter
- Cache page metadata (changes infrequently)
- Use maximum pagination limit (100)
- Schedule data collection during off-peak hours
</rate_limits_overview>

<regional_instances>
**EN has multiple deployments - use the correct one:**
- US: `https://us.engagingnetworks.app/ens/service`
- US2: `https://us2.engagingnetworks.app/ens/service`
- CA: `https://ca.engagingnetworks.app/ens/service`
- EU: `https://eu.engagingnetworks.app/ens/service`
- AU: `https://au.engagingnetworks.app/ens/service`

Check your EN admin panel URL to determine your region.
</regional_instances>

<data_model>
**Core entities:**
- **Pages**: Campaign pages (donation, advocacy, event, survey) with configuration
- **Page Components**: Reusable building blocks (code blocks, form blocks, templates, text blocks, thank-you emails, upsell lightboxes, display widgets)
- **Supporters**: Contact records with fields, questions, memberships, and suppression status
- **Transactions**: Donation/payment records accessed via supporter (NOT via page)
- **Recurring Transactions**: Recurring payment schedules per supporter
- **Origin Sources**: Acquisition channel tracking per supporter
- **Export Jobs**: Asynchronous bulk data exports from pre-configured queries
- **Marketing Automations**: Automated supporter journeys with stats
- **Account**: Audit log access

**Typical integration flow:**
1. Authenticate (POST /authenticate) to get session token
2. List pages by type (GET /page?type=dc&status=live)
3. Look up supporters (GET /supporter?email=... or GET /supporter/query)
4. Access supporter transactions (GET /supporter/{id}/transactions)
5. Manage origin sources (GET/POST/PUT/DELETE /supporter/{id}/originsource)
6. Run export jobs for bulk data (POST /exportjob, poll, download)
7. Process page submissions (POST /page/{id}/process)
</data_model>

<error_handling_overview>
**Retry on:**
- 429 (rate limit) - with exponential backoff
- 5xx (server errors) - with fixed delay

**Don't retry:**
- 400 (bad request) - fix the request
- 401 (unauthorized) - check token
- 403 (forbidden) - check permissions
- 404 (not found) - resource doesn't exist
</error_handling_overview>

<data_privacy>
**This skill is a development assistant.** Its primary purpose is helping you write, debug, and deploy code that uses the EN API — not acting as a live data operations proxy.

**Recommended: use a sandbox or test EN account.** All workflows work without real donor data.

**Data flow awareness:** When the agent executes EN API calls, response data is processed by the model provider's infrastructure (Anthropic, OpenAI, etc.). Any supporter PII in API responses — names, emails, addresses, donation history — is transferred to a third party during inference. Read operations expose PII even though they don't modify data.

**Payment data restriction:** Never route credit card data through the agent. The `-vault` host variant for payment processing must only be used in direct server-to-EN connections in application code, not through agent sessions.

**If connecting to a production account:** query only what you need, avoid bulk supporter retrieval through the agent, inform your organization's data protection officer, and clear conversation history after working with live donor data. See SECURITY.md in the repository root for full guidance.
</data_privacy>

<write_safety>
**The EN API has endpoints that modify production nonprofit donor data. Treat write operations with the same care as database migrations.**

**Risk classification:**
- **READ** (GET requests, POST /authenticate, POST /exportjob): Execute freely
- **WRITE** (POST/PUT that create or modify data): Require human approval before execution
- **DESTRUCTIVE** (DELETE operations): Require human approval + explicit irreversibility warning

**Before executing any WRITE or DESTRUCTIVE API call:**
1. Show the user the exact HTTP method, endpoint path, and full request body
2. Explain in plain language what the call will do (e.g., "This will update supporter 212200's email address to newemail@example.com")
3. For DESTRUCTIVE calls, warn: "This action is irreversible -- the record will be permanently deleted"
4. Wait for explicit user confirmation before proceeding
5. Never batch write operations silently -- each write should be visible and approved

**WRITE endpoints:** POST /supporter, PUT /supporter/{id}, PUT /supporter/bulk, POST /page/{id}/process, POST/PUT /supporter/{id}/originsource, PUT /supporter/{id}/transactions/recurring/{txnId}, POST/PUT/DELETE /supporter/importformat, POST /ma/{id}/supporter/bulk

**DESTRUCTIVE endpoints:** DELETE /supporter/{id}, DELETE /supporter/{id}/originsource, DELETE /supporter/importformat/{id}

**When generating client code**, always include a `dryRun` option on write methods so users can preview what would be sent without actually executing it. See `references/typescript-client.md` for the pattern.
</write_safety>

</essential_principles>

<intake>
**Before we begin:** Are you working with a **sandbox/test account** or a **production account**?

- **Sandbox (recommended):** Great — we can work freely. All workflows are fully supported.
- **Production:** We can proceed, but I'll prioritize schema-only endpoints and avoid bulk data queries. Read SECURITY.md for what production use means for your data. I'll remind you when an operation would pull supporter PII into this session.

What would you like to do?

1. Build a new EN API integration
2. Add an endpoint to existing integration
3. Debug API connection/auth/rate limit issues
4. Optimize API usage (reduce calls, caching)
5. Work with supporters (query, create, update, delete)
6. Work with transactions and recurring donations
7. Manage page components (templates, code blocks, widgets)
8. Set up export jobs for bulk data
9. Work with marketing automations
10. Test the integration
11. Deploy to production
12. Something else

**Wait for response, then route to appropriate workflow.**
</intake>

<routing>
| Response | Workflow |
|----------|----------|
| 1, "new", "create", "build", "start", "integration" | `workflows/build-integration.md` |
| 2, "add", "endpoint", "new endpoint", "extend" | `workflows/add-endpoint.md` |
| 3, "debug", "broken", "error", "401", "429", "auth", "connection" | `workflows/debug-api-issues.md` |
| 4, "optimize", "slow", "rate limit", "cache", "reduce calls" | `workflows/optimize-api-usage.md` |
| 5, "supporters", "query", "create supporter", "update supporter" | `workflows/add-endpoint.md` (supporter service group) |
| 6, "transactions", "donations", "recurring", "payments" | `workflows/add-endpoint.md` (transaction service group) |
| 7, "components", "templates", "code blocks", "widgets", "form blocks" | `workflows/add-endpoint.md` (page component service group) |
| 8, "export", "bulk data", "export job", "download" | `workflows/add-endpoint.md` (export job service group) |
| 9, "automations", "marketing", "journeys" | `workflows/add-endpoint.md` (marketing automation service group) |
| 10, "test", "tests", "validate", "verify" | `workflows/test-integration.md` |
| 11, "deploy", "production", "ship", "release" | `workflows/deploy-integration.md` |
| 12, other | Ask clarifying questions, then select appropriate workflow or references |

**After routing, read the workflow file and follow it exactly.**
</routing>

<reference_index>
All in `references/`:

**API Fundamentals:**
- authentication.md - Token management, security best practices
- endpoints.md - Complete endpoint reference for all 8 API service groups (44 paths)
- rate-limits.md - Handling limits, backoff strategies
- error-handling.md - Error codes, retry logic, debugging

**Implementation:**
- typescript-client.md - Type-safe clients with TypeScript
- common-patterns.md - Data collection, synchronization, widgets
</reference_index>

<workflows_index>
All in `workflows/`:

| File | Purpose |
|------|---------|
| build-integration.md | Create new EN API integration from scratch |
| add-endpoint.md | Add new API endpoint usage to existing integration |
| debug-api-issues.md | Troubleshoot connection, auth, rate limit issues |
| optimize-api-usage.md | Reduce API calls, implement caching, handle rate limits |
| test-integration.md | Validate API integration with tests |
| deploy-integration.md | Production deployment considerations |
</workflows_index>

<verification_loop>
Always verify your EN API integration. These checks use **schema and metadata endpoints only** (no supporter PII):

```bash
# 1. Can we authenticate?
SESSION_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d "{\"ens-auth-token\":\"$EN_API_USER_TOKEN\"}" \
  https://ca.engagingnetworks.app/ens/service/authenticate | jq -r '."ens-auth-token"')

echo "Session token obtained: ${SESSION_TOKEN:0:10}..."

# 2. List pages (metadata only, no supporter PII)
curl -H "ens-auth-token: $SESSION_TOKEN" \
  -H "Accept: application/json" \
  "https://ca.engagingnetworks.app/ens/service/page?type=dc"

# 3. List supporter fields (schema only, no PII)
curl -H "ens-auth-token: $SESSION_TOKEN" \
  -H "Accept: application/json" \
  "https://ca.engagingnetworks.app/ens/service/supporter/fields"

# 4. Do types compile? (TypeScript)
npm run typecheck

# 5. Do tests pass?
npm test
```

**Report to user:**
- "Authenticated with EN REST API (region: CA)"
- "Session token obtained (expires in 60 minutes)"
- "Listed pages and supporter field schema"
- "Tests pass: 12/12"
- "Ready for you to test with your EN account"
</verification_loop>

<success_criteria>
- Authenticated successfully with the Engaging Networks REST API using session-based auth
- Data fetched correctly from the target endpoints with proper pagination
- TypeScript types compile without errors
- All tests pass (unit, integration)
- No API tokens or secrets exposed in code, logs, or client-side bundles
- Rate limits respected with exponential backoff implemented
- Error handling covers all expected HTTP status codes
</success_criteria>
