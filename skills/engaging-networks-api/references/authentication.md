<overview>
Session-based authentication for Engaging Networks REST API. The API uses a two-step authentication process: first obtain an API User token from your EN account, then exchange it for a session token via POST /authenticate. All subsequent requests use the session token.
</overview>

<authentication_flow>
## Authentication Flow

The EN REST API uses **session-based authentication**:

1. **Obtain API User Token** from EN admin panel (Settings → API Users)
2. **POST to /authenticate** with your API User token
3. **Receive session token** (`ens-auth-token`) in response
4. **Use session token** in all subsequent API calls
5. **Handle expiration** - session tokens expire (typically 1 hour)

### Step 1: Get API User Token

1. Log into EN Admin Panel
   - Navigate to your organization's EN instance
   - URL format: `https://{region}.engagingnetworks.app/`

2. Access API Users
   - Go to **Settings** → **API Users**
   - Create a new API User or select existing one

3. Configure API User
   - Copy the API User token (UUID format)
   - Example: `0b4336a5-23d2-314d-a1aa-2a0a29bff157`
   - Assign appropriate permissions
   - **Whitelist your IP addresses** (required!)

4. Store Securely
   ```bash
   # Add to .env file
   echo "EN_API_USER_TOKEN=0b4336a5-23d2-314d-a1aa-2a0a29bff157" >> .env
   echo "EN_REGION=ca" >> .env
   ```

**Important:** The API User token is NOT used directly in API calls. It's only used to authenticate and get a session token.

### Step 2: Authenticate to Get Session Token

**Endpoint:** `POST /authenticate`

**Request:**
- **Method:** POST
- **URL:** `https://{region}.engagingnetworks.app/ens/service/authenticate`
- **Headers:**
  - `Content-Type: application/json`
  - `Accept: application/json`
- **Body:** A JSON object with `ens-auth-token` key containing your API User token
  ```json
  {"ens-auth-token": "0b4336a5-23d2-314d-a1aa-2a0a29bff157"}
  ```

**Response:**
```json
{
  "ens-auth-token": "xyz789-session-token-here",
  "expires": 3600000
}
```

**Response Fields:**
- `ens-auth-token` (string): Session token to use in subsequent requests
- `expires` (number): Token lifetime in milliseconds (e.g., 3600000 = 1 hour)

**Example with curl:**
```bash
# Authenticate and get session token
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"ens-auth-token":"0b4336a5-23d2-314d-a1aa-2a0a29bff157"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate
```

**Example Response:**
```json
{
  "ens-auth-token": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expires": 3600000
}
```

### Step 3: Use Session Token in Requests

All subsequent API calls must include the session token in the header:

**Header name:** `ens-auth-token`
**Header value:** The session token from authenticate response

```bash
# Extract session token
SESSION_TOKEN=$(curl -s -X POST -H "Content-Type: application/json" \
  -d "{\"ens-auth-token\":\"$EN_API_USER_TOKEN\"}" \
  https://ca.engagingnetworks.app/ens/service/authenticate | jq -r '."ens-auth-token"')

# Use session token in subsequent calls
curl -H "ens-auth-token: $SESSION_TOKEN" \
     -H "Accept: application/json" \
     https://ca.engagingnetworks.app/ens/service/page
```

### Step 4: Handle Token Expiration

Session tokens expire after the time specified in the `expires` field (milliseconds).

**Best practice:** Re-authenticate 1 minute before expiration:
```typescript
const now = Date.now();
const expiryTime = authTime + expiresMs;
const shouldReauth = now >= (expiryTime - 60000); // 1 min buffer

if (shouldReauth) {
  await authenticate();
}
```

### Token Validation: GET /authenticate/{ens-auth-token}

Check whether a session token is still valid without making a real API call:

```bash
curl -H "ens-auth-token: SESSION_TOKEN" \
  "https://ca.engagingnetworks.app/ens/service/authenticate/SESSION_TOKEN"
```

**Response 200:**
```json
{
  "valid": true,
  "ens-auth-token": "session-token-uuid-here"
}
```

Use this to verify a token's validity before starting a long-running operation, rather than discovering it's expired mid-way through.

### Token Retirement: DELETE /authenticate/{ens-auth-token}

Explicitly retire a session token before it naturally expires. Good practice for cleanup after batch jobs:

```bash
curl -X DELETE \
  "https://ca.engagingnetworks.app/ens/service/authenticate/SESSION_TOKEN"
```

**Response 200:** Token retired successfully.

This is especially useful when running short-lived scripts — retire the token when done rather than leaving it active.
</authentication_flow>

<usage>
## Using Session-Based Auth

<typescript_example>
```typescript
import axios from 'axios';
import * as dotenv from 'dotenv';

dotenv.config();

class ENAuthenticatedClient {
  private baseURL: string;
  private apiUserToken: string;
  private sessionToken: string | null = null;
  private sessionExpiry: number | null = null;

  constructor(region: 'us' | 'us2' | 'ca' | 'eu' | 'au', apiUserToken: string) {
    const baseURLs = {
      us: 'https://us.engagingnetworks.app/ens/service',
      us2: 'https://us2.engagingnetworks.app/ens/service',
      ca: 'https://ca.engagingnetworks.app/ens/service',
      eu: 'https://eu.engagingnetworks.app/ens/service',
      au: 'https://au.engagingnetworks.app/ens/service',
    };
    this.baseURL = baseURLs[region];
    this.apiUserToken = apiUserToken;
  }

  async authenticate(): Promise<void> {
    const response = await axios.post<{
      'ens-auth-token': string;
      expires: number;
    }>(
      `${this.baseURL}/authenticate`,
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

    console.log(`✓ Authenticated. Session expires in ${Math.floor(response.data.expires / 60000)} minutes`);
  }

  async ensureAuthenticated(): Promise<void> {
    const now = Date.now();
    const needsAuth = !this.sessionToken ||
                      !this.sessionExpiry ||
                      now >= this.sessionExpiry - 60000;

    if (needsAuth) {
      await this.authenticate();
    }
  }

  async get<T>(endpoint: string, params?: any): Promise<T> {
    await this.ensureAuthenticated();

    const response = await axios.get<T>(`${this.baseURL}${endpoint}`, {
      headers: {
        'ens-auth-token': this.sessionToken!,
        'Accept': 'application/json',
      },
      params,
    });

    return response.data;
  }
}

// Usage
const client = new ENAuthenticatedClient('ca', process.env.EN_API_USER_TOKEN!);
const pages = await client.get('/page', { limit: 100 });
```
</typescript_example>

<python_example>
```python
import requests
import os
import time
import json

class ENAuthenticatedClient:
    def __init__(self, region: str, api_user_token: str):
        base_urls = {
            'us': 'https://us.engagingnetworks.app/ens/service',
            'us2': 'https://us2.engagingnetworks.app/ens/service',
            'ca': 'https://ca.engagingnetworks.app/ens/service',
            'eu': 'https://eu.engagingnetworks.app/ens/service',
            'au': 'https://au.engagingnetworks.app/ens/service',
        }
        self.base_url = base_urls[region]
        self.api_user_token = api_user_token
        self.session_token = None
        self.session_expiry = None

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

        if response.status_code == 401:
            raise Exception('Authentication failed. Check API User token and IP whitelist.')

        response.raise_for_status()
        data = response.json()

        self.session_token = data['ens-auth-token']
        self.session_expiry = time.time() + (data['expires'] / 1000)

        print(f"✓ Authenticated. Session expires in {int(data['expires'] / 60000)} minutes")

    def ensure_authenticated(self):
        """Ensure we have a valid session token"""
        now = time.time()
        needs_auth = (
            not self.session_token or
            not self.session_expiry or
            now >= self.session_expiry - 60  # Re-auth 1 min before expiry
        )

        if needs_auth:
            self.authenticate()

    def get(self, endpoint: str, params=None):
        """Make authenticated GET request"""
        self.ensure_authenticated()

        response = requests.get(
            f'{self.base_url}{endpoint}',
            headers={
                'ens-auth-token': self.session_token,
                'Accept': 'application/json',
            },
            params=params
        )

        response.raise_for_status()
        return response.json()

# Usage
client = ENAuthenticatedClient('ca', os.environ['EN_API_USER_TOKEN'])
pages = client.get('/page', {'limit': 100})
```
</python_example>
</usage>

<security_best_practices>
## Security Guidelines

<environment_variables>
**✅ DO:**
```bash
# .env file - Store API User token, NOT session token
EN_API_USER_TOKEN=0b4336a5-23d2-314d-a1aa-2a0a29bff157
EN_REGION=ca

# .gitignore
.env
.env.*
```

**❌ DON'T:**
```typescript
// Hardcoded API User token (BAD!)
const apiUserToken = '0b4336a5-23d2-314d-a1aa-2a0a29bff157';

// Token in version control (BAD!)
git add .env

// Storing session token permanently (BAD!)
// Session tokens are temporary and should be regenerated
localStorage.setItem('session-token', token);
```
</environment_variables>

<token_types>
**Two Token Types:**

1. **API User Token** (permanent)
   - Obtained from EN Admin → Settings → API Users
   - Format: UUID (e.g., `0b4336a5-23d2-314d-a1aa-2a0a29bff157`)
   - Store in environment variables
   - Used ONLY for POST /authenticate
   - Rotate every 90 days

2. **Session Token** (temporary)
   - Obtained from POST /authenticate response
   - Expires after time period (typically 1 hour)
   - Used for all other API calls
   - Never store permanently - regenerate as needed
   - Automatically refreshed by client when expired
</token_types>

<token_rotation>
**Rotate API User Tokens every 90 days:**
1. Generate new API User token in EN admin
2. Update .env file with new token
3. Deploy with new token
4. Revoke old token after verification
5. Document rotation date
</token_rotation>

<error_handling>
**Never log tokens:**
```typescript
// ❌ BAD: Tokens in logs
console.log('API User Token:', apiUserToken);
console.log('Session Token:', sessionToken);

// ✅ GOOD: Redact tokens
console.log('API User Token:', '***REDACTED***');
console.log('Session Token:', sessionToken ? '***REDACTED***' : 'null');
```

**Never expose client-side:**
```typescript
// ❌ BAD: Tokens in frontend
// frontend/api.js
const apiUserToken = '0b4336a5...';  // Exposed to users!

// ✅ GOOD: Tokens on server only
// backend/api.js
const apiUserToken = process.env.EN_API_USER_TOKEN;
```
</error_handling>

<ip_whitelisting>
**IP Whitelisting is REQUIRED:**
- All API Users must have whitelisted IP addresses
- Configure in EN Admin → Settings → API Users → [Select User] → IP Whitelist
- Authentication will fail (401) if request comes from non-whitelisted IP
- Provide static IP addresses for production servers
- Test from whitelisted IPs before deployment

**Common Issue:**
```
POST /authenticate returns 401 even with valid token
→ Check if your IP address is whitelisted for the API User
```
</ip_whitelisting>

<vault_restriction>
**Payment Processing (`-vault` host) — Do Not Use Through AI Agent:**

The EN API uses a `-vault` host variant (e.g., `ca-vault.engagingnetworks.app`) for processing credit card payments. **Never route payment card data through an AI agent session.** Credit card numbers, CVVs, and expiration dates in an agent's context window create PCI DSS compliance exposure. Process payments through direct server-to-EN connections in your application code only.
</vault_restriction>
</security_best_practices>

<authentication_errors>
## Troubleshooting

<error_401_authenticate>
**HTTP 401 on POST /authenticate**

**Causes:**
- Invalid API User token
- IP address not whitelisted for this API User
- API User disabled or deleted
- Extra spaces/quotes in token (check .env file)

**Solutions:**
```bash
# Test authentication manually
curl -v -X POST \
  -H "Content-Type: application/json" \
  -d '{"ens-auth-token":"YOUR_API_USER_TOKEN"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate

# Check environment variable
echo $EN_API_USER_TOKEN | wc -c  # Should be ~36-37 characters (UUID)

# Verify no extra spaces
printenv EN_API_USER_TOKEN | cat -A  # Should show no $, spaces, or quotes at start/end

# Check IP whitelist in EN Admin
# Settings → API Users → [Your User] → IP Whitelist
# Add your current IP: curl ifconfig.me

# Regenerate token if needed
# EN Admin → Settings → API Users → [Your User] → Regenerate Token
```
</error_401_authenticate>

<error_401_subsequent>
**HTTP 401 on subsequent API calls (after /authenticate)**

**Causes:**
- Session token expired
- Session token not included in header
- Wrong header name (must be `ens-auth-token`)

**Solutions:**
```typescript
// Check token expiration
const now = Date.now();
const isExpired = now >= sessionExpiry;

if (isExpired) {
  // Re-authenticate
  await authenticate();
}

// Verify header format
headers: {
  'ens-auth-token': sessionToken,  // ✓ Correct
  // NOT 'Authorization': `Bearer ${sessionToken}`  // ✗ Wrong
}
```
</error_401_subsequent>

<error_403>
**HTTP 403 Forbidden**

**Causes:**
- API User lacks required permissions for endpoint
- Account access restrictions
- Resource doesn't belong to your account

**Solutions:**
- Check API User permissions in EN admin
- Verify the API User has access to the specific endpoint (page, supporter, etc.)
- Ensure you're querying resources from your own account
- Contact EN support for access issues
</error_403>

<token_expiry>
**Session Token Expiration**

Session tokens expire after the time specified in the `expires` field:
- Typical expiration: 3600000ms (1 hour)
- Tokens expire at: `authTime + expires`

**Best practices:**
1. Store expiration time when authenticating
2. Check expiration before each request
3. Re-authenticate 1 minute before expiry (buffer)
4. Handle 401 errors as potential expiration

**Don't:**
- Store session tokens permanently (they expire!)
- Assume tokens last forever
- Hardcode expiration times (use `expires` from response)
</token_expiry>
</authentication_errors>

<testing_authentication>
## Test Your Authentication

**Step-by-step test:**
```bash
# 1. Test authentication endpoint
curl -v -X POST \
  -H "Content-Type: application/json" \
  -d '{"ens-auth-token":"YOUR_API_USER_TOKEN"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate

# Expected: 200 OK with { "ens-auth-token": "...", "expires": 3600000 }

# 2. Extract session token (requires jq)
SESSION_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"ens-auth-token":"YOUR_API_USER_TOKEN"}' \
  https://ca.engagingnetworks.app/ens/service/authenticate | jq -r '."ens-auth-token"')

# 3. Test session token with API call
curl -H "ens-auth-token: $SESSION_TOKEN" \
     -H "Accept: application/json" \
     https://ca.engagingnetworks.app/ens/service/page?limit=1

# Expected: 200 OK with array of pages
```

**TypeScript test:**
```typescript
// test-auth.ts
import axios from 'axios';
import * as dotenv from 'dotenv';

dotenv.config();

async function testAuth() {
  const baseURL = 'https://ca.engagingnetworks.app/ens/service';
  const apiUserToken = process.env.EN_API_USER_TOKEN!;

  try {
    // Step 1: Authenticate
    console.log('Authenticating...');
    const authResponse = await axios.post<{
      'ens-auth-token': string;
      expires: number;
    }>(
      `${baseURL}/authenticate`,
      { 'ens-auth-token': apiUserToken },
      {
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
      }
    );

    const sessionToken = authResponse.data['ens-auth-token'];
    const expiresMinutes = Math.floor(authResponse.data.expires / 60000);

    console.log(`✓ Authentication successful`);
    console.log(`✓ Session token expires in ${expiresMinutes} minutes`);

    // Step 2: Test API call with session token
    console.log('\nTesting API call...');
    const pagesResponse = await axios.get(
      `${baseURL}/page`,
      {
        headers: {
          'ens-auth-token': sessionToken,
          'Accept': 'application/json',
        },
        params: { limit: 1 },
      }
    );

    console.log(`✓ API call successful`);
    console.log(`✓ Found ${pagesResponse.data.length} page(s)`);
    console.log('\n✓ Authentication and API access verified!');

  } catch (error: any) {
    if (axios.isAxiosError(error)) {
      if (error.response?.status === 401) {
        console.error('✗ Authentication failed (401)');
        console.error('  Check: API User token is valid');
        console.error('  Check: IP address is whitelisted');
        console.error('  Check: No extra quotes/spaces in token');
      } else if (error.response?.status === 403) {
        console.error('✗ Access forbidden (403)');
        console.error('  Check: API User has required permissions');
      } else {
        console.error('✗ Error:', error.message);
        console.error('  Status:', error.response?.status);
        console.error('  Data:', error.response?.data);
      }
    } else {
      console.error('✗ Error:', error);
    }
    process.exit(1);
  }
}

testAuth();
```

**Run:**
```bash
npx ts-node test-auth.ts
```
</testing_authentication>

<multi_environment_setup>
## Multiple Environments

```bash
# .env.development
EN_API_USER_TOKEN=dev_token_here
EN_REGION=ca

# .env.staging
EN_API_USER_TOKEN=staging_token_here
EN_REGION=ca

# .env.production
EN_API_USER_TOKEN=prod_token_here
EN_REGION=ca
```

**Load correct environment:**
```typescript
import * as dotenv from 'dotenv';

const env = process.env.NODE_ENV || 'development';
dotenv.config({ path: `.env.${env}` });

const client = new ENClient({
  apiUserToken: process.env.EN_API_USER_TOKEN!,
  region: process.env.EN_REGION as Region,
});
```
</multi_environment_setup>

<common_issues>
## Common Authentication Issues

### Issue 1: 401 on /authenticate
**Symptom:** POST /authenticate returns 401 Unauthorized

**Checklist:**
- [ ] API User token is correct (copy-paste from EN Admin)
- [ ] Your IP address is whitelisted (EN Admin → API Users → IP Whitelist)
- [ ] No extra quotes in .env file (should be: `EN_API_USER_TOKEN=abc123`, not `EN_API_USER_TOKEN="abc123"`)
- [ ] API User is active (not disabled)
- [ ] Using correct region (ca/us/eu/au)

### Issue 2: Session token expires quickly
**Symptom:** Getting 401 errors after some time

**Solution:** Implement automatic re-authentication:
```typescript
private async ensureAuthenticated(): Promise<void> {
  const now = Date.now();
  const needsAuth = !this.sessionToken ||
                    !this.sessionExpiry ||
                    now >= this.sessionExpiry - 60000; // 1 min buffer

  if (needsAuth) {
    await this.authenticate();
  }
}
```

### Issue 3: Can't authenticate from production server
**Symptom:** Works locally but 401 in production

**Cause:** Production server IP not whitelisted

**Solution:**
1. Get production server IP: `curl ifconfig.me`
2. Add to EN Admin → Settings → API Users → [Your User] → IP Whitelist
3. Test from production: `curl -X POST -H "Content-Type: application/json" -d '{"ens-auth-token":"TOKEN"}' https://ca.engagingnetworks.app/ens/service/authenticate`
</common_issues>
