# Workflow: Debug EN API Issues

<required_reading>
**Read these reference files NOW before debugging:**
1. references/error-handling.md
2. references/authentication.md
3. references/rate-limits.md
</required_reading>

<process>
## Step 1: Identify the Error

Ask the user what error they're seeing:

**Common symptoms:**
- "401 Unauthorized"
- "429 Too Many Requests"
- "404 Not Found"
- "Connection timeout"
- "Empty/unexpected response"
- "Intermittent failures"

Get the exact error message and HTTP status code if available.

## Step 2: Diagnostic Tests

Run these tests to isolate the problem:

### Test 1: Verify Token

```bash
# Test authentication with curl
curl -v \
  -H "ens-auth-token: YOUR_TOKEN_HERE" \
  -H "Accept: application/json" \
  https://us.engagingnetworks.app/ens/service/page?limit=1
```

**Expected:** HTTP 200 with JSON array
**If 401:** Token is invalid/expired
**If 403:** Token lacks permissions
**If connection refused:** Wrong regional URL

### Test 2: Check Regional Instance

Verify you're using the correct base URL:

```bash
# Try each region until one works
for region in us ca eu au; do
  echo "Testing $region..."
  curl -s \
    -H "ens-auth-token: YOUR_TOKEN" \
    "https://$region.engagingnetworks.app/ens/service/page?limit=1" \
    && echo "✓ $region works" || echo "✗ $region failed"
done
```

### Test 3: Rate Limit Check

If getting 429 errors:

```bash
# Check rate limit headers in response
curl -v \
  -H "ens-auth-token: YOUR_TOKEN" \
  https://us.engagingnetworks.app/ens/service/page?limit=1 \
  2>&1 | grep -i "x-ratelimit"
```

Look for:
- `X-RateLimit-Limit`: Your maximum requests
- `X-RateLimit-Remaining`: How many you have left
- `X-RateLimit-Reset`: When limit resets (Unix timestamp)

### Test 4: Validate Request Format

If getting 400 errors, check your parameters:

```typescript
// WRONG: Invalid parameter values
await client.getPages({
  type: 'fundraiser',  // ✗ Invalid type (should be 'donation')
  status: 'published', // ✗ Invalid status (should be 'live')
});

// CORRECT:
await client.getPages({
  type: 'donation',   // ✓ Valid
  status: 'live',     // ✓ Valid
});
```

## Step 3: Common Issues & Fixes

### Issue: 401 Unauthorized

**Causes:**
1. Token is invalid or expired
2. Token not in environment variables
3. Token has extra spaces/quotes
4. Wrong environment file loaded

**Fix:**

```bash
# 1. Check .env file
cat .env | grep EN_API_TOKEN
# Should show: EN_API_TOKEN=abc123...
# NOT: EN_API_TOKEN="abc123..." (no quotes)
# NOT: EN_API_TOKEN = abc123 (no spaces)

# 2. Verify environment loads
# TypeScript
node -e "require('dotenv').config(); console.log(process.env.EN_API_TOKEN?.substring(0, 10) + '...')"

# Python
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(os.environ.get('EN_API_TOKEN', 'NOT FOUND')[:10] + '...')"

# 3. Regenerate token in EN admin if needed
# Settings → API → Generate New Token
```

### Issue: 429 Rate Limit Exceeded

**Causes:**
1. Making too many requests too quickly
2. No retry logic with backoff
3. Multiple processes hitting same account
4. Previous requests still counting toward limit

**Fix:**

Update client to implement exponential backoff:

```typescript
private async fetchWithRetry<T>(
  fn: () => Promise<T>,
  attempt: number = 0
): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    if (axios.isAxiosError(error) && error.response?.status === 429) {
      if (attempt < this.maxRetries) {
        // Exponential backoff: 1s, 2s, 4s, 8s (max 10s)
        const baseDelay = Math.min(1000 * Math.pow(2, attempt), 10000);

        // Add jitter to prevent thundering herd
        const jitter = Math.random() * 1000;
        const delay = baseDelay + jitter;

        console.log(`Rate limited. Retrying in ${delay}ms...`);
        await this.sleep(delay);

        return this.fetchWithRetry(fn, attempt + 1);
      }
    }
    throw error;
  }
}
```

**Also check:**
```bash
# Are multiple processes running?
ps aux | grep node  # or python
# Kill duplicates if found
```

### Issue: 404 Not Found

**Causes:**
1. Page ID doesn't exist
2. Page was deleted/archived
3. Wrong endpoint path
4. Typo in page ID

**Fix:**

```typescript
// 1. List all pages first to verify ID exists
const pages = await client.getPages({ limit: 100 });
console.log('Available page IDs:', pages.map(p => p.id));

// 2. Add defensive check
async function getPageSafe(pageId: string) {
  try {
    return await client.getPage(pageId);
  } catch (error) {
    if (axios.isAxiosError(error) && error.response?.status === 404) {
      console.error(`Page ${pageId} not found. It may be deleted or archived.`);
      return null;
    }
    throw error;
  }
}
```

### Issue: Connection Timeout

**Causes:**
1. Wrong regional URL
2. Network/firewall issues
3. EN service temporarily down
4. IP not whitelisted (if EN requires it)

**Fix:**

```bash
# 1. Test connectivity
ping us.engagingnetworks.app

# 2. Try with increased timeout
curl --max-time 60 \
  -H "ens-auth-token: YOUR_TOKEN" \
  https://us.engagingnetworks.app/ens/service/page

# 3. Check EN status
# Visit: https://status.engagingnetworks.net/
# Or contact EN support if service is down
```

Update client timeout:

```typescript
const client = new ENClient({
  apiToken: process.env.EN_API_TOKEN!,
  region: 'us',
  timeout: 60000, // Increase to 60s
});
```

### Issue: Empty or Unexpected Response

**Causes:**
1. No data matches your filters
2. Pagination offset too high
3. Account has no pages
4. Response format changed

**Fix:**

```typescript
// 1. Remove filters to see all data
const allPages = await client.getPages({ limit: 100 });
console.log('Total pages found:', allPages.length);

// 2. Check supporter transactions
// GET /supporter/{supporterId}/transactions
const transactions = await client.get(`/supporter/${supporterId}/transactions`);
console.log(`Found ${transactions.length} transactions for supporter ${supporterId}`);

// 3. If looking for supporters, use the query endpoint with pagination
// GET /supporter/query?type=latestModified&daysBack=7&rows=100
const supporters = await client.get('/supporter/query', {
  params: { type: 'latestModified', daysBack: 7, rows: 100 },
});
console.log(`Found ${supporters.length} recently modified supporters`);

// 4. Log raw response for inspection
console.log('Raw response:', JSON.stringify(transactions, null, 2));
```

### Issue: Intermittent Failures

**Causes:**
1. Transient network issues
2. EN server occasionally slow
3. Rate limit fluctuations
4. Not retrying server errors (5xx)

**Fix:**

Ensure you retry server errors:

```typescript
private shouldRetry(error: AxiosError, attempt: number): boolean {
  if (attempt >= this.maxRetries) return false;

  const status = error.response?.status;

  // Retry on rate limit (429) OR server errors (500-599)
  return status === 429 || (status !== undefined && status >= 500);
}
```

Add logging:

```typescript
console.log(`Request failed: ${error.message}`);
console.log(`Status: ${error.response?.status}`);
console.log(`Attempt: ${attempt + 1}/${this.maxRetries}`);
```

## Step 4: Enable Debug Logging

Add comprehensive logging to see what's happening:

```typescript
class ENClient {
  private debug: boolean;

  constructor(config: ENClientConfig & { debug?: boolean }) {
    // ... existing code ...
    this.debug = config.debug || false;
  }

  private log(message: string, data?: any) {
    if (this.debug) {
      console.log(`[EN Client] ${message}`, data || '');
    }
  }

  async getPages(params?: any): Promise<ENPage[]> {
    this.log('Fetching pages', params);

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<ENPage[]>('/page', { params });
      this.log(`Received ${response.data.length} pages`);
      return response.data;
    });
  }
}

// Usage:
const client = new ENClient({
  apiToken: process.env.EN_API_TOKEN!,
  region: 'us',
  debug: true, // Enable debug logging
});
```

## Step 5: Test Error Scenarios

Create a test script to verify error handling:

```typescript
async function testErrorHandling() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
    debug: true,
  });

  // Test 1: 404 handling
  console.log('\n=== Test 1: 404 Error ===');
  try {
    await client.getPage('nonexistent-id-12345');
  } catch (error) {
    console.log('✓ 404 handled correctly');
  }

  // Test 2: Invalid parameters (400)
  console.log('\n=== Test 2: 400 Error ===');
  try {
    await client.getPages({ type: 'invalid-type' } as any);
  } catch (error) {
    console.log('✓ 400 handled correctly');
  }

  // Test 3: Rate limit retry
  console.log('\n=== Test 3: Rate Limit ===');
  // Make many rapid requests to trigger 429
  const promises = Array.from({ length: 20 }, (_, i) =>
    client.getPages({ limit: 10 })
  );

  try {
    await Promise.all(promises);
    console.log('✓ Rate limit handled with retries');
  } catch (error) {
    console.log('✗ Rate limit not handled');
  }
}

testErrorHandling();
```

## Step 6: Document the Solution

Once fixed, document what was wrong and how you fixed it:

```markdown
## Issue Summary
- **Error:** 429 Rate Limit Exceeded
- **Cause:** No exponential backoff in retry logic
- **Fix:** Implemented exponential backoff with jitter
- **Files Changed:** src/lib/en-client.ts
- **Verified:** Successfully processed 1000 pages without rate limit errors
```

</process>

<anti_patterns>
Avoid:
- **Ignoring HTTP status codes** - They tell you exactly what's wrong
- **No logging** - You need visibility into what's failing
- **Hardcoded delays** - Use exponential backoff, not fixed sleep
- **Assuming one region** - Always verify correct regional instance
- **Not checking token expiry** - EN tokens can expire
- **Retry on 4xx errors** - Client errors won't succeed on retry
- **No timeout** - Set reasonable timeouts (30-60s)
</anti_patterns>

<success_criteria>
API issues resolved when:
- ✓ Error is identified with specific HTTP status code
- ✓ Root cause is understood (auth, rate limit, network, etc.)
- ✓ Fix is implemented with proper error handling
- ✓ Retry logic works for transient errors
- ✓ Logging added to debug future issues
- ✓ Test script verifies error scenarios work
- ✓ Integration runs successfully without errors
- ✓ Solution is documented for reference
</success_criteria>
