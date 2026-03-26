<overview>
TypeScript implementation patterns for Engaging Networks API client with full type safety, error handling, and best practices.
</overview>

<project_setup>
## Project Setup

```bash
mkdir en-api-client
cd en-api-client
npm init -y

# Core dependencies
npm install axios dotenv

# Dev dependencies
npm install -D typescript @types/node @types/axios ts-node
npm install -D jest @types/jest ts-jest
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin

# Optional: Caching
npm install node-cache

# Optional: Logging
npm install winston

# Optional: Monitoring
npm install @sentry/node
```

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```
</project_setup>

<type_definitions>
## Type Definitions

**src/types/en-api.ts:**
```typescript
export type Region = 'us' | 'ca' | 'eu' | 'au';

export type PageType = 'donation' | 'advocacy' | 'event' | 'survey';

export interface ENClientConfig {
  apiUserToken: string;  // API User token from EN Admin (for authentication)
  region: Region;
  timeout?: number;
  maxRetries?: number;
  debug?: boolean;
}

export interface AuthResponse {
  'ens-auth-token': string;  // Session token for API calls
  expires: number;           // Token lifetime in milliseconds
}

export interface ENPage {
  id: number;
  campaignId: number;
  name: string;
  title: string;
  type: string;
  subType: string;
  clientId: number;
  createdOn: number;       // Epoch timestamp
  modifiedOn: number;      // Epoch timestamp
  campaignBaseUrl: string;
  campaignStatus: string;
  defaultLocale: string;
  trackingParameters?: string;
}

export interface SupporterRecord {
  supporterId: number;
  title?: string;
  firstName?: string;
  lastName?: string;
  emailAddress?: string;
  phoneNumber?: string;
  phoneNumber2?: string;
  address1?: string;
  address2?: string;
  city?: string;
  region?: string;
  postcode?: string;
  country?: string;
  createdOn?: string;
  modifiedOn?: string;
  questions?: Record<string, string>;
}

export interface SupporterQueryResponse {
  pagination: {
    start: number;
    rows: number;
    total: number;
  };
  data: Array<{
    supporterId: number;
    emailAddress: string;
    createdOn: string;
    modifiedOn: string;
    scores?: any[];
    summary?: any;
  }>;
}

export interface QuerySupportersParams {
  type: 'latestCreated' | 'latestModified' | 'suppressed' | 'profile' | 'search';
  daysBack?: number;      // 1-32
  rows?: number;          // max 100
  start?: number;
  exportGroup?: string;   // max 100 chars
  profileId?: number;     // required for type=profile
  filter?: string;        // required for type=search
}

export interface SupporterTransaction {
  transactionId: number;
  supporterId: number;
  type: string;
  amount: number;
  currency: string;
  fund: string;
  appeal: string;
  campaign: string;
  pageId: number;
  pageName: string;
  transactionDate: string;
  status: string;
  paymentType: string;
  recurringId?: number;
}

export interface RecurringTransaction {
  recurringId: number;
  supporterId: number;
  amount: number;
  currency: string;
  frequency: string;
  status: string;
  startDate: string;
  nextDate: string;
  fund: string;
  appeal: string;
  campaign: string;
  pageId: number;
  paymentType: string;
}

export interface ExportJob {
  id: number;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  type: string;
  dateCreated: string;
  dateCompleted?: string;
  totalRows?: number;
  downloadUrl?: string;
}

export interface ExportJobRequest {
  type: string;
  dateFrom?: string;
  dateTo?: string;
  fields?: string[];
  filter?: string;
}

export interface MarketingAutomation {
  id: number;
  name: string;
  type: string;
  status: string;
  createdOn: string;
  modifiedOn: string;
}

export interface MarketingAutomationStats {
  id: number;
  name: string;
  sent: number;
  opened: number;
  clicked: number;
  bounced: number;
  unsubscribed: number;
}

export interface PageComponent {
  id: number;
  pageId: number;
  name: string;
  type: string;
  content?: string;
}

export interface OriginSource {
  supporterId: number;
  source: string;
  medium: string;
  campaign: string;
  content?: string;
  term?: string;
  createdOn?: string;
}

export interface PageProcessRequest {
  supporter: {
    emailAddress: string;
    firstName?: string;
    lastName?: string;
    [key: string]: any;
  };
  transaction?: {
    paymenttype?: string;
    donationAmt?: string;
    recurrpay?: string;
    [key: string]: any;
  };
  [key: string]: any;
}

export interface PageProcessResponse {
  id: number;
  status: string;
  supporterId: number;
  transactionId?: number;
}

export interface ENApiError {
  error: string;
  message: string;
  field?: string;
  value?: any;
}

export interface WriteOptions {
  /** When true, logs the request details without executing. Use to preview write operations. */
  dryRun?: boolean;
}

export interface PaginationParams {
  limit?: number;
  offset?: number;
}

export interface GetPagesParams extends PaginationParams {
  type?: PageType;
}
```
</type_definitions>

<client_implementation>
## Complete Client Implementation

**src/lib/en-client.ts:**
```typescript
import axios, { AxiosInstance, AxiosError } from 'axios';
import type {
  Region,
  ENClientConfig,
  ENPage,
  SupporterRecord,
  SupporterQueryResponse,
  QuerySupportersParams,
  SupporterTransaction,
  RecurringTransaction,
  ExportJob,
  ExportJobRequest,
  MarketingAutomation,
  MarketingAutomationStats,
  OriginSource,
  PageProcessRequest,
  PageProcessResponse,
  GetPagesParams,
} from '../types/en-api';

const REGION_BASE_URLS: Record<Region, string> = {
  us: 'https://us.engagingnetworks.app/ens/service',
  ca: 'https://ca.engagingnetworks.app/ens/service',
  eu: 'https://eu.engagingnetworks.app/ens/service',
  au: 'https://au.engagingnetworks.app/ens/service',
};

export class ENClient {
  private client: AxiosInstance;
  private maxRetries: number;
  private debug: boolean;
  private apiUserToken: string;
  private sessionToken: string | null = null;
  private sessionExpiry: number | null = null;

  constructor(config: ENClientConfig) {
    const baseURL = REGION_BASE_URLS[config.region];

    if (!baseURL) {
      throw new Error(`Invalid region: ${config.region}`);
    }

    if (!config.apiUserToken) {
      throw new Error('API User token is required');
    }

    this.maxRetries = config.maxRetries ?? 3;
    this.debug = config.debug ?? false;
    this.apiUserToken = config.apiUserToken;

    this.client = axios.create({
      baseURL,
      timeout: config.timeout || 30000,
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
    });

    // Request interceptor to add session token
    this.client.interceptors.request.use(async (config) => {
      // Ensure we have a valid session token before each request
      await this.ensureAuthenticated();

      // Add session token to headers
      if (this.sessionToken) {
        config.headers['ens-auth-token'] = this.sessionToken;
      }

      return config;
    });

    // Response interceptor for logging
    this.client.interceptors.response.use(
      response => {
        this.log('Request succeeded', {
          url: response.config.url,
          status: response.status,
        });
        return response;
      },
      error => {
        this.log('Request failed', {
          url: error.config?.url,
          status: error.response?.status,
          message: error.message,
        });
        return Promise.reject(error);
      }
    );
  }

  /**
   * Authenticate with EN API to obtain session token
   */
  private async authenticate(): Promise<void> {
    this.log('Authenticating with EN REST API...');

    try {
      const response = await axios.post<AuthResponse>(
        `${this.client.defaults.baseURL}/authenticate`,
        { 'ens-auth-token': this.apiUserToken },
        {
          headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
          },
        }
      );

      this.sessionToken = response.data['ens-auth-token'];
      this.sessionExpiry = Date.now() + response.data.expires;

      this.log('Authentication successful', {
        expiresIn: `${Math.floor(response.data.expires / 60000)} minutes`,
      });
    } catch (error) {
      if (axios.isAxiosError(error) && error.response?.status === 401) {
        throw new Error(
          'EN API authentication failed. Check your API User token and verify your IP address is whitelisted in EN Admin > Settings > API Users.'
        );
      }
      throw error;
    }
  }

  /**
   * Ensure we have a valid session token, re-authenticate if needed
   */
  private async ensureAuthenticated(): Promise<void> {
    const now = Date.now();
    const needsAuth = !this.sessionToken ||
                      !this.sessionExpiry ||
                      now >= this.sessionExpiry - 60000; // Re-auth 1 min before expiry

    if (needsAuth) {
      await this.authenticate();
    }
  }

  private log(message: string, data?: any): void {
    if (this.debug) {
      console.log(`[ENClient] ${message}`, data ? JSON.stringify(data, null, 2) : '');
    }
  }

  /**
   * Preview a write operation without executing it.
   * Returns the request details that would be sent.
   */
  private previewWrite(method: string, endpoint: string, body?: any): null {
    console.log('\n=== DRY RUN (no data will be sent) ===');
    console.log(`${method} ${this.client.defaults.baseURL}${endpoint}`);
    if (body) {
      console.log('Request body:', JSON.stringify(body, null, 2));
    }
    console.log('=====================================\n');
    return null;
  }

  private async fetchWithRetry<T>(
    fn: () => Promise<T>,
    attempt: number = 0
  ): Promise<T> {
    try {
      return await fn();
    } catch (error) {
      if (axios.isAxiosError(error)) {
        const shouldRetry = this.shouldRetry(error, attempt);

        if (shouldRetry) {
          const delay = this.calculateBackoff(attempt);
          this.log(`Retry attempt ${attempt + 1}/${this.maxRetries} after ${delay}ms`);

          await this.sleep(delay);
          return this.fetchWithRetry(fn, attempt + 1);
        }
      }
      throw this.enhanceError(error);
    }
  }

  private shouldRetry(error: AxiosError, attempt: number): boolean {
    if (attempt >= this.maxRetries) return false;

    const status = error.response?.status;

    // Retry on rate limit or server errors
    return status === 429 || (status !== undefined && status >= 500);
  }

  private calculateBackoff(attempt: number): number {
    // Exponential backoff: 1s, 2s, 4s, 8s (max 10s)
    const baseDelay = Math.min(1000 * Math.pow(2, attempt), 10000);
    // Add jitter (0-1000ms) to prevent thundering herd
    const jitter = Math.random() * 1000;
    return baseDelay + jitter;
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  private enhanceError(error: unknown): Error {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status;
      const message = error.response?.data?.message || error.message;

      if (status === 401) {
        return new Error('EN API authentication failed. Check your API token.');
      }
      if (status === 403) {
        return new Error('EN API access forbidden. Check permissions.');
      }
      if (status === 404) {
        return new Error(`EN API resource not found: ${error.config?.url}`);
      }
      if (status === 429) {
        return new Error('EN API rate limit exceeded.');
      }
      if (status && status >= 500) {
        return new Error(`EN API server error (${status}): ${message}`);
      }

      return new Error(`EN API error: ${message}`);
    }

    return error instanceof Error ? error : new Error(String(error));
  }

  // ── Page Endpoints ──────────────────────────────────────────────────

  /**
   * List pages with optional filtering
   * GET /page
   */
  async getPages(params?: GetPagesParams): Promise<ENPage[]> {
    this.log('Fetching pages', params);

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<ENPage[]>('/page', {
        params: {
          ...params,
          limit: params?.limit ?? 100,
        },
      });
      return response.data;
    });
  }

  /**
   * Get single page details
   * GET /page/{id}
   */
  async getPage(pageId: number): Promise<ENPage> {
    this.log('Fetching page details', { pageId });

    if (!pageId) {
      throw new Error('Page ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<ENPage>(`/page/${pageId}`);
      return response.data;
    });
  }

  /**
   * Process a page submission (donation, advocacy action, etc.)
   * POST /page/{id}/process
   */
  async processPage(pageId: number, data: PageProcessRequest, options?: WriteOptions): Promise<PageProcessResponse | null> {
    this.log('Processing page', { pageId });

    if (!pageId) {
      throw new Error('Page ID is required');
    }

    if (options?.dryRun) {
      return this.previewWrite('POST', `/page/${pageId}/process`, data);
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.post<PageProcessResponse>(
        `/page/${pageId}/process`,
        data
      );
      return response.data;
    });
  }

  // ── Supporter Endpoints ─────────────────────────────────────────────

  /**
   * Query supporters with various filters
   * GET /supporter/query
   */
  async querySupporters(params: QuerySupportersParams): Promise<SupporterQueryResponse> {
    this.log('Querying supporters', params);

    // Validation
    if (params.type === 'profile' && !params.profileId) {
      throw new Error('profileId is required for type=profile queries');
    }
    if (params.type === 'search' && !params.filter) {
      throw new Error('filter is required for type=search queries');
    }
    if (params.daysBack !== undefined && (params.daysBack < 1 || params.daysBack > 32)) {
      throw new Error('daysBack must be between 1 and 32');
    }
    if (params.rows !== undefined && params.rows > 100) {
      throw new Error('rows cannot exceed 100');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterQueryResponse>(
        '/supporter/query',
        { params }
      );
      return response.data;
    });
  }

  /**
   * Get supporter by email address
   * GET /supporter?email={email}
   */
  async getSupporter(email: string): Promise<SupporterRecord> {
    this.log('Fetching supporter by email', { email });

    if (!email) {
      throw new Error('Email address is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterRecord>(
        '/supporter',
        { params: { email } }
      );
      return response.data;
    });
  }

  /**
   * Get supporter by ID
   * GET /supporter/{id}
   */
  async getSupporterById(supporterId: number): Promise<SupporterRecord> {
    this.log('Fetching supporter by ID', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterRecord>(
        `/supporter/${supporterId}`
      );
      return response.data;
    });
  }

  /**
   * Create or update a supporter record (upsert by email)
   * POST /supporter
   */
  async createOrUpdateSupporter(data: Partial<SupporterRecord>, options?: WriteOptions): Promise<SupporterRecord | null> {
    this.log('Creating or updating supporter', { email: data.emailAddress });

    if (!data.emailAddress) {
      throw new Error('emailAddress is required');
    }

    if (options?.dryRun) {
      return this.previewWrite('POST', '/supporter', data);
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.post<SupporterRecord>(
        '/supporter',
        data
      );
      return response.data;
    });
  }

  /**
   * Update an existing supporter by ID
   * PUT /supporter/{id}
   */
  async updateSupporter(supporterId: number, data: Partial<SupporterRecord>, options?: WriteOptions): Promise<SupporterRecord | null> {
    this.log('Updating supporter', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    if (options?.dryRun) {
      return this.previewWrite('PUT', `/supporter/${supporterId}`, data);
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.put<SupporterRecord>(
        `/supporter/${supporterId}`,
        data
      );
      return response.data;
    });
  }

  /**
   * Delete a supporter by ID
   * DELETE /supporter/{id}
   */
  async deleteSupporter(supporterId: number, options?: WriteOptions): Promise<void> {
    this.log('Deleting supporter', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    if (options?.dryRun) {
      this.previewWrite('DELETE', `/supporter/${supporterId}`);
      return;
    }

    return this.fetchWithRetry(async () => {
      await this.client.delete(`/supporter/${supporterId}`);
    });
  }

  // ── Supporter Transaction Endpoints ─────────────────────────────────

  /**
   * Get transactions for a supporter
   * GET /supporter/{supporterId}/transactions
   */
  async getSupporterTransactions(supporterId: number): Promise<SupporterTransaction[]> {
    this.log('Fetching supporter transactions', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterTransaction[]>(
        `/supporter/${supporterId}/transactions`
      );
      return response.data;
    });
  }

  /**
   * Get a specific transaction for a supporter
   * GET /supporter/{supporterId}/transactions/{transactionId}
   */
  async getTransactionDetails(
    supporterId: number,
    transactionId: number
  ): Promise<SupporterTransaction> {
    this.log('Fetching transaction details', { supporterId, transactionId });

    if (!supporterId) throw new Error('Supporter ID is required');
    if (!transactionId) throw new Error('Transaction ID is required');

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterTransaction>(
        `/supporter/${supporterId}/transactions/${transactionId}`
      );
      return response.data;
    });
  }

  /**
   * Get recurring transactions for a supporter
   * GET /supporter/{supporterId}/transactions/recurring
   */
  async getRecurringTransactions(supporterId: number): Promise<RecurringTransaction[]> {
    this.log('Fetching recurring transactions', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<RecurringTransaction[]>(
        `/supporter/${supporterId}/transactions/recurring`
      );
      return response.data;
    });
  }

  /**
   * Update a recurring transaction for a supporter
   * PUT /supporter/{supporterId}/transactions/recurring/{transactionId}
   */
  async updateRecurringTransaction(
    supporterId: number,
    transactionId: string,
    data: Partial<RecurringTransaction>,
    options?: WriteOptions
  ): Promise<RecurringTransaction | null> {
    this.log('Updating recurring transaction', { supporterId, transactionId });

    if (!supporterId) throw new Error('Supporter ID is required');
    if (!transactionId) throw new Error('Transaction ID is required');

    if (options?.dryRun) {
      return this.previewWrite('PUT', `/supporter/${supporterId}/transactions/recurring/${transactionId}`, data);
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.put<RecurringTransaction>(
        `/supporter/${supporterId}/transactions/recurring/${encodeURIComponent(transactionId)}`,
        data
      );
      return response.data;
    });
  }

  // ── Origin Source Endpoints ─────────────────────────────────────────

  /**
   * Get origin source for a supporter
   * GET /supporter/{supporterId}/originSource
   */
  async getOriginSource(supporterId: number): Promise<OriginSource> {
    this.log('Fetching origin source', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<OriginSource>(
        `/supporter/${supporterId}/originSource`
      );
      return response.data;
    });
  }

  /**
   * Set origin source for a supporter
   * POST /supporter/{supporterId}/originSource
   */
  async setOriginSource(supporterId: number, data: Partial<OriginSource>, options?: WriteOptions): Promise<OriginSource | null> {
    this.log('Setting origin source', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    if (options?.dryRun) {
      return this.previewWrite('POST', `/supporter/${supporterId}/originSource`, data);
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.post<OriginSource>(
        `/supporter/${supporterId}/originSource`,
        data
      );
      return response.data;
    });
  }

  /**
   * Update origin source for a supporter
   * PUT /supporter/{supporterId}/originSource
   */
  async updateOriginSource(supporterId: number, data: Partial<OriginSource>, options?: WriteOptions): Promise<OriginSource | null> {
    this.log('Updating origin source', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    if (options?.dryRun) {
      return this.previewWrite('PUT', `/supporter/${supporterId}/originSource`, data);
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.put<OriginSource>(
        `/supporter/${supporterId}/originSource`,
        data
      );
      return response.data;
    });
  }

  /**
   * Delete origin source for a supporter
   * DELETE /supporter/{supporterId}/originSource
   */
  async deleteOriginSource(supporterId: number, options?: WriteOptions): Promise<void> {
    this.log('Deleting origin source', { supporterId });

    if (!supporterId) {
      throw new Error('Supporter ID is required');
    }

    if (options?.dryRun) {
      this.previewWrite('DELETE', `/supporter/${supporterId}/originSource`);
      return;
    }

    return this.fetchWithRetry(async () => {
      await this.client.delete(`/supporter/${supporterId}/originSource`);
    });
  }

  // ── Export Job Endpoints ────────────────────────────────────────────

  /**
   * Create a data export job
   * POST /export
   */
  async createExportJob(data: ExportJobRequest): Promise<ExportJob> {
    this.log('Creating export job', data);

    return this.fetchWithRetry(async () => {
      const response = await this.client.post<ExportJob>('/export', data);
      return response.data;
    });
  }

  /**
   * Get the status of an export job
   * GET /export/{id}
   */
  async getExportJobStatus(exportId: number): Promise<ExportJob> {
    this.log('Fetching export job status', { exportId });

    if (!exportId) {
      throw new Error('Export ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<ExportJob>(`/export/${exportId}`);
      return response.data;
    });
  }

  /**
   * Download the results of a completed export job
   * GET /export/{id}/download
   */
  async downloadExportJob(exportId: number): Promise<string> {
    this.log('Downloading export job', { exportId });

    if (!exportId) {
      throw new Error('Export ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<string>(
        `/export/${exportId}/download`,
        { responseType: 'text' as any }
      );
      return response.data;
    });
  }

  // ── Marketing Automation Endpoints ──────────────────────────────────

  /**
   * List marketing automations
   * GET /marketingAutomation
   */
  async listMarketingAutomations(): Promise<MarketingAutomation[]> {
    this.log('Listing marketing automations');

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<MarketingAutomation[]>(
        '/marketingAutomation'
      );
      return response.data;
    });
  }

  /**
   * Get statistics for a marketing automation
   * GET /marketingAutomation/{id}/stats
   */
  async getMarketingAutomationStats(automationId: number): Promise<MarketingAutomationStats> {
    this.log('Fetching marketing automation stats', { automationId });

    if (!automationId) {
      throw new Error('Automation ID is required');
    }

    return this.fetchWithRetry(async () => {
      const response = await this.client.get<MarketingAutomationStats>(
        `/marketingAutomation/${automationId}/stats`
      );
      return response.data;
    });
  }
}
```
</client_implementation>

<usage_example>
## Usage Example

**src/index.ts:**
```typescript
import * as dotenv from 'dotenv';
import { ENClient } from './lib/en-client';
import type { Region } from './types/en-api';

dotenv.config();

async function main(): Promise<void> {
  // Initialize client with API User token
  // Client will automatically authenticate and manage session tokens
  const client = new ENClient({
    apiUserToken: process.env.EN_API_USER_TOKEN!,
    region: (process.env.EN_REGION as Region) || 'ca',
    timeout: 30000,
    maxRetries: 3,
    debug: process.env.NODE_ENV === 'development',
  });

  try {
    // ── List pages ───────────────────────────────────────────────────
    console.log('Fetching pages...');
    const pages = await client.getPages({ type: 'donation' });
    console.log(`Found ${pages.length} donation pages`);

    if (pages.length > 0) {
      const page = pages[0];
      console.log(`  First page: ${page.name} (ID: ${page.id}, status: ${page.campaignStatus})`);
    }

    // ── Look up a supporter by email ─────────────────────────────────
    console.log('\nLooking up supporter...');
    const supporter = await client.getSupporter('supporter@example.org');
    console.log(`Supporter ID: ${supporter.supporterId}`);
    console.log(`  Name: ${supporter.firstName} ${supporter.lastName}`);

    // ── Fetch supporter transactions ─────────────────────────────────
    console.log('\nFetching transactions...');
    const transactions = await client.getSupporterTransactions(supporter.supporterId);
    console.log(`Found ${transactions.length} transactions`);

    for (const txn of transactions.slice(0, 3)) {
      console.log(`  ${txn.transactionDate} — $${txn.amount} ${txn.currency} (${txn.status})`);
    }

    // ── Check recurring donations ────────────────────────────────────
    console.log('\nFetching recurring donations...');
    const recurring = await client.getRecurringTransactions(supporter.supporterId);
    console.log(`Found ${recurring.length} recurring donations`);

    for (const rec of recurring) {
      console.log(`  $${rec.amount} ${rec.currency} ${rec.frequency} — next: ${rec.nextDate}`);
    }

    // ── Create or update a supporter ─────────────────────────────────
    console.log('\nUpserting supporter...');
    const upserted = await client.createOrUpdateSupporter({
      emailAddress: 'new-supporter@example.org',
      firstName: 'Jane',
      lastName: 'Doe',
      country: 'CA',
    });
    console.log(`Upserted supporter ID: ${upserted.supporterId}`);

    // ── Query recently created supporters ────────────────────────────
    console.log('\nQuerying recent supporters...');
    const recent = await client.querySupporters({
      type: 'latestCreated',
      daysBack: 7,
      rows: 10,
    });
    console.log(`Found ${recent.pagination.total} supporters created in last 7 days`);

    // ── Process a page submission ────────────────────────────────────
    console.log('\nProcessing page submission...');
    if (pages.length > 0) {
      const result = await client.processPage(pages[0].id, {
        supporter: {
          emailAddress: 'donor@example.org',
          firstName: 'John',
          lastName: 'Smith',
        },
        transaction: {
          donationAmt: '25.00',
          paymenttype: 'Visa',
        },
      });
      console.log(`Submission processed — supporter: ${result.supporterId}`);
    }

    console.log('\nAll operations completed successfully');
  } catch (error) {
    console.error('Error:', error instanceof Error ? error.message : error);
    process.exit(1);
  }
}

main();
```

**Run:**
```bash
npx ts-node src/index.ts
```
</usage_example>

<package_json>
## Package Scripts

**package.json:**
```json
{
  "name": "en-api-client",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/ --ext .ts",
    "typecheck": "tsc --noEmit"
  }
}
```
</package_json>
