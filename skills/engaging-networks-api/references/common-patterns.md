<overview>
Common integration patterns and use cases for Engaging Networks API. Real-world examples using verified endpoints for supporter management, transactions, export jobs, page processing, and reporting.

**These patterns are for your application code** — build and test them against a sandbox or test EN account, then deploy as server-side applications. Patterns that retrieve supporter data (daily sync, donor roll, supporter CRUD, polling) pull PII. When executed through an AI agent, that PII is processed by the model provider's infrastructure. Use the agent to write and refine these patterns, then run them independently.
</overview>

<daily_sync_pattern>
## Daily Data Sync

Query recently modified supporters and fetch their transactions to update local database.

```typescript
import { ENClient } from './lib/en-client';
import { db } from './lib/database';

async function dailySync() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  console.log('Syncing recently modified supporters...');

  // Query supporters modified in the last day
  let offset = 0;
  const rows = 100;
  let totalSynced = 0;

  while (true) {
    const supporters = await client.get('/supporter/query', {
      params: {
        type: 'latestModified',
        daysBack: 1,
        rows,
        offset,
      },
    });

    if (!supporters.length) break;

    console.log(`Fetched ${supporters.length} modified supporters (offset: ${offset})`);

    for (const supporter of supporters) {
      // Upsert supporter record
      await db.supporter.upsert({
        where: { enSupporterId: supporter.supporterId },
        update: { ...supporter, syncedAt: new Date() },
        create: { ...supporter, syncedAt: new Date() },
      });

      // Fetch transactions for this supporter
      const transactions = await client.get(
        `/supporter/${supporter.supporterId}/transactions`
      );

      for (const txn of transactions) {
        await db.transaction.upsert({
          where: { enTransactionId: txn.transactionId },
          update: { ...txn, supporterId: supporter.supporterId, syncedAt: new Date() },
          create: { ...txn, supporterId: supporter.supporterId, syncedAt: new Date() },
        });
      }

      totalSynced++;
    }

    if (supporters.length < rows) break;
    offset += rows;
  }

  console.log(`Daily sync complete: ${totalSynced} supporters updated`);
}

// Run via cron: 0 2 * * * (daily at 2 AM)
dailySync().catch(console.error);
```

**Schedule:** Daily at 2 AM (off-peak hours)
**API Calls:** ~N (supporter query pages) + N (transaction fetches per supporter)
**Note:** For large data volumes, consider using Export Jobs instead (see export_job_workflow).
</daily_sync_pattern>

<campaign_widget>
## Campaign Progress Widget

Display campaign metrics on a website using locally tracked data. The EN API has no statistics/aggregation endpoint, so metrics must be computed from synced transaction data stored in a local database.

```typescript
import express from 'express';
import { db } from './lib/database';

const app = express();

// Serve campaign stats from local database (populated by daily sync or export jobs)
app.get('/api/campaign/:pageId/stats', async (req, res) => {
  const { pageId } = req.params;

  try {
    const today = new Date();
    const startOfMonth = new Date(today.getFullYear(), today.getMonth(), 1);

    // Query local database for aggregated metrics
    const metrics = await db.transaction.aggregate({
      where: {
        pageId,
        createdAt: { gte: startOfMonth },
      },
      _sum: { amount: true },
      _count: { id: true },
      _avg: { amount: true },
    });

    const goalAmount = (await db.campaign.findUnique({
      where: { pageId },
    }))?.goalAmount || 100000;

    const totalRevenue = metrics._sum.amount || 0;
    const progress = (totalRevenue / goalAmount) * 100;

    const response = {
      totalRevenue,
      donorCount: metrics._count.id || 0,
      averageDonation: metrics._avg.amount || 0,
      goalAmount,
      progress: Math.min(progress, 100),
      daysRemaining:
        new Date(today.getFullYear(), today.getMonth() + 1, 0).getDate() -
        today.getDate(),
    };

    res.json(response);
  } catch (error) {
    console.error('Error fetching campaign stats:', error);
    res.status(500).json({ error: 'Failed to fetch campaign stats' });
  }
});

app.listen(3000);
```

**Frontend Integration:**
```html
<div id="campaign-widget">
  <h3>Emergency Relief Campaign</h3>
  <div class="progress-bar">
    <div class="progress" style="width: 67%"></div>
  </div>
  <p><strong>$67,500</strong> raised of $100,000 goal</p>
  <p>450 donors | $150 average donation</p>
  <p>15 days remaining</p>
</div>

<script>
async function updateWidget() {
  const response = await fetch('/api/campaign/12345/stats');
  const data = await response.json();

  document.querySelector('.progress').style.width = `${data.progress}%`;
  document.querySelector('strong').textContent =
    `$${data.totalRevenue.toLocaleString()}`;
  // Update other fields...
}

updateWidget();
setInterval(updateWidget, 60000); // Update every minute
</script>
```

**Data Source:** Local database populated by daily sync or export jobs.
**API Calls:** Zero at widget serve time. All EN API calls happen during background sync.
</campaign_widget>

<donor_roll_generation>
## Generate Donor Roll

Query supporters and fetch per-supporter transactions to build a donor list for a campaign period.

```typescript
import * as fs from 'fs/promises';

async function generateDonorRoll(
  startDate: string,
  endDate: string
): Promise<void> {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  // Step 1: Query all supporters (paginated)
  const allSupporters: any[] = [];
  let offset = 0;
  const rows = 100;

  console.log('Querying supporters...');

  while (true) {
    const supporters = await client.get('/supporter/query', {
      params: { type: 'all', rows, offset },
    });

    if (!supporters.length) break;

    allSupporters.push(...supporters);
    console.log(`Fetched ${allSupporters.length} supporters so far...`);

    if (supporters.length < rows) break;
    offset += rows;
  }

  console.log(`Total supporters: ${allSupporters.length}`);

  // Step 2: Fetch transactions for each supporter and filter by date range
  const donorMap = new Map<string, any>();
  const start = new Date(startDate).getTime();
  const end = new Date(endDate).getTime();

  for (const supporter of allSupporters) {
    const transactions = await client.get(
      `/supporter/${supporter.supporterId}/transactions`
    );

    // Filter transactions within the date range
    const matched = transactions.filter((txn: any) => {
      const txnDate = new Date(txn.transactionDate).getTime();
      return txnDate >= start && txnDate <= end;
    });

    if (matched.length === 0) continue;

    const email = supporter.emailAddress;
    const totalDonated = matched.reduce((sum: number, txn: any) => sum + txn.amount, 0);

    donorMap.set(email, {
      firstName: supporter.firstName,
      lastName: supporter.lastName,
      email,
      totalDonated,
      donationCount: matched.length,
      firstDonationDate: matched[0].transactionDate,
    });
  }

  // Sort by total donated (descending)
  const donors = Array.from(donorMap.values()).sort(
    (a, b) => b.totalDonated - a.totalDonated
  );

  // Export to CSV
  const csv = [
    'First Name,Last Name,Email,Total Donated,# Donations,First Donation',
    ...donors.map(d =>
      `${d.firstName},${d.lastName},${d.email},${d.totalDonated},${d.donationCount},${d.firstDonationDate}`
    ),
  ].join('\n');

  await fs.writeFile('donor-roll.csv', csv);

  console.log(`Generated donor roll: ${donors.length} unique donors`);
  console.log(`Total donations: $${donors.reduce((sum, d) => sum + d.totalDonated, 0)}`);
}

// Usage
generateDonorRoll('2024-01-01', '2024-12-31');
```

**API Calls:** ceil(totalSupporters / 100) for query + 1 per supporter for transactions
**Note:** For large supporter bases, prefer using an Export Job (see export_job_workflow) to avoid excessive API calls.
</donor_roll_generation>

<multi_page_reporting>
## Multi-Page Dashboard

Build a dashboard comparing campaign performance using export jobs for bulk data extraction and local metrics computation.

```typescript
async function generateCampaignReport(): Promise<void> {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  // Step 1: Initiate an export job for transaction data
  const exportJob = await client.post('/exportjob', {
    fileType: 'csv',
    queryName: 'all_transactions_last_30_days', // Pre-configured query in EN
  });

  const jobId = exportJob.id;
  console.log(`Export job created: ${jobId}`);

  // Step 2: Poll until the export is ready
  let status = 'pending';
  while (status !== 'completed') {
    await new Promise(resolve => setTimeout(resolve, 5000));

    const jobStatus = await client.get(`/exportjob/${jobId}`);
    status = jobStatus.status;
    console.log(`Export job status: ${status}`);

    if (status === 'failed') {
      throw new Error(`Export job ${jobId} failed`);
    }
  }

  // Step 3: Download the export
  const csvData = await client.get(`/exportjob/${jobId}/download`);

  // Step 4: Parse and aggregate by page
  const pageMetrics = new Map<string, {
    name: string;
    revenue: number;
    donors: Set<string>;
    donationCount: number;
  }>();

  for (const row of csvData) {
    const pageId = row.pageId;
    if (!pageMetrics.has(pageId)) {
      pageMetrics.set(pageId, {
        name: row.pageName || pageId,
        revenue: 0,
        donors: new Set(),
        donationCount: 0,
      });
    }
    const metrics = pageMetrics.get(pageId)!;
    metrics.revenue += row.amount;
    metrics.donors.add(row.supporterEmail);
    metrics.donationCount++;
  }

  // Sort by revenue
  const results = Array.from(pageMetrics.values())
    .map(m => ({
      name: m.name,
      revenue: m.revenue,
      donors: m.donors.size,
      avgDonation: m.donationCount > 0 ? m.revenue / m.donationCount : 0,
    }))
    .sort((a, b) => b.revenue - a.revenue);

  // Generate report
  console.log('\n=== Campaign Performance Report (Last 30 Days) ===\n');
  console.log('Rank | Campaign                                 | Revenue    | Donors | Avg Gift');
  console.log('-----|------------------------------------------|------------|--------|----------');

  results.forEach((r, i) => {
    console.log(
      `${(i + 1).toString().padEnd(5)}| ${r.name.padEnd(40)} | ` +
      `$${r.revenue.toLocaleString().padEnd(10)} | ${r.donors.toString().padEnd(6)} | ` +
      `$${r.avgDonation.toFixed(2)}`
    );
  });

  const totals = {
    revenue: results.reduce((sum, r) => sum + r.revenue, 0),
    donors: results.reduce((sum, r) => sum + r.donors, 0),
  };

  console.log('-----|------------------------------------------|------------|--------|----------');
  console.log(
    `TOTAL|${' '.repeat(40)} | $${totals.revenue.toLocaleString().padEnd(10)} | ${totals.donors}`
  );
}

generateCampaignReport();
```

**API Calls:** 1 (create export) + N (poll status) + 1 (download) -- significantly fewer than per-page fetching
</multi_page_reporting>

<webhook_alternative>
## Polling Pattern (EN has no webhooks)

Since EN does not provide webhooks, poll for recently modified supporters for near-real-time updates.

```typescript
import nodeCron from 'node-cron';

// Track last poll time
let lastPollTime: string = new Date(Date.now() - 3600000).toISOString();

async function pollForUpdates() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  const now = new Date().toISOString();

  // Query supporters modified since last poll
  let offset = 0;
  const rows = 100;
  let newCount = 0;

  while (true) {
    const supporters = await client.get('/supporter/query', {
      params: {
        type: 'latestModified',
        daysBack: 1, // Minimum granularity is 1 day; filter further client-side
        rows,
        offset,
      },
    });

    if (!supporters.length) break;

    for (const supporter of supporters) {
      // Check if this supporter was modified after our last poll
      if (supporter.modifiedDate && supporter.modifiedDate > lastPollTime) {
        // Fetch their transactions for context
        const transactions = await client.get(
          `/supporter/${supporter.supporterId}/transactions`
        );

        // Process updated supporter and their transactions
        await processUpdatedSupporter(supporter, transactions);
        newCount++;
      }
    }

    if (supporters.length < rows) break;
    offset += rows;
  }

  if (newCount > 0) {
    console.log(`Processed ${newCount} updated supporters`);
  }

  lastPollTime = now;
}

async function processUpdatedSupporter(supporter: any, transactions: any[]) {
  console.log(`Updated supporter: ${supporter.emailAddress}`);

  // Send to CRM
  // await crm.upsertContact(supporter);

  // Check for new transactions
  // await syncTransactions(supporter.supporterId, transactions);

  // Update analytics
  // await analytics.track('supporter_updated', { id: supporter.supporterId });
}

// Poll every 5 minutes
nodeCron.schedule('*/5 * * * *', pollForUpdates);
```

**Trade-offs:**
- Near-real-time (5 min latency)
- Catches all modified supporters and their transactions
- More API calls (every 5 min)
- Not true real-time
- daysBack minimum is 1, so client-side filtering is needed for sub-day precision

**API Calls:** ~N (supporter query pages) + M (transaction fetches) per poll interval
</webhook_alternative>

<supporter_crud>
## Supporter CRUD Operations

Create, read, update, and delete supporter records.

```typescript
async function supporterCrudExamples() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  // CREATE a new supporter
  const newSupporter = await client.post('/supporter', {
    firstName: 'Jane',
    lastName: 'Doe',
    emailAddress: 'jane.doe@example.com',
    address1: '123 Main St',
    city: 'Springfield',
    region: 'IL',
    postcode: '62704',
    country: 'US',
    customField1: 'value1',
  });
  console.log('Created supporter:', newSupporter.supporterId);

  // READ a supporter by ID
  const supporter = await client.get(`/supporter/${newSupporter.supporterId}`);
  console.log('Supporter:', supporter.firstName, supporter.lastName);

  // READ supporter with specific fields
  const partialSupporter = await client.get(
    `/supporter/${newSupporter.supporterId}`,
    { params: { fields: 'firstName,lastName,emailAddress' } }
  );

  // UPDATE a supporter
  await client.put(`/supporter/${newSupporter.supporterId}`, {
    firstName: 'Janet',
    customField1: 'updated_value',
  });
  console.log('Supporter updated');

  // DELETE a supporter
  await client.delete(`/supporter/${newSupporter.supporterId}`);
  console.log('Supporter deleted');

  // QUERY supporters
  const recentSupporters = await client.get('/supporter/query', {
    params: {
      type: 'latestModified',
      daysBack: 7,
      rows: 50,
      offset: 0,
    },
  });
  console.log(`Found ${recentSupporters.length} recently modified supporters`);

  // Query all supporters (paginated)
  const allSupporters = await client.get('/supporter/query', {
    params: { type: 'all', rows: 100, offset: 0 },
  });
}

supporterCrudExamples();
```

**Endpoints:**
- `POST /supporter` -- create
- `GET /supporter/{supporterId}` -- read
- `PUT /supporter/{supporterId}` -- update
- `DELETE /supporter/{supporterId}` -- delete
- `GET /supporter/query?type=...&rows=...&offset=...` -- query/search
</supporter_crud>

<origin_source_management>
## Origin Source Management

Assign, update, and remove acquisition channel tracking (origin sources) on supporters.

```typescript
async function originSourceExamples() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  const supporterId = 12345;

  // GET current origin sources for a supporter
  const origins = await client.get(`/supporter/${supporterId}/originSource`);
  console.log('Current origin sources:', origins);

  // ADD an origin source to a supporter
  await client.post(`/supporter/${supporterId}/originSource`, {
    source: 'google_ads',
    medium: 'cpc',
    campaign: 'spring_2024_appeal',
    content: 'banner_v2',
  });
  console.log('Origin source added');

  // UPDATE an origin source
  await client.put(`/supporter/${supporterId}/originSource/${origins[0].id}`, {
    campaign: 'spring_2024_appeal_updated',
  });
  console.log('Origin source updated');

  // REMOVE an origin source
  await client.delete(
    `/supporter/${supporterId}/originSource/${origins[0].id}`
  );
  console.log('Origin source removed');
}

originSourceExamples();
```

**Endpoints:**
- `GET /supporter/{supporterId}/originSource` -- list origin sources
- `POST /supporter/{supporterId}/originSource` -- add origin source
- `PUT /supporter/{supporterId}/originSource/{id}` -- update origin source
- `DELETE /supporter/{supporterId}/originSource/{id}` -- remove origin source
</origin_source_management>

<export_job_workflow>
## Export Job Workflow

Initiate, poll, and download bulk data exports. This is the recommended approach for large data sets instead of paginating through individual API calls.

```typescript
import * as fs from 'fs/promises';

async function runExportJob(queryName: string, fileType: 'csv' | 'xlsx' = 'csv') {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  // Step 1: Initiate the export job
  // queryName must match a saved query configured in Engaging Networks
  const job = await client.post('/exportjob', {
    fileType,
    queryName,
  });

  const jobId = job.id;
  console.log(`Export job initiated: ${jobId}`);

  // Step 2: Poll for completion
  const maxAttempts = 60; // 5 minutes at 5-second intervals
  let attempts = 0;

  while (attempts < maxAttempts) {
    await new Promise(resolve => setTimeout(resolve, 5000));
    attempts++;

    const status = await client.get(`/exportjob/${jobId}`);
    console.log(`Poll ${attempts}: status = ${status.status}`);

    if (status.status === 'completed') {
      // Step 3: Download the file
      const fileData = await client.get(`/exportjob/${jobId}/download`, {
        responseType: 'arraybuffer',
      });

      const filename = `export-${jobId}.${fileType}`;
      await fs.writeFile(filename, Buffer.from(fileData));
      console.log(`Export downloaded: ${filename}`);
      return filename;
    }

    if (status.status === 'failed') {
      throw new Error(`Export job ${jobId} failed: ${status.errorMessage || 'unknown error'}`);
    }
  }

  throw new Error(`Export job ${jobId} timed out after ${maxAttempts} attempts`);
}

// Usage: run a pre-configured query
runExportJob('all_donors_2024').catch(console.error);
```

**Endpoints:**
- `POST /exportjob` -- create export job (body: `{ fileType, queryName }`)
- `GET /exportjob/{id}` -- check job status
- `GET /exportjob/{id}/download` -- download completed export

**Notes:**
- The `queryName` must reference a saved query previously created in the Engaging Networks admin.
- Export jobs run asynchronously; poll at reasonable intervals (5-10 seconds).
- Prefer export jobs over paginated API queries when extracting more than a few hundred records.
</export_job_workflow>

<marketing_automation>
## Marketing Automation Management

List automations, view stats, and add supporters to marketing automation workflows in bulk.

```typescript
async function marketingAutomationExamples() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  // LIST all marketing automations
  const automations = await client.get('/marketingAutomation');
  console.log(`Found ${automations.length} automations`);

  for (const auto of automations) {
    console.log(`- ${auto.name} (ID: ${auto.id}, status: ${auto.status})`);
  }

  // GET statistics for a specific automation
  const autoId = automations[0].id;
  const stats = await client.get(`/marketingAutomation/${autoId}/stats`);
  console.log(`Automation stats:`, stats);

  // ADD supporters to an automation in bulk
  const supporterIds = [12345, 12346, 12347, 12348];

  await client.post(`/marketingAutomation/${autoId}/supporters`, {
    supporterIds,
  });
  console.log(`Added ${supporterIds.length} supporters to automation ${autoId}`);
}

// Practical example: enroll new donors into a welcome series
async function enrollNewDonorsInWelcomeSeries() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  const welcomeSeriesId = 100; // ID of the welcome series automation

  // Query supporters added in the last day
  const newSupporters = await client.get('/supporter/query', {
    params: {
      type: 'latestModified',
      daysBack: 1,
      rows: 100,
      offset: 0,
    },
  });

  if (newSupporters.length > 0) {
    const ids = newSupporters.map((s: any) => s.supporterId);

    await client.post(`/marketingAutomation/${welcomeSeriesId}/supporters`, {
      supporterIds: ids,
    });

    console.log(`Enrolled ${ids.length} new supporters in welcome series`);
  }
}

enrollNewDonorsInWelcomeSeries();
```

**Endpoints:**
- `GET /marketingAutomation` -- list all automations
- `GET /marketingAutomation/{id}/stats` -- get automation statistics
- `POST /marketingAutomation/{id}/supporters` -- add supporters to automation in bulk
</marketing_automation>

<page_processing>
## Page Processing (Donations, Petitions, Events)

Submit supporter actions through EN pages using the page process endpoint. This is how you programmatically submit donations, sign petitions, or register for events.

```typescript
async function pageProcessingExamples() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
  });

  // PROCESS a donation
  const donationPageId = 12345;
  const donationResult = await client.post(
    `/page/${donationPageId}/process`,
    {
      supporter: {
        firstName: 'John',
        lastName: 'Smith',
        emailAddress: 'john.smith@example.com',
        address1: '456 Oak Ave',
        city: 'Portland',
        region: 'OR',
        postcode: '97201',
        country: 'US',
      },
      transaction: {
        donationAmt: 50.00,
        recurrFreq: 'MONTHLY', // or 'ONE_TIME'
        paymentType: 'VI', // Visa
        ccnumber: '4111111111111111',
        ccexpire: '12/2026',
        ccvv: '123',
      },
    }
  );
  console.log('Donation processed:', donationResult);

  // PROCESS a petition signature
  const petitionPageId = 12346;
  const petitionResult = await client.post(
    `/page/${petitionPageId}/process`,
    {
      supporter: {
        firstName: 'Jane',
        lastName: 'Doe',
        emailAddress: 'jane.doe@example.com',
        postcode: '10001',
        country: 'US',
      },
    }
  );
  console.log('Petition signed:', petitionResult);

  // PROCESS an event registration
  const eventPageId = 12347;
  const eventResult = await client.post(
    `/page/${eventPageId}/process`,
    {
      supporter: {
        firstName: 'Bob',
        lastName: 'Wilson',
        emailAddress: 'bob.wilson@example.com',
      },
      event: {
        ticketType: 'general_admission',
        quantity: 2,
      },
    }
  );
  console.log('Event registration:', eventResult);
}

pageProcessingExamples();
```

**Endpoint:**
- `POST /page/{pageId}/process` -- submit a supporter action through a page

**Notes:**
- The page type (donation, petition, event, etc.) determines which fields are required in the request body.
- For donations, payment details are required alongside supporter information.
- The `supporter` object will create or update the supporter record automatically.
- Test with sandbox/test pages before processing against production pages.
</page_processing>

<recurring_transaction_management>
## Recurring Transaction Management

List, migrate, update, and cancel recurring donations.

```typescript
async function recurringTransactionExamples() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'ca',
  });

  const supporterId = 12345;

  // LIST recurring transactions for a supporter (dedicated endpoint)
  const recurring = await client.get(
    `/supporter/${supporterId}/transactions/recurring`
  );

  // Filter to active only
  const active = recurring.filter((txn: any) => txn.status === 'ACTIVE');
  console.log(`Found ${active.length} active recurring transactions`);

  for (const txn of active) {
    console.log(
      `- ${txn.transactionId}: $${txn.amount} ${txn.frequency} (next: ${txn.nextPaymentDate})`
    );
  }

  // UPDATE a recurring transaction (e.g., change amount)
  // Uses the gateway transactionId, not the internal id
  const gatewayTxnId = active[0]?.transactionId;
  if (gatewayTxnId) {
    await client.put(
      `/supporter/${supporterId}/transactions/recurring/${encodeURIComponent(gatewayTxnId)}`,
      { amount: 75.00 }
    );
    console.log(`Updated recurring transaction ${gatewayTxnId} to $75.00`);
  }

  // PAUSE a recurring transaction
  if (gatewayTxnId) {
    await client.put(
      `/supporter/${supporterId}/transactions/recurring/${encodeURIComponent(gatewayTxnId)}`,
      { status: 'PAUSED' }
    );
    console.log(`Paused recurring transaction ${gatewayTxnId}`);
  }

  // CANCEL a recurring transaction
  if (gatewayTxnId) {
    await client.put(
      `/supporter/${supporterId}/transactions/recurring/${encodeURIComponent(gatewayTxnId)}`,
      { status: 'CANCELED', reason: 'Requested by supporter' }
    );
    console.log(`Cancelled recurring transaction ${gatewayTxnId}`);
  }
}

recurringTransactionExamples();
```

**Endpoints:**
- `GET /supporter/{supporterId}/transactions/recurring` -- list recurring transactions
- `PUT /supporter/{supporterId}/transactions/recurring/{transactionId}` -- update (amount, frequency, status, paymentDay, nextPaymentDate)

**Updatable fields:** `amount`, `frequency`, `paymentDay`, `endDate`, `status`, `reason`, `nextPaymentDate`
**Status values:** `ACTIVE`, `PAUSED`, `SUSPENDED`, `CANCELED`

**Notes:**
- Use the dedicated `/transactions/recurring` endpoint (not the general `/transactions` list).
- The `transactionId` in the path is the gateway-defined ID (e.g., `ND272532241__A23028626`), not the internal `id`.
- URL-encode the transactionId since gateway IDs may contain special characters.
</recurring_transaction_management>

<best_practices_summary>
## Integration Best Practices

1. **Use Export Jobs for Bulk Data**
   - Prefer export jobs over paginated queries for large data sets
   - Export jobs reduce total API calls significantly
   - Pre-configure saved queries in EN admin for reuse

2. **Cache and Store Locally**
   - There is no statistics/aggregation endpoint; compute metrics locally
   - Sync supporter and transaction data to a local database
   - Serve dashboards and widgets from local data, not live API calls

3. **Batch Operations**
   - Process supporters in parallel (5-10 at a time)
   - Add delays between batches to avoid rate limits
   - Use Promise.all() for independent requests within a batch

4. **Schedule Wisely**
   - Bulk operations (export jobs, full syncs): Off-peak hours (2-4 AM)
   - Incremental polling: Every 5-15 minutes using latestModified query
   - Daily syncs: Once per day with daysBack=1

5. **Error Handling**
   - Retry 429 (rate limit) and 5xx errors with exponential backoff
   - Log all failures with request details
   - Alert on repeated failures
   - Graceful degradation (serve cached/local data when API is unavailable)

6. **Supporter Query Pagination**
   - Always paginate with `rows` and `offset` parameters
   - Default to rows=100 for efficient batching
   - Continue until response length is less than `rows`

7. **Transaction Access**
   - Transactions are always per-supporter: GET /supporter/{id}/transactions
   - There is no per-page transaction listing endpoint
   - To get all transactions for a page, use export jobs

8. **Monitoring**
   - Track API call count per hour
   - Monitor rate limit (429) responses
   - Alert when approaching limits
   - Dashboard for sync health and data freshness
</best_practices_summary>
