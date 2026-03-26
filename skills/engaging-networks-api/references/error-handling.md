<overview>
Comprehensive error handling for Engaging Networks API integrations. EN uses standard HTTP status codes with JSON error responses.
</overview>

<error_response_format>
## Standard Error Response

```json
{
  "error": "ErrorType",
  "message": "Human-readable error description",
  "field": "fieldName",  // Optional: which field caused error
  "value": "invalidValue"  // Optional: what value was invalid
}
```
</error_response_format>

<http_status_codes>
## Status Codes Reference

<client_errors>
### 4xx Client Errors (Don't Retry)

<error_400>
**400 Bad Request**

**Causes:**
- Invalid parameter values
- Malformed JSON body
- Missing required parameters
- Invalid date format

**Examples:**
```json
{
  "error": "InvalidParameter",
  "message": "The 'type' parameter must be one of: donation, advocacy, event, survey",
  "field": "type",
  "value": "fundraiser"
}
```

**Solution:**
```typescript
// Validate parameters before sending
const validTypes = ['donation', 'advocacy', 'event', 'survey'];
if (!validTypes.includes(type)) {
  throw new Error(`Invalid type: ${type}`);
}

// Use TypeScript types to prevent invalid values
type PageType = 'donation' | 'advocacy' | 'event' | 'survey';
```
</error_400>

<error_401>
**401 Unauthorized**

**Causes:**
- Invalid API token
- Expired token
- Token not in header
- Malformed token

**Solution:**
```typescript
if (error.response?.status === 401) {
  console.error('Authentication failed. Check EN_API_TOKEN in .env');
  console.error('Regenerate token: EN Admin → Settings → API');
  throw new Error('Invalid EN API token');
}
```

**Quick Test:**
```bash
# Test token manually
curl -v -H "ens-auth-token: YOUR_TOKEN" \
  https://us.engagingnetworks.app/ens/service/page?limit=1
```
</error_401>

<error_403>
**403 Forbidden**

**Causes:**
- Token lacks required permissions
- IP address not whitelisted
- Account access restrictions
- Feature not enabled for your account

**Solution:**
```typescript
if (error.response?.status === 403) {
  console.error('Access forbidden. Possible causes:');
  console.error('- Token lacks permissions');
  console.error('- IP not whitelisted');
  console.error('- Feature not enabled');
  console.error('Contact EN support for access');
  throw new Error('EN API access forbidden');
}
```
</error_403>

<error_404>
**404 Not Found**

**Causes:**
- Page ID doesn't exist
- Page was deleted/archived
- Typo in endpoint or page ID
- Wrong regional instance

**Solution:**
```typescript
async function getPageSafe(pageId: string) {
  try {
    return await client.getPage(pageId);
  } catch (error) {
    if (error.response?.status === 404) {
      console.warn(`Page ${pageId} not found. It may be deleted or archived.`);
      return null;
    }
    throw error;
  }
}

// Verify page exists before fetching details
const pages = await client.getPages();
const pageIds = new Set(pages.map(p => p.id));

if (pageIds.has(targetPageId)) {
  const page = await client.getPage(targetPageId);
} else {
  console.log('Page not found in current pages list');
}
```
</error_404>
</client_errors>

<server_errors>
### 5xx Server Errors (DO Retry)

<error_429>
**429 Too Many Requests (Rate Limit)**

**Handled via exponential backoff** (see rate-limits.md)

```typescript
if (error.response?.status === 429) {
  const retryAfter = error.response.headers['retry-after'] || 60;
  console.log(`Rate limited. Retry after ${retryAfter}s`);

  // Exponential backoff handles this automatically
  return await fetchWithRetry(fn, attempt + 1);
}
```
</error_429>

<error_500>
**500 Internal Server Error**

**Causes:**
- EN server issue
- Database error on EN side
- Temporary service disruption

**Solution:** Retry with fixed delay
```typescript
if (error.response?.status >= 500 && attempt < maxRetries) {
  console.log(`Server error. Retry ${attempt + 1}/${maxRetries} in 2s`);
  await sleep(2000);
  return await fetchWithRetry(fn, attempt + 1);
}
```
</error_500>

<error_503>
**503 Service Unavailable**

**Causes:**
- EN maintenance window
- Planned downtime
- Service overload

**Solution:** Retry with longer delay
```typescript
if (error.response?.status === 503) {
  console.log('EN API unavailable. Check https://status.engagingnetworks.net/');
  // Retry with longer delay
  await sleep(10000);  // 10 seconds
  return await fetchWithRetry(fn, attempt + 1);
}
```
</error_503>
</server_errors>
</http_status_codes>

<network_errors>
## Network-Level Errors

<timeout>
**Request Timeout**

```typescript
try {
  await axios.get(url, { timeout: 30000 });
} catch (error) {
  if (error.code === 'ECONNABORTED') {
    console.error('Request timed out after 30s');
    // Retry or fail gracefully
  }
}
```

**Recommended timeout:** 30 seconds
</timeout>

<connection_refused>
**Connection Refused**

**Causes:**
- Wrong base URL
- Network issues
- Firewall blocking request

**Solution:**
```typescript
try {
  await client.getPages();
} catch (error) {
  if (error.code === 'ECONNREFUSED') {
    console.error('Connection refused. Check:');
    console.error('- Base URL matches your region');
    console.error('- Network connectivity');
    console.error('- Firewall settings');
  }
}
```
</connection_refused>

<dns_resolution>
**DNS Resolution Failed**

```typescript
if (error.code === 'ENOTFOUND') {
  console.error('Cannot resolve hostname. Check internet connection.');
  console.error('Hostname:', error.hostname);
}
```
</dns_resolution>
</network_errors>

<error_handling_pattern>
## Comprehensive Error Handler

```typescript
import axios, { AxiosError } from 'axios';

async function handleENApiError(error: unknown): Promise<never> {
  if (!axios.isAxiosError(error)) {
    throw error;  // Re-throw non-Axios errors
  }

  const status = error.response?.status;
  const data = error.response?.data;

  // Log error details
  console.error('EN API Error:', {
    status,
    message: data?.message || error.message,
    endpoint: error.config?.url,
    method: error.config?.method,
  });

  // Handle specific status codes
  if (status === 401) {
    throw new Error('EN API authentication failed. Check EN_API_TOKEN.');
  }

  if (status === 403) {
    throw new Error('EN API access forbidden. Check permissions.');
  }

  if (status === 404) {
    throw new Error(`EN API resource not found: ${error.config?.url}`);
  }

  if (status === 429) {
    throw new Error('EN API rate limit exceeded. Retry with backoff.');
  }

  if (status && status >= 500) {
    throw new Error(`EN API server error (${status}). Retry later.`);
  }

  // Network errors
  if (error.code === 'ECONNABORTED') {
    throw new Error('EN API request timed out.');
  }

  if (error.code === 'ECONNREFUSED') {
    throw new Error('Cannot connect to EN API. Check base URL.');
  }

  // Unknown error
  throw new Error(`EN API error: ${error.message}`);
}

// Usage in client
try {
  const pages = await client.getPages();
} catch (error) {
  handleENApiError(error);
}
```
</error_handling_pattern>

<error_logging>
## Error Logging Best Practices

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

async function getPageWithLogging(pageId: string) {
  try {
    logger.info('Fetching page', { pageId });
    const page = await client.getPage(pageId);
    logger.info('Page fetched successfully', { pageId, pageName: page.name });
    return page;
  } catch (error) {
    logger.error('Failed to fetch page', {
      pageId,
      error: error.message,
      status: error.response?.status,
      // NEVER log the token!
      // headers: error.config?.headers,  // ❌ Contains token
    });
    throw error;
  }
}
```

**What to Log:**
- ✅ Endpoint URL
- ✅ HTTP method
- ✅ Status code
- ✅ Error message
- ✅ Page/resource ID
- ✅ Timestamp
- ✅ Request parameters (sanitized)

**What NOT to Log:**
- ❌ API tokens
- ❌ Full headers (contain token)
- ❌ PII (emails, names, addresses)
- ❌ Passwords or payment info
</error_logging>

<retry_decision_tree>
## Retry Decision Tree

```
Error Occurred
│
├─ 429 Rate Limit → RETRY with exponential backoff
├─ 500-599 Server Error → RETRY with fixed delay (2s)
├─ Timeout → RETRY (1-2 times max)
├─ Network Error → RETRY (connection issues may be transient)
│
├─ 400 Bad Request → DON'T RETRY (fix the request)
├─ 401 Unauthorized → DON'T RETRY (fix token)
├─ 403 Forbidden → DON'T RETRY (fix permissions)
├─ 404 Not Found → DON'T RETRY (resource doesn't exist)
└─ Other 4xx → DON'T RETRY (client error)
```

**Implementation:**
```typescript
function shouldRetry(error: AxiosError, attempt: number, maxRetries: number): boolean {
  if (attempt >= maxRetries) return false;

  const status = error.response?.status;

  // Retry rate limits and server errors
  if (status === 429 || (status && status >= 500)) {
    return true;
  }

  // Retry network errors
  if (!status && (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT')) {
    return true;
  }

  // Don't retry client errors
  return false;
}
```
</retry_decision_tree>

<graceful_degradation>
## Graceful Degradation

When API is unavailable, provide fallback behavior:

```typescript
async function getPagesWithFallback(): Promise<ENPage[]> {
  try {
    return await client.getPages();
  } catch (error) {
    logger.error('EN API unavailable, using cached data');

    // Return cached data if available
    const cached = await cache.get('pages:list');
    if (cached) {
      logger.warn('Serving stale data from cache');
      return cached;
    }

    // Return empty array as last resort
    logger.error('No cached data available');
    return [];
  }
}
```

**Strategies:**
- Use cached data (with staleness warning)
- Return partial results
- Queue requests for later retry
- Notify user of degraded service
- Skip non-critical operations
</graceful_degradation>
