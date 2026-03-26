# Workflow: Test EN API Integration

<required_reading>
**Read NOW:**
1. references/testing-strategies.md
2. references/error-handling.md
</required_reading>

<process>
## Step 1: Set Up Testing Framework

### TypeScript/Jest

```bash
npm install -D jest @types/jest ts-jest @types/node
npm install -D nock  # For HTTP mocking
```

**jest.config.js:**
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: ['src/**/*.ts'],
};
```

### Python/pytest

```bash
pip install pytest pytest-cov responses
```

**pytest.ini:**
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
```

## Step 2: Unit Tests (Mock EN API)

### TypeScript Example

**tests/en-client.test.ts:**
```typescript
import nock from 'nock';
import { ENClient } from '../src/lib/en-client';

describe('ENClient', () => {
  let client: ENClient;
  const BASE_URL = 'https://us.engagingnetworks.app/ens/service';

  beforeEach(() => {
    client = new ENClient({
      apiToken: 'test-token',
      region: 'us',
    });
  });

  afterEach(() => {
    nock.cleanAll();
  });

  describe('getPages', () => {
    it('should fetch pages successfully', async () => {
      const mockPages = [
        {
          id: '12345',
          name: 'Test Page',
          type: 'donation',
          status: 'live',
          url: 'https://example.org/donate',
          createdDate: '2024-01-01T00:00:00Z',
          modifiedDate: '2024-01-01T00:00:00Z',
          locale: 'en_US',
        },
      ];

      nock(BASE_URL)
        .get('/page')
        .query({ type: 'donation', status: 'live', limit: 100 })
        .reply(200, mockPages);

      const pages = await client.getPages({
        type: 'donation',
        status: 'live',
        limit: 100,
      });

      expect(pages).toHaveLength(1);
      expect(pages[0].name).toBe('Test Page');
    });

    it('should handle empty results', async () => {
      nock(BASE_URL)
        .get('/page')
        .query({ type: 'donation', status: 'live' })
        .reply(200, []);

      const pages = await client.getPages({
        type: 'donation',
        status: 'live',
      });

      expect(pages).toHaveLength(0);
    });

    it('should retry on 429 rate limit', async () => {
      nock(BASE_URL)
        .get('/page')
        .reply(429, { error: 'Rate limit exceeded' })
        .get('/page')
        .reply(200, [{ id: '123', name: 'Test' }]);

      const pages = await client.getPages();

      expect(pages).toHaveLength(1);
    });

    it('should retry on 500 server error', async () => {
      nock(BASE_URL)
        .get('/page')
        .reply(500, { error: 'Internal server error' })
        .get('/page')
        .reply(200, [{ id: '123', name: 'Test' }]);

      const pages = await client.getPages();

      expect(pages).toHaveLength(1);
    });

    it('should throw on 401 unauthorized', async () => {
      nock(BASE_URL)
        .get('/page')
        .reply(401, { error: 'Unauthorized' });

      await expect(client.getPages()).rejects.toThrow();
    });

    it('should throw on 404 not found', async () => {
      nock(BASE_URL)
        .get('/page/invalid-id')
        .reply(404, { error: 'Not found' });

      await expect(client.getPage('invalid-id')).rejects.toThrow();
    });
  });

  describe('getPage', () => {
    it('should fetch page details', async () => {
      const mockPage = {
        id: '12345',
        name: 'Test Page',
        type: 'donation',
        status: 'live',
        url: 'https://example.org/donate',
        formFields: [
          {
            name: 'firstName',
            label: 'First Name',
            type: 'text',
            required: true,
          },
        ],
      };

      nock(BASE_URL)
        .get('/page/12345')
        .reply(200, mockPage);

      const page = await client.getPage('12345');

      expect(page.name).toBe('Test Page');
      expect(page.formFields).toHaveLength(1);
    });
  });

  describe('querySupporters', () => {
    it('should query supporters successfully', async () => {
      const mockSupporters = [
        {
          supporterId: '100001',
          firstName: 'Jane',
          lastName: 'Doe',
          emailAddress: 'jane@example.org',
        },
        {
          supporterId: '100002',
          firstName: 'John',
          lastName: 'Smith',
          emailAddress: 'john@example.org',
        },
      ];

      nock(BASE_URL)
        .get('/supporter/query')
        .query({ email: 'jane@example.org' })
        .reply(200, mockSupporters);

      const supporters = await client.querySupporters({ email: 'jane@example.org' });

      expect(supporters).toHaveLength(2);
      expect(supporters[0].firstName).toBe('Jane');
    });

    it('should return empty array when no supporters match', async () => {
      nock(BASE_URL)
        .get('/supporter/query')
        .query({ email: 'nonexistent@example.org' })
        .reply(200, []);

      const supporters = await client.querySupporters({ email: 'nonexistent@example.org' });

      expect(supporters).toHaveLength(0);
    });
  });

  describe('getSupporterTransactions', () => {
    it('should fetch transactions for a supporter', async () => {
      const mockTransactions = [
        {
          transactionId: 'txn-001',
          supporterId: '100001',
          amount: 50.0,
          currency: 'USD',
          date: '2024-01-15T10:30:00Z',
          type: 'donation',
        },
        {
          transactionId: 'txn-002',
          supporterId: '100001',
          amount: 25.0,
          currency: 'USD',
          date: '2024-02-10T14:00:00Z',
          type: 'donation',
        },
      ];

      nock(BASE_URL)
        .get('/supporter/100001/transactions')
        .reply(200, mockTransactions);

      const transactions = await client.getSupporterTransactions('100001');

      expect(transactions).toHaveLength(2);
      expect(transactions[0].amount).toBe(50.0);
      expect(transactions[1].transactionId).toBe('txn-002');
    });

    it('should return empty array for supporter with no transactions', async () => {
      nock(BASE_URL)
        .get('/supporter/100002/transactions')
        .reply(200, []);

      const transactions = await client.getSupporterTransactions('100002');

      expect(transactions).toHaveLength(0);
    });
  });

  describe('processPage', () => {
    it('should process page data successfully', async () => {
      const processData = {
        supporter: {
          firstName: 'Jane',
          lastName: 'Doe',
          emailAddress: 'jane@example.org',
        },
        transaction: {
          amount: 100.0,
          currency: 'USD',
        },
      };

      const mockResponse = {
        status: 'success',
        supporterId: '100001',
        transactionId: 'txn-003',
      };

      nock(BASE_URL)
        .post('/page/12345/process', processData)
        .reply(200, mockResponse);

      const result = await client.processPage('12345', processData);

      expect(result.status).toBe('success');
      expect(result.supporterId).toBe('100001');
    });

    it('should handle validation errors on process', async () => {
      const invalidData = {
        supporter: { emailAddress: '' },
      };

      nock(BASE_URL)
        .post('/page/12345/process', invalidData)
        .reply(400, { error: 'Validation failed', fields: ['emailAddress'] });

      await expect(client.processPage('12345', invalidData)).rejects.toThrow();
    });
  });
});
```

### Python Example

**tests/test_en_client.py:**
```python
import pytest
import responses
from src.en_client import ENClient, ENClientConfig, RateLimitError

BASE_URL = 'https://us.engagingnetworks.app/ens/service'

@pytest.fixture
def client():
    return ENClient(ENClientConfig(
        api_token='test-token',
        region='us',
    ))

@responses.activate
def test_get_pages_success(client):
    """Test successful page fetch"""
    mock_pages = [
        {
            'id': '12345',
            'name': 'Test Page',
            'type': 'donation',
            'status': 'live',
            'url': 'https://example.org/donate',
        }
    ]

    responses.add(
        responses.GET,
        f'{BASE_URL}/page',
        json=mock_pages,
        status=200,
    )

    pages = client.get_pages(page_type='donation', status='live')

    assert len(pages) == 1
    assert pages[0]['name'] == 'Test Page'

@responses.activate
def test_get_pages_empty(client):
    """Test empty results"""
    responses.add(
        responses.GET,
        f'{BASE_URL}/page',
        json=[],
        status=200,
    )

    pages = client.get_pages()

    assert len(pages) == 0

@responses.activate
def test_rate_limit_retry(client):
    """Test retry on rate limit"""
    # First request returns 429
    responses.add(
        responses.GET,
        f'{BASE_URL}/page',
        json={'error': 'Rate limit exceeded'},
        status=429,
    )
    # Second request succeeds
    responses.add(
        responses.GET,
        f'{BASE_URL}/page',
        json=[{'id': '123', 'name': 'Test'}],
        status=200,
    )

    pages = client.get_pages()

    assert len(pages) == 1
    assert len(responses.calls) == 2  # Verify retry happened

@responses.activate
def test_unauthorized_error(client):
    """Test 401 unauthorized"""
    responses.add(
        responses.GET,
        f'{BASE_URL}/page',
        json={'error': 'Unauthorized'},
        status=401,
    )

    with pytest.raises(Exception):
        client.get_pages()
```

## Step 3: Integration Tests (Real EN API)

**tests/integration/en-integration.test.ts:**
```typescript
import { ENClient } from '../../src/lib/en-client';
import * as dotenv from 'dotenv';

dotenv.config();

describe('EN API Integration Tests', () => {
  let client: ENClient;

  beforeAll(() => {
    // Skip if no token (e.g., in CI)
    if (!process.env.EN_API_TOKEN) {
      console.log('Skipping integration tests: No EN_API_TOKEN');
      return;
    }

    client = new ENClient({
      apiToken: process.env.EN_API_TOKEN!,
      region: (process.env.EN_REGION as any) || 'us',
    });
  });

  it('should connect to EN API', async () => {
    const pages = await client.getPages({ limit: 1 });
    expect(Array.isArray(pages)).toBe(true);
  }, 30000); // 30s timeout

  it('should fetch page details', async () => {
    // First get a page ID
    const pages = await client.getPages({ limit: 1 });
    expect(pages.length).toBeGreaterThan(0);

    const pageId = pages[0].id;
    const page = await client.getPage(pageId);

    expect(page.id).toBe(pageId);
    expect(page).toHaveProperty('formFields');
  }, 30000);

  it('should handle pagination', async () => {
    const page1 = await client.getPages({ limit: 5, offset: 0 });
    const page2 = await client.getPages({ limit: 5, offset: 5 });

    // If there are more than 5 pages, they should be different
    if (page1.length === 5 && page2.length > 0) {
      expect(page1[0].id).not.toBe(page2[0].id);
    }
  }, 30000);
});
```

Run with:
```bash
# Only run integration tests when explicitly requested
npm test -- --testPathPattern=integration
```

## Step 4: Error Handling Tests

```typescript
describe('Error Handling', () => {
  it('should handle network timeouts', async () => {
    const slowClient = new ENClient({
      apiToken: process.env.EN_API_TOKEN!,
      region: 'us',
      timeout: 1, // 1ms timeout (will always fail)
    });

    await expect(slowClient.getPages()).rejects.toThrow(/timeout/i);
  });

  it('should handle invalid page ID', async () => {
    await expect(client.getPage('invalid-id-12345')).rejects.toThrow();
  });

  it('should throw after max retries', async () => {
    // Mock server that always returns 500
    nock(BASE_URL)
      .get('/page')
      .times(3) // Match maxRetries
      .reply(500, { error: 'Server error' });

    await expect(client.getPages()).rejects.toThrow();
  });
});
```

## Step 5: Performance Tests

```typescript
describe('Performance', () => {
  it('should cache page details', async () => {
    const cachedClient = new CachedENClient({
      apiToken: process.env.EN_API_TOKEN!,
      region: 'us',
    });

    const pages = await cachedClient.getPages({ limit: 1 });
    const pageId = pages[0].id;

    // First call (cache miss)
    const start1 = Date.now();
    await cachedClient.getPage(pageId);
    const time1 = Date.now() - start1;

    // Second call (cache hit)
    const start2 = Date.now();
    await cachedClient.getPage(pageId);
    const time2 = Date.now() - start2;

    // Cache hit should be significantly faster (>90% faster)
    expect(time2).toBeLessThan(time1 * 0.1);
  });

  it('should handle concurrent requests', async () => {
    const promises = Array.from({ length: 10 }, () =>
      client.getPages({ limit: 1 })
    );

    const results = await Promise.all(promises);

    expect(results).toHaveLength(10);
    results.forEach(pages => {
      expect(Array.isArray(pages)).toBe(true);
    });
  }, 60000);
});
```

## Step 6: Run Tests

### TypeScript

```bash
# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run only unit tests
npm test -- --testPathIgnorePatterns=integration

# Run only integration tests
npm test -- --testPathPattern=integration

# Watch mode for development
npm test -- --watch
```

### Python

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=src --cov-report=html

# Run only unit tests
pytest tests/test_*.py

# Run only integration tests
pytest tests/integration/

# Verbose output
pytest -v
```

## Step 7: CI/CD Integration

**GitHub Actions (.github/workflows/test.yml):**
```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm test -- --testPathIgnorePatterns=integration

      - name: Run integration tests
        if: github.ref == 'refs/heads/main'
        env:
          EN_API_TOKEN: ${{ secrets.EN_API_TOKEN }}
          EN_REGION: us
        run: npm test -- --testPathPattern=integration

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

</process>

<anti_patterns>
Avoid:
- **No mocking in unit tests** - Don't hit real API in unit tests
- **Hardcoded test data** - Use fixtures or factories
- **Integration tests in CI** - Too slow and requires secrets
- **No timeout on integration tests** - Can hang forever
- **Testing implementation details** - Test behavior, not internals
- **No coverage threshold** - Aim for > 80% coverage
- **Ignoring flaky tests** - Fix them, don't skip them
</anti_patterns>

<success_criteria>
Well-tested integration when:
- ✓ Unit tests cover all methods
- ✓ Error handling tested (401, 429, 500, etc.)
- ✓ Retry logic verified
- ✓ Integration tests work with real API
- ✓ Performance tests validate caching
- ✓ CI runs unit tests automatically
- ✓ Code coverage > 80%
- ✓ All tests pass reliably
</success_criteria>
