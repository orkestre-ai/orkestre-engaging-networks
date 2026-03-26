# Workflow: Optimize EN API Usage

<required_reading>
**Read these reference files NOW:**
1. references/rate-limits.md
2. references/caching-strategies.md
3. references/common-patterns.md
</required_reading>

<process>
## Step 1: Identify Optimization Target

Ask the user what they want to optimize:

**Common goals:**
- Reduce API calls (hitting rate limits)
- Speed up response times
- Lower costs (if EN charges per API call)
- Handle large datasets efficiently
- Improve reliability

**Current pain points:**
- How many requests per hour?
- Which endpoints are called most?
- What data changes frequently vs rarely?

## Step 2: Implement Caching

Cache data that doesn't change often to reduce API calls.

### Page Metadata Caching

Pages rarely change configuration. Cache for 24 hours:

```typescript
import NodeCache from 'node-cache';

class CachedENClient extends ENClient {
  private cache: NodeCache;

  constructor(config: ENClientConfig) {
    super(config);
    this.cache = new NodeCache({
      stdTTL: 86400, // 24 hours
      checkperiod: 3600, // Check for expired keys every hour
    });
  }

  async getPage(pageId: string): Promise<ENPage> {
    const cacheKey = `page:${pageId}`;

    // Check cache first
    const cached = this.cache.get<ENPage>(cacheKey);
    if (cached) {
      console.log(`Cache hit for page ${pageId}`);
      return cached;
    }

    // Cache miss - fetch from API (GET /page/{id} returns same shape as list)
    console.log(`Cache miss for page ${pageId} - fetching from API`);
    const page = await client.get(`/page/${pageId}`);

    // Store in cache
    this.cache.set(cacheKey, page);

    return page;
  }

  async getPages(params?: any): Promise<ENPage[]> {
    const cacheKey = `pages:${JSON.stringify(params || {})}`;

    const cached = this.cache.get<ENPage[]>(cacheKey);
    if (cached) {
      console.log('Cache hit for pages list');
      return cached;
    }

    const pages = await super.getPages(params);
    this.cache.set(cacheKey, pages, 3600); // Cache for 1 hour

    return pages;
  }

  // Clear cache when needed
  invalidateCache(pattern?: string) {
    if (pattern) {
      const keys = this.cache.keys().filter(key => key.includes(pattern));
      this.cache.del(keys);
      console.log(`Invalidated ${keys.length} cache keys matching: ${pattern}`);
    } else {
      this.cache.flushAll();
      console.log('Cleared entire cache');
    }
  }
}
```

### Export Job Result Caching

Export job results for past date ranges never change. Cache indefinitely:

```typescript
async getExportData(
  exportType: string,
  startDate: string,
  endDate: string
): Promise<any[]> {
  const today = new Date().toISOString().split('T')[0];

  // If end date is in the past, cache forever (historical data won't change)
  const isPastData = endDate < today;
  const ttl = isPastData ? 0 : 3600; // 0 = never expire, 3600 = 1 hour

  const cacheKey = `export:${exportType}:${startDate}:${endDate}`;

  const cached = this.cache.get<any[]>(cacheKey);
  if (cached) {
    console.log(`Cache hit for export data (past data: ${isPastData})`);
    return cached;
  }

  // Create export job, poll until complete, then download
  const job = await client.post('/exportjob', {
    type: exportType,
    startDate,
    endDate,
  });

  // Poll for completion
  let status = job;
  while (status.status !== 'complete') {
    await sleep(2000);
    status = await client.get(`/exportjob/${job.id}`);
  }

  const data = await client.get(`/exportjob/${job.id}/download`);
  this.cache.set(cacheKey, data, ttl);

  return data;
}
```

## Step 3: Batch Requests

Instead of making requests sequentially, batch them:

### Parallel Requests (Within Rate Limits)

```typescript
async function fetchAllPageDetails(pageIds: string[]): Promise<ENPage[]> {
  // Fetch in batches of 5 to stay under rate limits
  const batchSize = 5;
  const results: ENPage[] = [];

  for (let i = 0; i < pageIds.length; i += batchSize) {
    const batch = pageIds.slice(i, i + batchSize);

    console.log(`Fetching batch ${i / batchSize + 1}...`);

    // Fetch batch in parallel (GET /page/{id} returns same shape as list)
    const batchResults = await Promise.all(
      batch.map(id => client.get(`/page/${id}`))
    );

    results.push(...batchResults);

    // Brief pause between batches to avoid rate limits
    if (i + batchSize < pageIds.length) {
      await sleep(1000); // 1 second between batches
    }
  }

  return results;
}
```

### Rate-Limited Queue

Implement a request queue to control throughput:

```typescript
class RateLimitedENClient extends CachedENClient {
  private queue: Array<() => Promise<any>> = [];
  private processing = false;
  private requestsThisHour = 0;
  private hourStartTime = Date.now();
  private maxRequestsPerHour = 4500; // Stay under 5000 limit

  private async processQueue() {
    if (this.processing) return;
    this.processing = true;

    while (this.queue.length > 0) {
      // Reset counter if hour has passed
      if (Date.now() - this.hourStartTime > 3600000) {
        this.requestsThisHour = 0;
        this.hourStartTime = Date.now();
      }

      // Wait if we're at the limit
      if (this.requestsThisHour >= this.maxRequestsPerHour) {
        const waitTime = 3600000 - (Date.now() - this.hourStartTime);
        console.log(`Rate limit reached. Waiting ${waitTime}ms...`);
        await sleep(waitTime);
        this.requestsThisHour = 0;
        this.hourStartTime = Date.now();
      }

      const request = this.queue.shift();
      if (request) {
        await request();
        this.requestsThisHour++;

        // Pace requests (1 per second max)
        await sleep(1000);
      }
    }

    this.processing = false;
  }

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

  // Override methods to use queue
  async getPage(pageId: string): Promise<ENPage> {
    return this.enqueue(() => super.getPage(pageId));
  }

  async getPages(params?: any): Promise<ENPage[]> {
    return this.enqueue(() => super.getPages(params));
  }
}
```

## Step 4: Optimize Data Retrieval

Fetch only what you need using the right approach:

### Per-Supporter Transactions

Transactions are accessed per-supporter via `GET /supporter/{supporterId}/transactions`:

```typescript
async function getRecentSupporterTransactions(
  supporterId: string
): Promise<any[]> {
  // Transactions are accessed per-supporter, not per-page
  const transactions = await client.get(
    `/supporter/${supporterId}/transactions`
  );

  // Filter locally for recent transactions
  const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  return transactions.filter(
    (tx: any) => new Date(tx.transactionDate) >= oneWeekAgo
  );
}
```

### Bulk Transaction Data via Export Jobs

For large-scale transaction data across many supporters, use export jobs
instead of fetching per-supporter:

```typescript
async function exportTransactions(
  startDate: string,
  endDate: string
): Promise<any[]> {
  // POST /exportjob to create the job
  const job = await client.post('/exportjob', {
    type: 'transaction',
    startDate,
    endDate,
  });

  // Poll until complete
  let status = job;
  while (status.status !== 'complete') {
    await sleep(2000);
    status = await client.get(`/exportjob/${job.id}`);
  }

  // Download results
  return client.get(`/exportjob/${job.id}/download`);
}
```

### Intelligent Supporter Search

Stop early if you have what you need:

```typescript
async function findSupporterTransaction(
  supporterId: string,
  predicate: (tx: any) => boolean
): Promise<any | null> {
  const transactions = await client.get(
    `/supporter/${supporterId}/transactions`
  );

  // Search the supporter's transactions
  const found = transactions.find(predicate);
  return found || null;
}
```

## Step 5: Use Export Jobs for Bulk Aggregation

The EN API has no aggregation/statistics endpoint. For bulk data, use export
jobs instead of fetching transactions one supporter at a time:

```typescript
// ❌ SLOW: Fetch transactions per-supporter and aggregate manually
async function getTotalRevenueSlow(supporterIds: string[]): Promise<number> {
  let total = 0;

  for (const supporterId of supporterIds) {
    const transactions = await client.get(
      `/supporter/${supporterId}/transactions`
    );
    total += transactions.reduce(
      (sum: number, tx: any) => sum + tx.amount, 0
    );
  }

  return total;
}

// ✅ FAST: Use an export job to get all transactions in bulk (fewer API calls)
async function getTotalRevenueFast(
  startDate: string,
  endDate: string
): Promise<number> {
  // Create export job
  const job = await client.post('/exportjob', {
    type: 'transaction',
    startDate,
    endDate,
  });

  // Poll for completion
  let status = job;
  while (status.status !== 'complete') {
    await sleep(2000);
    status = await client.get(`/exportjob/${job.id}`);
  }

  // Download and aggregate locally
  const transactions = await client.get(`/exportjob/${job.id}/download`);
  return transactions.reduce(
    (sum: number, tx: any) => sum + tx.amount, 0
  );
}
```

## Step 6: Implement Local Database

For frequently accessed data, store in local database:

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function syncPagesToDatabase(): Promise<void> {
  const pages = await client.getPages({ type: 'donation', status: 'live' });

  for (const page of pages) {
    await prisma.page.upsert({
      where: { enPageId: page.id },
      update: {
        name: page.name,
        title: page.title,
        campaignStatus: page.campaignStatus,
        modifiedOn: page.modifiedOn,
        updatedAt: new Date(),
      },
      create: {
        enPageId: page.id,
        name: page.name,
        title: page.title,
        campaignStatus: page.campaignStatus,
        pageType: page.type,
        subType: page.subType,
        campaignId: page.campaignId,
      },
    });
  }

  console.log(`Synced ${pages.length} pages to database`);
}

// Run sync once per day via cron job
// Now query local DB instead of hitting EN API constantly
async function getActivePagesLocal() {
  return prisma.page.findMany({
    where: { campaignStatus: 'live' },
  });
}
```

## Step 7: Monitor API Usage

Track your API usage to identify optimization opportunities:

```typescript
class MonitoredENClient extends RateLimitedENClient {
  private metrics = {
    totalRequests: 0,
    cacheHits: 0,
    cacheMisses: 0,
    requestsByEndpoint: new Map<string, number>(),
  };

  async getPage(pageId: string): Promise<ENPage> {
    this.metrics.totalRequests++;
    this.metrics.requestsByEndpoint.set(
      'getPage',
      (this.metrics.requestsByEndpoint.get('getPage') || 0) + 1
    );

    return super.getPage(pageId);
  }

  getMetrics() {
    return {
      ...this.metrics,
      cacheHitRate:
        this.metrics.cacheHits / (this.metrics.cacheHits + this.metrics.cacheMisses),
      requestsByEndpoint: Array.from(this.metrics.requestsByEndpoint.entries()),
    };
  }

  printMetrics() {
    const metrics = this.getMetrics();
    console.log('\n=== EN API Usage Metrics ===');
    console.log(`Total Requests: ${metrics.totalRequests}`);
    console.log(`Cache Hit Rate: ${(metrics.cacheHitRate * 100).toFixed(2)}%`);
    console.log('\nRequests by Endpoint:');
    metrics.requestsByEndpoint.forEach(([endpoint, count]) => {
      console.log(`  ${endpoint}: ${count}`);
    });
  }
}
```

## Step 8: Verify Improvements

Measure before and after:

```typescript
async function benchmarkOptimizations() {
  console.log('=== Before Optimizations ===');
  const start1 = Date.now();
  // Fetch 100 pages without caching
  for (let i = 0; i < 100; i++) {
    await basicClient.getPages({ limit: 10 });
  }
  const time1 = Date.now() - start1;
  console.log(`Time: ${time1}ms`);
  console.log(`API calls: 100`);

  console.log('\n=== After Optimizations ===');
  const start2 = Date.now();
  // Fetch 100 pages with caching
  for (let i = 0; i < 100; i++) {
    await optimizedClient.getPages({ limit: 10 });
  }
  const time2 = Date.now() - start2;
  console.log(`Time: ${time2}ms`);
  console.log(`API calls: ${optimizedClient.getMetrics().cacheMisses}`);
  console.log(`\nImprovement: ${((1 - time2 / time1) * 100).toFixed(2)}% faster`);
  console.log(`API calls reduced by: ${100 - optimizedClient.getMetrics().cacheMisses}%`);
}
```

</process>

<anti_patterns>
Avoid:
- **Aggressive caching of live data** - Recent transactions change constantly
- **No cache invalidation** - Stale data causes bugs
- **Too many parallel requests** - Will trigger rate limits
- **Fetching per-supporter when you need bulk data** - Use export jobs instead
- **No monitoring** - Can't optimize what you don't measure
- **Complex caching without testing** - Cache bugs are hard to debug
- **Caching sensitive data** - Be careful with PII
</anti_patterns>

<success_criteria>
Successfully optimized when:
- ✓ API calls reduced by at least 50%
- ✓ Caching implemented for static data (pages, past export results)
- ✓ Rate limiting prevents 429 errors
- ✓ Monitoring shows cache hit rates > 70%
- ✓ Response times improved measurably
- ✓ Batch processing for bulk operations
- ✓ No stale data bugs
- ✓ Stays within EN rate limits comfortably
</success_criteria>
