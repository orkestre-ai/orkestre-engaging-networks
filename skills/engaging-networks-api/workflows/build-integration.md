# Workflow: Build EN API Integration

<required_reading>
**Read these reference files NOW before building:**
1. references/authentication.md
2. references/endpoints.md
3. references/integration-architecture.md
4. references/typescript-client.md (if TypeScript) OR references/python-client.md (if Python)
5. references/rate-limits.md
</required_reading>

<process>
## Step 1: Confirm Requirements

Ask the user:

**What language/framework?**
- TypeScript/Node.js
- Python
- Other (may need adaptation)

**What's the integration goal?**
- Data collection (fetch pages, transactions, stats)
- Real-time sync (webhook-style updates)
- Campaign widget (display live metrics)
- Donor roll generation
- CRM synchronization

**Which API services do you need?**
- Page Services: List pages, view details, process submissions, get survey responses
- Supporter Services: Query, create, update, delete supporters
- Transaction Services: List transactions, recurring management
- Export Jobs: Bulk data export
- Marketing Automations: List, stats, add supporters
- Page Components: Templates, code blocks, widgets, form blocks
- Origin Source: Acquisition tracking
- Account: Audit logs

**Regional instance:**
- US, US2, CA, EU, or AU?

## Step 2: Project Setup

### TypeScript/Node.js

```bash
# Initialize project
mkdir en-api-integration
cd en-api-integration
npm init -y

# Install dependencies
npm install axios dotenv
npm install -D typescript @types/node @types/axios ts-node

# Initialize TypeScript
npx tsc --init
```

**Update tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

### Python

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install requests python-dotenv tenacity

# Create requirements.txt
cat > requirements.txt << EOF
requests==2.31.0
python-dotenv==1.0.0
tenacity==8.2.3
EOF
```

## Step 3: Environment Configuration

Create `.env` file:

```bash
# Engaging Networks API Configuration
EN_API_TOKEN=your_token_here
EN_REGION=us  # us, us2, or ca (EU/AU clients use ca)
```

**Security checklist:**
- [ ] .env added to .gitignore
- [ ] Token stored securely (not in code)
- [ ] No token in logs or error messages

## Step 4: Create Base Client

### TypeScript Implementation

**src/lib/en-client.ts:**

```typescript
import axios, { AxiosInstance, AxiosError } from 'axios';

// EU and AU clients currently use the CA servers
const REGION_BASE_URLS = {
  us: 'https://us.engagingnetworks.app/ens/service',
  us2: 'https://us2.engagingnetworks.app/ens/service',
  ca: 'https://ca.engagingnetworks.app/ens/service',
} as const;

type Region = keyof typeof REGION_BASE_URLS;

interface ENClientConfig {
  apiToken: string;
  region: Region;
  timeout?: number;
  maxRetries?: number;
}

export class ENClient {
  private client: AxiosInstance;
  private maxRetries: number;
  private apiUserToken: string;
  private sessionToken: string | null = null;
  private sessionExpiry: number | null = null;

  constructor(config: ENClientConfig) {
    const baseURL = REGION_BASE_URLS[config.region];
    this.apiUserToken = config.apiToken;

    this.client = axios.create({
      baseURL,
      timeout: config.timeout || 30000,
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
    });

    // Request interceptor to add session token to all requests
    this.client.interceptors.request.use(async (config) => {
      await this.ensureAuthenticated();
      if (this.sessionToken) {
        config.headers['ens-auth-token'] = this.sessionToken;
      }
      return config;
    });

    this.maxRetries = config.maxRetries || 3;
  }

  /**
   * Authenticate with EN API to obtain session token
   */
  private async authenticate(): Promise<void> {
    const response = await axios.post(
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
  }

  /**
   * Ensure we have a valid session token, re-authenticate if needed
   */
  private async ensureAuthenticated(): Promise<void> {
    const now = Date.now();
    const needsAuth = !this.sessionToken ||
                      !this.sessionExpiry ||
                      now >= this.sessionExpiry - 60000;

    if (needsAuth) {
      await this.authenticate();
    }
  }

  /**
   * Fetch with exponential backoff retry logic
   */
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
          console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
          await this.sleep(delay);
          return this.fetchWithRetry(fn, attempt + 1);
        }
      }
      throw error;
    }
  }

  private shouldRetry(error: AxiosError, attempt: number): boolean {
    if (attempt >= this.maxRetries) return false;

    const status = error.response?.status;

    // Retry on rate limit (429) or server errors (5xx)
    return status === 429 || (status !== undefined && status >= 500);
  }

  private calculateBackoff(attempt: number): number {
    // Exponential backoff: 1s, 2s, 4s, 8s (with jitter)
    const baseDelay = Math.min(1000 * Math.pow(2, attempt), 10000);
    const jitter = Math.random() * 1000;
    return baseDelay + jitter;
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  /**
   * List pages with optional filtering
   */
  async getPages(params?: {
    type?: string;
    status?: string;
  }): Promise<ENPage[]> {
    return this.fetchWithRetry(async () => {
      const response = await this.client.get<ENPage[]>('/page', { params });
      return response.data;
    });
  }

  /**
   * Get single page details
   */
  async getPage(pageId: number): Promise<ENPage> {
    return this.fetchWithRetry(async () => {
      const response = await this.client.get<ENPage>(`/page/${pageId}`);
      return response.data;
    });
  }

  /**
   * Process a page submission (e.g., submit a donation or advocacy action)
   */
  async processPage(pageId: number, data: Record<string, any>): Promise<any> {
    return this.fetchWithRetry(async () => {
      const response = await this.client.post(`/page/${pageId}/process`, data);
      return response.data;
    });
  }

  /**
   * Look up a supporter by email address
   */
  async getSupporterByEmail(email: string): Promise<SupporterRecord[]> {
    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterRecord[]>('/supporter', {
        params: { email },
      });
      return response.data;
    });
  }

  /**
   * Get transactions for a specific supporter
   */
  async getSupporterTransactions(
    supporterId: number
  ): Promise<SupporterTransaction[]> {
    return this.fetchWithRetry(async () => {
      const response = await this.client.get<SupporterTransaction[]>(
        `/supporter/${supporterId}/transactions`
      );
      return response.data;
    });
  }
}

// Type definitions
export interface ENPage {
  id: number;
  campaignId: number;
  name: string;
  title: string;
  type: string;
  subType: string;
  clientId: number;
  createdOn: number;    // epoch timestamp
  modifiedOn: number;   // epoch timestamp
  campaignBaseUrl: string;
  campaignStatus: string;
  defaultLocale: string;
}

export interface SupporterRecord {
  supporterId: number;
  firstName: string;
  lastName: string;
  emailAddress: string;
  createdOn: number;       // epoch timestamp
  modifiedOn: number;      // epoch timestamp
  [key: string]: any;      // additional profile fields vary by account
}

export interface SupporterTransaction {
  transactionId: number;
  supporterId: number;
  totalAmount: number;
  currency: string;
  transactionDate: number; // epoch timestamp
  transactionType: string;
  status: string;
  [key: string]: any;
}
```

### Python Implementation

**src/en_client.py:**

```python
import os
import time
import random
from typing import Optional, Dict, List, Any
from dataclasses import dataclass
import requests
from requests.exceptions import RequestException
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

# EU and AU clients currently use the CA servers
REGION_BASE_URLS = {
    'us': 'https://us.engagingnetworks.app/ens/service',
    'us2': 'https://us2.engagingnetworks.app/ens/service',
    'ca': 'https://ca.engagingnetworks.app/ens/service',
}

@dataclass
class ENClientConfig:
    api_token: str
    region: str = 'us'
    timeout: int = 30
    max_retries: int = 3

class RateLimitError(Exception):
    """Rate limit exceeded"""
    pass

class ENClient:
    def __init__(self, config: ENClientConfig):
        self.base_url = REGION_BASE_URLS[config.region]
        self.timeout = config.timeout
        self.max_retries = config.max_retries
        self.api_user_token = config.api_token
        self.session_token = None
        self.session_expiry = None

        self.session = requests.Session()
        self.session.headers.update({
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        })

    def authenticate(self):
        """Authenticate and obtain session token"""
        response = requests.post(
            f'{self.base_url}/authenticate',
            json={'ens-auth-token': self.api_user_token},
            headers={
                'Content-Type': 'application/json',
                'Accept': 'application/json',
            }
        )

        response.raise_for_status()
        data = response.json()

        self.session_token = data['ens-auth-token']
        self.session_expiry = time.time() + (data['expires'] / 1000)

    def ensure_authenticated(self):
        """Ensure we have a valid session token"""
        now = time.time()
        needs_auth = (
            not self.session_token or
            not self.session_expiry or
            now >= self.session_expiry - 60
        )

        if needs_auth:
            self.authenticate()

    def _should_retry(self, exception: Exception) -> bool:
        """Determine if request should be retried"""
        if isinstance(exception, requests.exceptions.HTTPError):
            # Retry on rate limit (429) or server errors (5xx)
            status_code = exception.response.status_code
            return status_code == 429 or status_code >= 500
        return isinstance(exception, requests.exceptions.RequestException)

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type((RateLimitError, RequestException)),
        reraise=True
    )
    def _request(self, method: str, endpoint: str, **kwargs) -> Dict[str, Any]:
        """Make HTTP request with retry logic"""
        # Ensure we have a valid session token
        self.ensure_authenticated()

        # Add session token to headers
        headers = kwargs.get('headers', {})
        headers['ens-auth-token'] = self.session_token
        kwargs['headers'] = headers

        url = f"{self.base_url}{endpoint}"

        try:
            response = self.session.request(
                method,
                url,
                timeout=self.timeout,
                **kwargs
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                raise RateLimitError("Rate limit exceeded") from e
            raise

    def get_pages(
        self,
        page_type: Optional[str] = None,
        status: Optional[str] = None,
    ) -> List[Dict[str, Any]]:
        """List pages with optional filtering"""
        params: Dict[str, Any] = {}
        if page_type:
            params['type'] = page_type
        if status:
            params['status'] = status

        return self._request('GET', '/page', params=params)

    def get_page(self, page_id: int) -> Dict[str, Any]:
        """Get single page details"""
        return self._request('GET', f'/page/{page_id}')

    def process_page(self, page_id: int, data: Dict[str, Any]) -> Dict[str, Any]:
        """Process a page submission (e.g., submit a donation or advocacy action)"""
        return self._request('POST', f'/page/{page_id}/process', json=data)

    def get_supporter_by_email(self, email: str) -> List[Dict[str, Any]]:
        """Look up a supporter by email address"""
        return self._request('GET', '/supporter', params={'email': email})

    def get_supporter_transactions(self, supporter_id: int) -> List[Dict[str, Any]]:
        """Get transactions for a specific supporter"""
        return self._request('GET', f'/supporter/{supporter_id}/transactions')
```

## Step 5: Create Usage Example

### TypeScript

**src/index.ts:**

```typescript
import * as dotenv from 'dotenv';
import { ENClient } from './lib/en-client';

dotenv.config();

async function main() {
  const client = new ENClient({
    apiToken: process.env.EN_API_TOKEN!,
    region: (process.env.EN_REGION as any) || 'us',
  });

  try {
    // 1. Fetch all live pages
    console.log('Fetching live pages...');
    const pages = await client.getPages({ status: 'live' });
    console.log(`Found ${pages.length} live pages`);

    if (pages.length > 0) {
      const page = pages[0];
      console.log(`\nFirst page: "${page.title}" (type: ${page.type}, id: ${page.id})`);
    }

    // 2. Look up a supporter by email
    const email = 'jane.doe@example.org';
    console.log(`\nLooking up supporter: ${email}`);
    const supporters = await client.getSupporterByEmail(email);

    if (supporters.length > 0) {
      const supporter = supporters[0];
      console.log(`Found: ${supporter.firstName} ${supporter.lastName} (ID: ${supporter.supporterId})`);

      // 3. Get transactions for that supporter
      console.log(`\nFetching transactions for supporter ${supporter.supporterId}...`);
      const transactions = await client.getSupporterTransactions(supporter.supporterId);
      console.log(`Found ${transactions.length} transactions`);

      for (const txn of transactions.slice(0, 5)) {
        const date = new Date(txn.transactionDate).toLocaleDateString();
        console.log(`  ${date} - ${txn.currency} ${txn.totalAmount} (${txn.transactionType})`);
      }
    } else {
      console.log('No supporter found with that email.');
    }
  } catch (error) {
    console.error('Error:', error);
    process.exit(1);
  }
}

main();
```

### Python

**main.py:**

```python
import os
from datetime import datetime
from dotenv import load_dotenv
from src.en_client import ENClient, ENClientConfig

load_dotenv()

def main():
    client = ENClient(ENClientConfig(
        api_token=os.environ['EN_API_TOKEN'],
        region=os.environ.get('EN_REGION', 'us'),
    ))

    try:
        # 1. Fetch all live pages
        print('Fetching live pages...')
        pages = client.get_pages(status='live')
        print(f'Found {len(pages)} live pages')

        if pages:
            page = pages[0]
            print(f"\nFirst page: \"{page['title']}\" (type: {page['type']}, id: {page['id']})")

        # 2. Look up a supporter by email
        email = 'jane.doe@example.org'
        print(f'\nLooking up supporter: {email}')
        supporters = client.get_supporter_by_email(email)

        if supporters:
            supporter = supporters[0]
            print(f"Found: {supporter['firstName']} {supporter['lastName']} (ID: {supporter['supporterId']})")

            # 3. Get transactions for that supporter
            supporter_id = supporter['supporterId']
            print(f'\nFetching transactions for supporter {supporter_id}...')
            transactions = client.get_supporter_transactions(supporter_id)
            print(f'Found {len(transactions)} transactions')

            for txn in transactions[:5]:
                date = datetime.fromtimestamp(txn['transactionDate'] / 1000).strftime('%Y-%m-%d')
                print(f"  {date} - {txn['currency']} {txn['totalAmount']} ({txn['transactionType']})")
        else:
            print('No supporter found with that email.')

    except Exception as e:
        print(f'Error: {e}')
        exit(1)

if __name__ == '__main__':
    main()
```

## Step 6: Add .gitignore

```bash
cat > .gitignore << EOF
# Environment variables
.env
.env.local

# Dependencies
node_modules/
venv/
__pycache__/

# Build output
dist/
build/
*.pyc

# IDE
.vscode/
.idea/
*.swp
EOF
```

## Step 7: Verify

Test the integration:

### TypeScript

```bash
# Compile
npm run build  # or npx tsc

# Run
npm start  # or npx ts-node src/index.ts
```

### Python

```bash
python main.py
```

**Expected output:**
```
Fetching live pages...
Found 12 live pages

First page: "Emergency Relief Fund" (type: donation, id: 54321)

Looking up supporter: jane.doe@example.org
Found: Jane Doe (ID: 123456)

Fetching transactions for supporter 123456...
Found 3 transactions
  2026-02-15 - USD 50.00 (one-time)
  2026-01-10 - USD 25.00 (recurring)
  2025-12-01 - USD 100.00 (one-time)
```

**If you see errors:**
- 401: Check EN_API_TOKEN in .env
- 404: Verify EN_REGION matches your account
- Connection refused: Check base URL for region

</process>

<anti_patterns>
Avoid:
- **Hardcoding tokens** - Always use environment variables
- **No retry logic** - Rate limits and transient errors will break your integration
- **Synchronous pagination** - Can take forever for large datasets
- **Ignoring rate limit headers** - Monitor X-RateLimit-Remaining
- **Client-side token exposure** - EN API tokens must stay on the server
- **No error logging** - You'll struggle to debug production issues
- **Fixed delays** - Use exponential backoff with jitter, not fixed sleep()
</anti_patterns>

<write_safety>
## Write Operation Safety

When building client methods that write data (POST/PUT/DELETE to supporter, page/process, originsource, or recurring transaction endpoints):

1. **Always include a `dryRun` option** on write methods so users can preview what would be sent:
   ```typescript
   async updateSupporter(id: number, data: any, options?: { dryRun?: boolean }) {
     if (options?.dryRun) {
       console.log('=== DRY RUN ===');
       console.log(`PUT /supporter/${id}`);
       console.log(JSON.stringify(data, null, 2));
       return null;
     }
     // actual call...
   }
   ```

2. **Default to dryRun in examples** -- When showing write operation usage, demonstrate the dry-run first:
   ```typescript
   // Preview first
   await client.updateSupporter(212200, { 'First Name': 'Jane' }, { dryRun: true });
   // Then execute after user confirms
   await client.updateSupporter(212200, { 'First Name': 'Jane' });
   ```

3. **Warn about destructive operations** -- DELETE endpoints should log a warning before execution.
</write_safety>

<success_criteria>
A well-built EN API integration:
- ✓ Authenticates successfully with token
- ✓ Fetches data from correct regional instance
- ✓ Implements exponential backoff for 429/5xx errors
- ✓ Has comprehensive TypeScript types (if TS) or type hints (if Python)
- ✓ Never exposes token in logs or client code
- ✓ Uses pagination for large datasets
- ✓ Handles all documented error codes gracefully
- ✓ Write methods include dryRun option for safe previewing
- ✓ Includes basic usage example
- ✓ Has .env in .gitignore
- ✓ Runs successfully and fetches real data
</success_criteria>
