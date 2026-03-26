<overview>
Engaging Networks enforces rate limits to protect API infrastructure. Understanding and handling these limits is critical for reliable integrations.
</overview>

<current_limits>
## EN API Rate Limits

<hard_limits>
**Per API User:**
- **5,000 requests per hour** - Main API limit
- **Resets:** Every hour from first request

**Per IP Address:**
- **200 page processing requests per 5 minutes** - For POST/PUT operations
- **Resets:** Every 5 minutes

**Response When Exceeded:**
- HTTP Status: `429 Too Many Requests`
- Response Body:
  ```json
  {
    "error": "Rate limit exceeded",
    "message": "Too many requests. Please try again later.",
    "retryAfter": 3600
  }
  ```
</hard_limits>

<rate_limit_headers>
**Response Headers:**
```
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4847
X-RateLimit-Reset: 1698336000  # Unix timestamp
```

**Track Usage:**
```typescript
const headers = response.headers;
console.log(`Remaining: ${headers['x-ratelimit-remaining']} / ${headers['x-ratelimit-limit']}`);

const resetTime = new Date(parseInt(headers['x-ratelimit-reset']) * 1000);
console.log(`Resets at: ${resetTime.toISOString()}`);
```
</rate_limit_headers>
</current_limits>

<exponential_backoff>
## Exponential Backoff Strategy

<implementation>
**Algorithm:**
1. First retry: Wait 1 second
2. Second retry: Wait 2 seconds
3. Third retry: Wait 4 seconds
4. Fourth retry: Wait 8 seconds
5. Maximum wait: 10 seconds

**With Jitter:**
Add random 0-1 second to prevent thundering herd

```typescript
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  attempt: number = 0
): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    if (error.response?.status === 429 && attempt < maxRetries) {
      // Calculate backoff
      const baseDelay = Math.min(1000 * Math.pow(2, attempt), 10000);
      const jitter = Math.random() * 1000;
      const delay = baseDelay + jitter;

      console.log(`Rate limited. Retry ${attempt + 1}/${maxRetries} in ${delay}ms`);

      await sleep(delay);
      return fetchWithRetry(fn, maxRetries, attempt + 1);
    }
    throw error;
  }
}
```

**Python Implementation:**
```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

class RateLimitError(Exception):
    pass

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(RateLimitError),
    reraise=True
)
def fetch_with_retry():
    # Your API call here
    pass
```
</implementation>

<when_to_retry>
**DO Retry:**
- HTTP 429 (Rate Limit)
- HTTP 500-599 (Server Errors)
- Connection timeouts
- Network errors

**DON'T Retry:**
- HTTP 400 (Bad Request) - Fix the request
- HTTP 401 (Unauthorized) - Check token
- HTTP 403 (Forbidden) - Check permissions
- HTTP 404 (Not Found) - Resource doesn't exist
</when_to_retry>
</exponential_backoff>

<optimization_strategies>
## Reduce API Calls

<caching>
**Cache Static Data:**
```typescript
// Page metadata changes rarely - cache 24 hours
cache.set(`page:${pageId}`, pageData, 86400);

// Past statistics never change - cache forever
if (endDate < today) {
  cache.set(`stats:${pageId}:${dates}`, stats, 0);  // ttl=0 = never expire
}

// Today's statistics change - cache 1 hour
cache.set(`stats:${pageId}:${today}`, stats, 3600);
```

**When to Cache:**
- ✅ Page list (1 hour)
- ✅ Page details (24 hours)
- ✅ Past statistics (forever)
- ✅ Page templates (never change)

**When NOT to Cache:**
- ❌ Real-time transaction data
- ❌ Current day statistics (if need live data)
- ❌ Supporter search results
</caching>

<batching>
**Batch Requests:**
```typescript
// ❌ BAD: Sequential requests
for (const pageId of pageIds) {
  await client.getPage(pageId);  // 100 API calls
}

// ✅ GOOD: Parallel batches
const batchSize = 5;  // Stay under rate limits
for (let i = 0; i < pageIds.length; i += batchSize) {
  const batch = pageIds.slice(i, i + batchSize);
  await Promise.all(batch.map(id => client.getPage(id)));
  await sleep(1000);  // Pace batches
}
```
</batching>

<aggregation>
**Use Export Jobs for Bulk Data:**
```typescript
// ❌ BAD: Fetch transactions per supporter one at a time (100+ API calls)
let total = 0;
for (const supporter of supporters) {
  const txns = await client.get(`/supporter/${supporter.supporterId}/transactions`);
  total += txns.reduce((sum: number, t: any) => sum + t.amount, 0);
}

// ✅ GOOD: Use export job for bulk data (3 API calls total)
const job = await client.post('/exportjob', { fileType: 'csv', queryName: 'Donation Report' });
// Poll until complete, then download
const csv = await client.get(`/exportjob/${job.id}/download`);
```
</aggregation>

<local_database>
**Sync to Local Database:**
```typescript
// Sync pages once per day
async function dailySync() {
  const pages = await client.getPages({ type: 'donation', status: 'live' });

  for (const page of pages) {
    await db.pages.upsert({ enPageId: page.id, ...page });
  }
}

// Query local DB instead of EN API
const activePages = await db.pages.findMany({ status: 'live' });
```
</local_database>
</optimization_strategies>

<request_queue>
## Rate-Limited Queue

<implementation>
**Control Request Throughput:**
```typescript
class RateLimitedClient {
  private queue: Array<() => Promise<any>> = [];
  private requestsThisHour = 0;
  private hourStartTime = Date.now();
  private maxRequestsPerHour = 4500;  // Stay under 5000 limit
  private processing = false;

  async enqueue<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          const result = await fn();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });

      this.processQueue();
    });
  }

  private async processQueue() {
    if (this.processing) return;
    this.processing = true;

    while (this.queue.length > 0) {
      // Reset counter if hour passed
      if (Date.now() - this.hourStartTime > 3600000) {
        this.requestsThisHour = 0;
        this.hourStartTime = Date.now();
      }

      // Wait if at limit
      if (this.requestsThisHour >= this.maxRequestsPerHour) {
        const waitTime = 3600000 - (Date.now() - this.hourStartTime);
        console.log(`Rate limit approaching. Waiting ${waitTime}ms...`);
        await sleep(waitTime);
        this.requestsThisHour = 0;
        this.hourStartTime = Date.now();
      }

      const request = this.queue.shift();
      if (request) {
        await request();
        this.requestsThisHour++;
        await sleep(200);  // 200ms between requests (max 300/min)
      }
    }

    this.processing = false;
  }
}
```

**Usage:**
```typescript
const client = new RateLimitedClient(config);

// All requests automatically queued and rate-limited
await client.enqueue(() => originalClient.getPages());
await client.enqueue(() => originalClient.getPage('123'));
```
</implementation>
</request_queue>

<monitoring>
## Monitor API Usage

```typescript
class MonitoredClient extends ENClient {
  private metrics = {
    totalRequests: 0,
    rateLimitHits: 0,
    requestsByEndpoint: new Map<string, number>(),
    averageLatency: 0,
  };

  async getPages(params?: any): Promise<ENPage[]> {
    const start = Date.now();
    this.metrics.totalRequests++;
    this.metrics.requestsByEndpoint.set(
      'getPages',
      (this.metrics.requestsByEndpoint.get('getPages') || 0) + 1
    );

    try {
      return await super.getPages(params);
    } catch (error) {
      if (error.response?.status === 429) {
        this.metrics.rateLimitHits++;
      }
      throw error;
    } finally {
      const latency = Date.now() - start;
      this.updateAverageLatency(latency);
    }
  }

  printMetrics() {
    console.log('=== EN API Metrics ===');
    console.log(`Total Requests: ${this.metrics.totalRequests}`);
    console.log(`Rate Limit Hits: ${this.metrics.rateLimitHits}`);
    console.log(`Hit Rate: ${(this.metrics.rateLimitHits / this.metrics.totalRequests * 100).toFixed(2)}%`);
    console.log(`Average Latency: ${this.metrics.averageLatency.toFixed(0)}ms`);
    console.log('\nRequests by Endpoint:');
    this.metrics.requestsByEndpoint.forEach((count, endpoint) => {
      console.log(`  ${endpoint}: ${count}`);
    });
  }
}
```

**Alert When:**
- Rate limit hits > 1% of requests
- Hourly requests approaching 4500
- Queue depth > 100 pending requests
</monitoring>

<best_practices>
## Best Practices Summary

1. **Implement Exponential Backoff**
   - Always retry 429 with exponential delays
   - Add jitter to prevent thundering herd
   - Max 3 retries recommended

2. **Cache Aggressively**
   - Page metadata: 24 hours
   - Past statistics: Forever
   - Page list: 1 hour

3. **Batch Operations**
   - Parallel requests in small batches (5-10)
   - Add delays between batches
   - Monitor queue depth

4. **Use Right Endpoints**
   - Statistics for aggregates (not transactions)
   - Pagination with date filters
   - Limit parameter set to max (100)

5. **Schedule Wisely**
   - Run bulk operations during off-peak hours
   - Spread requests evenly throughout hour
   - Avoid burst patterns

6. **Monitor Usage**
   - Track requests per hour
   - Log rate limit hits
   - Alert when approaching limits
   - Review patterns monthly

7. **Test Limits**
   - Verify retry logic works
   - Simulate rate limit scenarios
   - Load test before production
</best_practices>
