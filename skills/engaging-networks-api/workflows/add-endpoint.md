# Workflow: Add EN API Endpoint

<required_reading>
**Read these reference files NOW:**
1. references/endpoints.md
2. references/typescript-client.md (if TypeScript) OR references/python-client.md (if Python)
</required_reading>

<process>
## Step 1: Identify the Endpoint

Ask the user which endpoint they need:

**Page Services:**
- GET /page - List pages by type
- GET /page/{id} - View page details
- POST /page/{id}/process - Process page submission (donations, petitions, events)
- GET /page/{id}/survey - Get survey responses

**Page Components:**
- GET /page/component/{type} - List components (codeblock, formblock, template, textblock, thankemail, upsell, widget)
- GET /page/component/{type}/{id} - View component details

**Supporter Services:**
- GET /supporter?email=... - Look up supporter by email
- GET /supporter/{id} - View supporter by ID
- POST /supporter - Create or update supporter
- PUT /supporter/{id} - Update supporter by ID
- DELETE /supporter/{id} - Delete supporter
- GET /supporter/query - Query supporters (latestCreated, latestModified, suppressed, profile, search)
- GET /supporter/fields - List available fields
- PUT /supporter/bulk - Bulk update suppression status

**Transactions:**
- GET /supporter/{id}/transactions - List supporter transactions
- GET /supporter/{id}/transactions/{txnId} - View transaction details
- GET /supporter/{id}/transactions/recurring - List recurring transactions
- POST /supporter/{id}/transactions/recurring - Migrate recurring transaction
- PUT /supporter/{id}/transactions/recurring/{txnId} - Update recurring transaction

**Origin Source:**
- GET/POST/PUT/DELETE /supporter/{id}/originsource

**Export Jobs:**
- POST /exportjob - Initiate export
- GET /exportjob/{id} - Check status
- GET /exportjob/{id}/download - Download results

**Marketing Automations:**
- GET /ma - List automations
- GET /ma/{id} - View details
- GET /ma/{id}/stats - Get statistics
- POST /ma/{id}/supporter/bulk - Add supporters

**Account:**
- GET /account/auditlog - View audit log

**What data do you need from this endpoint?**

## Step 2: Review Endpoint Documentation

Check the EN API reference for:
- Path parameters (e.g., `{id}`)
- Query parameters (e.g., `?startDate=2024-01-01`)
- Request body (for POST/PUT)
- Response structure
- Rate limit considerations

## Step 3: Add Type Definitions

### TypeScript

Add types to `src/lib/en-client.ts`:

**Example: Adding the export job endpoint**

```typescript
// Add interface for response
export interface ENExportJob {
  id: string;
  status: string;
  type: string;
  createdDate: string;
  completedDate?: string;
  totalRows?: number;
  errorMessage?: string;
}

export interface ENExportJobRequest {
  type: string;
  startDate?: string;
  endDate?: string;
  dateField?: string;
  fields?: string[];
}

// Add methods to ENClient class
async initiateExport(params: ENExportJobRequest): Promise<ENExportJob> {
  return this.fetchWithRetry(async () => {
    const response = await this.client.post<ENExportJob>(
      '/exportjob',
      params
    );
    return response.data;
  });
}

async getExportStatus(jobId: string): Promise<ENExportJob> {
  return this.fetchWithRetry(async () => {
    const response = await this.client.get<ENExportJob>(
      `/exportjob/${jobId}`
    );
    return response.data;
  });
}

async downloadExport(jobId: string): Promise<any[]> {
  return this.fetchWithRetry(async () => {
    const response = await this.client.get(
      `/exportjob/${jobId}/download`
    );
    return response.data;
  });
}
```

### Python

Add method to `src/en_client.py`:

```python
def initiate_export(
    self,
    export_type: str,
    start_date: Optional[str] = None,
    end_date: Optional[str] = None,
    date_field: Optional[str] = None,
    fields: Optional[List[str]] = None,
) -> Dict[str, Any]:
    """Initiate an export job via POST /exportjob"""
    payload: Dict[str, Any] = {'type': export_type}
    if start_date:
        payload['startDate'] = start_date
    if end_date:
        payload['endDate'] = end_date
    if date_field:
        payload['dateField'] = date_field
    if fields:
        payload['fields'] = fields

    return self._request('POST', '/exportjob', json=payload)


def get_export_status(self, job_id: str) -> Dict[str, Any]:
    """Check export job status via GET /exportjob/{id}"""
    return self._request('GET', f'/exportjob/{job_id}')


def download_export(self, job_id: str) -> List[Dict[str, Any]]:
    """Download export results via GET /exportjob/{id}/download"""
    return self._request('GET', f'/exportjob/{job_id}/download')
```

## Step 4: Test the New Endpoint

Create a test script:

### TypeScript

```typescript
// test-new-endpoint.ts
import { ENClient } from './lib/en-client';
import * as dotenv from 'dotenv';

dotenv.config();

async function testExportJob() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: 'us',
    debug: true,
  });

  try {
    console.log('Testing export job...');

    // Initiate an export
    const job = await client.initiateExport({
      type: 'supporter',
      startDate: '2024-01-01',
      endDate: '2024-12-31',
    });

    console.log(`Export job created: ${job.id}, status: ${job.status}`);

    // Poll for completion
    let status = job;
    while (status.status !== 'completed' && status.status !== 'failed') {
      await new Promise(resolve => setTimeout(resolve, 5000));
      status = await client.getExportStatus(job.id);
      console.log(`Status: ${status.status}`);
    }

    if (status.status === 'completed') {
      const results = await client.downloadExport(job.id);
      console.log(`Downloaded ${results.length} rows`);
    } else {
      console.error(`Export failed: ${status.errorMessage}`);
    }
  } catch (error) {
    console.error('Error:', error);
  }
}

testExportJob();
```

### Python

```python
# test_new_endpoint.py
from src.en_client import ENClient, ENClientConfig
import os
import time
from dotenv import load_dotenv

load_dotenv()

def test_export_job():
    client = ENClient(ENClientConfig(
        api_token=os.environ['EN_API_TOKEN'],
        region='us',
    ))

    try:
        print('Testing export job...')

        # Initiate an export
        job = client.initiate_export(
            export_type='supporter',
            start_date='2024-01-01',
            end_date='2024-12-31',
        )

        print(f"Export job created: {job['id']}, status: {job['status']}")

        # Poll for completion
        status = job
        while status['status'] not in ('completed', 'failed'):
            time.sleep(5)
            status = client.get_export_status(job['id'])
            print(f"Status: {status['status']}")

        if status['status'] == 'completed':
            results = client.download_export(job['id'])
            print(f"Downloaded {len(results)} rows")
        else:
            print(f"Export failed: {status.get('errorMessage')}")

    except Exception as e:
        print(f'Error: {e}')

if __name__ == '__main__':
    test_export_job()
```

Run the test:

```bash
# TypeScript
npx ts-node test-new-endpoint.ts

# Python
python test_new_endpoint.py
```

## Step 5: Handle Edge Cases

Consider edge cases for the endpoint:

**Empty results:**
```typescript
const results = await client.downloadExport(jobId);

if (results.length === 0) {
  console.log('Export returned no rows');
} else {
  console.log(`Downloaded ${results.length} rows`);
}
```

**Pagination:**
```typescript
async function getAllTransactions(supporterId: string): Promise<any[]> {
  const allTransactions: any[] = [];
  let offset = 0;
  const limit = 100;

  while (true) {
    const response = await client.getSupporterTransactions(supporterId, { limit, offset });
    allTransactions.push(...response);

    if (response.length < limit) {
      break; // No more pages
    }

    offset += limit;
  }

  return allTransactions;
}
```

**Error handling:**
```typescript
try {
  const job = await client.initiateExport({ type: 'supporter' });
  // Process result
} catch (error) {
  if (axios.isAxiosError(error)) {
    if (error.response?.status === 400) {
      console.error('Invalid export parameters');
    } else if (error.response?.status === 403) {
      console.error('No permission for this operation');
    } else if (error.response?.status === 404) {
      console.error('Export job not found');
    } else {
      console.error('API error:', error.message);
    }
  }
}
```

## Step 6: Update Documentation

Add the new endpoint to your integration docs:

```typescript
/**
 * EN Client Documentation
 *
 * Available methods:
 * - getPages(params?) - List pages (GET /page)
 * - getPage(id) - Get page details (GET /page/{id})
 * - processPage(id, data) - Submit page data (POST /page/{id}/process)
 * - getSupporter(id) - Get supporter (GET /supporter/{id})
 * - lookupSupporter(email) - Look up by email (GET /supporter?email=...)
 * - createSupporter(data) - Create/update supporter (POST /supporter)
 * - getSupporterTransactions(id) - List transactions (GET /supporter/{id}/transactions)
 * - initiateExport(params) - Start export job (POST /exportjob) [NEW]
 * - getExportStatus(id) - Check export status (GET /exportjob/{id}) [NEW]
 * - downloadExport(id) - Download export (GET /exportjob/{id}/download) [NEW]
 *
 * Example:
 * ```typescript
 * const job = await client.initiateExport({
 *   type: 'supporter',
 *   startDate: '2024-01-01',
 * });
 * ```
 */
```

## Step 7: Add Integration Test

Add to your test suite:

```typescript
// tests/en-client.test.ts
describe('ENClient', () => {
  // ... existing tests ...

  describe('initiateExport', () => {
    it('should create an export job', async () => {
      const job = await client.initiateExport({
        type: 'supporter',
        startDate: '2024-01-01',
      });

      expect(job).toHaveProperty('id');
      expect(job).toHaveProperty('status');
      expect(typeof job.id).toBe('string');
    });

    it('should check export status', async () => {
      const job = await client.initiateExport({ type: 'supporter' });
      const status = await client.getExportStatus(job.id);

      expect(status).toHaveProperty('id', job.id);
      expect(status).toHaveProperty('status');
    });

    it('should handle invalid export type', async () => {
      await expect(
        client.initiateExport({ type: 'invalid_type' })
      ).rejects.toThrow();
    });
  });

  describe('getSupporterTransactions', () => {
    it('should list transactions for a supporter', async () => {
      const transactions = await client.getSupporterTransactions('12345');

      expect(Array.isArray(transactions)).toBe(true);
    });

    it('should handle supporter with no transactions', async () => {
      const transactions = await client.getSupporterTransactions('99999');

      expect(transactions).toHaveLength(0);
    });
  });
});
```

</process>

<anti_patterns>
Avoid:
- **No type safety** - Always add TypeScript types or Python type hints
- **Skipping tests** - New endpoints must be tested
- **Ignoring pagination** - Most EN endpoints are paginated
- **Not handling empty results** - Check for empty arrays/lists
- **Forgetting error cases** - Add specific error handling
- **No documentation** - Comment your new methods
- **Hardcoding parameters** - Make them configurable
</anti_patterns>

<write_safety>
## Write Endpoint Safety

If the new endpoint is a write operation (POST/PUT/DELETE that modifies data):

1. **Add `dryRun` option** to the method signature (`options?: { dryRun?: boolean }`)
2. **When dryRun is true**, log the method, URL, and request body without executing
3. **In test scripts**, demonstrate dryRun first before any real execution
4. **For DESTRUCTIVE endpoints** (DELETE), add a console warning about irreversibility

Check the endpoint's risk classification in `references/endpoints.md` -- endpoints tagged WRITE or DESTRUCTIVE require this pattern.
</write_safety>

<success_criteria>
Successfully added endpoint when:
- Type definitions added for request/response
- Method added to client class with proper signature
- Uses existing `fetchWithRetry` for error handling
- Write methods include `dryRun` option for safe previewing
- Test script created and runs successfully
- Edge cases handled (empty results, pagination, errors)
- Documentation updated
- Integration tests added
- Code follows existing patterns in the client
</success_criteria>
