# Workflow: Deploy EN API Integration

<required_reading>
**Read NOW:**
1. references/deployment.md
2. references/monitoring-logging.md
</required_reading>

<process>
## Step 1: Pre-Deployment Checklist

Verify readiness before deploying:

**Security:**
- [ ] EN_API_TOKEN in environment variables (not code)
- [ ] .env added to .gitignore
- [ ] No tokens in logs or error messages
- [ ] Token rotation plan in place (every 90 days)
- [ ] HTTPS only (no HTTP endpoints)

**Code Quality:**
- [ ] All tests pass
- [ ] Code coverage > 80%
- [ ] No hardcoded values
- [ ] Error handling for all endpoints
- [ ] Retry logic with exponential backoff

**Performance:**
- [ ] Caching implemented for static data
- [ ] Rate limiting handled
- [ ] Pagination efficient
- [ ] No unnecessary API calls

**Monitoring:**
- [ ] Logging configured
- [ ] Error tracking (Sentry, etc.)
- [ ] Metrics collection
- [ ] Alerting set up

## Step 2: Environment Configuration

### Production .env

```bash
# .env.production
EN_API_TOKEN=your_production_token_here
EN_REGION=us  # or ca, eu, au

# Logging
LOG_LEVEL=info  # info, warn, error (not debug in prod)

# Performance
CACHE_TTL=3600  # 1 hour
MAX_RETRIES=3
REQUEST_TIMEOUT=30000  # 30 seconds

# Monitoring (optional)
SENTRY_DSN=https://...
```

**Security:**
- Store in platform secret manager (Vercel, AWS Secrets Manager, etc.)
- Never commit .env.production
- Rotate tokens regularly

## Step 3: Build for Production

### TypeScript

```bash
# Clean previous builds
rm -rf dist/

# Build
npm run build

# Verify build
ls -la dist/

# Test production build
NODE_ENV=production node dist/index.js
```

**package.json scripts:**
```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "start:dev": "ts-node src/index.ts",
    "test": "jest",
    "lint": "eslint src/ --ext .ts"
  }
}
```

### Python

```bash
# Verify dependencies
pip freeze > requirements.txt

# Run in production mode
export ENV=production
python main.py
```

## Step 4: Platform-Specific Deployment

### Vercel (Serverless)

**vercel.json:**
```json
{
  "version": 2,
  "builds": [
    {
      "src": "src/index.ts",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/api/sync",
      "dest": "src/api/sync.ts"
    }
  ],
  "env": {
    "EN_API_TOKEN": "@en-api-token",
    "EN_REGION": "us"
  },
  "crons": [
    {
      "path": "/api/cron/daily-sync",
      "schedule": "0 2 * * *"
    }
  ]
}
```

Deploy:
```bash
# Install Vercel CLI
npm install -g vercel

# Set secrets
vercel secrets add en-api-token your_token_here

# Deploy
vercel --prod
```

### AWS Lambda

**serverless.yml:**
```yaml
service: en-api-integration

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  environment:
    EN_API_TOKEN: ${env:EN_API_TOKEN}
    EN_REGION: ${env:EN_REGION}
    LOG_LEVEL: info

functions:
  syncPages:
    handler: dist/handlers/sync-pages.handler
    events:
      - schedule: cron(0 2 * * ? *)  # Daily at 2 AM UTC
    timeout: 300  # 5 minutes
    memorySize: 512

  apiHandler:
    handler: dist/handlers/api.handler
    events:
      - http:
          path: /sync
          method: post
```

Deploy:
```bash
# Install Serverless Framework
npm install -g serverless

# Deploy
sls deploy --stage production
```

### Docker Container

**Dockerfile:**
```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

ENV NODE_ENV=production

CMD ["node", "dist/index.js"]
```

Build and run:
```bash
# Build
docker build -t en-api-integration:latest .

# Run
docker run -d \
  --name en-api-integration \
  -e EN_API_TOKEN=$EN_API_TOKEN \
  -e EN_REGION=us \
  --restart unless-stopped \
  en-api-integration:latest

# Check logs
docker logs -f en-api-integration
```

### Traditional Server (PM2)

```bash
# Install PM2
npm install -g pm2

# Start
pm2 start dist/index.js --name en-api-integration

# Save configuration
pm2 save

# Auto-start on reboot
pm2 startup

# Monitor
pm2 monit
```

**ecosystem.config.js:**
```javascript
module.exports = {
  apps: [{
    name: 'en-api-integration',
    script: './dist/index.js',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production',
      EN_API_TOKEN: process.env.EN_API_TOKEN,
      EN_REGION: 'us',
    },
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
  }],
};
```

## Step 5: Set Up Monitoring

### Logging

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// In production, also log to console for container logs
if (process.env.NODE_ENV === 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}

// Usage
logger.info('Fetching pages from EN API', { region: 'us' });
logger.error('API request failed', { error: error.message, pageId });
```

### Error Tracking (Sentry)

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});

// Wrap API client
class MonitoredENClient extends ENClient {
  async getPages(params?: any): Promise<ENPage[]> {
    return Sentry.startSpan(
      { name: 'en_api.get_pages', op: 'http.client' },
      async () => {
        try {
          return await super.getPages(params);
        } catch (error) {
          Sentry.captureException(error, {
            extra: { params },
          });
          throw error;
        }
      }
    );
  }
}
```

### Health Check Endpoint

```typescript
// src/api/health.ts
export async function healthCheck(): Promise<any> {
  try {
    // Test EN API connection
    const client = new ENClient({
      apiToken: process.env.EN_API_TOKEN!,
      region: 'us',
    });

    await client.getPages({ limit: 1 });

    return {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {
        en_api: 'connected',
      },
    };
  } catch (error) {
    return {
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      error: error.message,
    };
  }
}
```

### Metrics Collection

```typescript
import { Counter, Histogram } from 'prom-client';

const apiCallsCounter = new Counter({
  name: 'en_api_calls_total',
  help: 'Total number of EN API calls',
  labelNames: ['endpoint', 'status'],
});

const apiLatencyHistogram = new Histogram({
  name: 'en_api_latency_seconds',
  help: 'EN API call latency',
  labelNames: ['endpoint'],
  buckets: [0.1, 0.5, 1, 2, 5],
});

// Track metrics in client
class MetricsENClient extends ENClient {
  async getPages(params?: any): Promise<ENPage[]> {
    const start = Date.now();

    try {
      const result = await super.getPages(params);
      apiCallsCounter.inc({ endpoint: 'getPages', status: 'success' });
      return result;
    } catch (error) {
      apiCallsCounter.inc({ endpoint: 'getPages', status: 'error' });
      throw error;
    } finally {
      const duration = (Date.now() - start) / 1000;
      apiLatencyHistogram.observe({ endpoint: 'getPages' }, duration);
    }
  }
}
```

## Step 6: Set Up Alerts

### CloudWatch Alarms (AWS)

```yaml
alarms:
  - name: HighErrorRate
    metric: Errors
    threshold: 10
    period: 300  # 5 minutes
    evaluationPeriods: 2
    comparisonOperator: GreaterThanThreshold
    actions:
      - !Ref AlertTopic

  - name: HighLatency
    metric: Duration
    threshold: 5000  # 5 seconds
    period: 60
    evaluationPeriods: 3
    comparisonOperator: GreaterThanThreshold
```

### Uptime Monitoring

Use services like UptimeRobot, Pingdom, or custom:

```typescript
// Simple cron job to test health
import nodeCron from 'node-cron';
import axios from 'axios';

// Check health every 5 minutes
nodeCron.schedule('*/5 * * * *', async () => {
  try {
    const response = await axios.get('https://your-api.com/health');

    if (response.data.status !== 'healthy') {
      // Send alert (email, Slack, PagerDuty)
      await sendAlert('EN API integration unhealthy', response.data);
    }
  } catch (error) {
    await sendAlert('EN API integration down', error);
  }
});
```

## Step 7: Document Deployment

Create DEPLOYMENT.md:

```markdown
# Deployment Guide

## Prerequisites
- Node.js 18+
- EN API token
- Production environment variables

## Deploy to Production

1. Set environment variables:
   \`\`\`bash
   export EN_API_TOKEN=xxx
   export EN_REGION=us
   \`\`\`

2. Build:
   \`\`\`bash
   npm run build
   \`\`\`

3. Test production build:
   \`\`\`bash
   NODE_ENV=production node dist/index.js
   \`\`\`

4. Deploy:
   \`\`\`bash
   vercel --prod  # or your deployment command
   \`\`\`

## Monitoring

- Health check: https://your-api.com/health
- Logs: `pm2 logs` or CloudWatch
- Metrics: Prometheus/Grafana dashboard
- Errors: Sentry dashboard

## Rollback

\`\`\`bash
vercel rollback  # or platform-specific command
\`\`\`

## Support

- On-call: [contact info]
- Runbook: [link to runbook]
```

## Step 8: Post-Deployment Verification

```bash
# 1. Test health endpoint
curl https://your-api.com/health

# 2. Test API integration
curl -X POST https://your-api.com/api/sync

# 3. Check logs
# (platform-specific command)

# 4. Monitor metrics for first hour
# Watch for:
# - Error rate
# - Response times
# - API call count
# - Rate limit hits
```

</process>

<anti_patterns>
Avoid:
- **Tokens in code** - Always use environment variables
- **No health check** - Can't monitor uptime
- **No logging** - Can't debug production issues
- **Deploying without testing** - Test in staging first
- **No rollback plan** - Things go wrong
- **Ignoring resource limits** - Set timeouts, memory limits
- **No monitoring** - You won't know when it breaks
- **Hardcoded region** - Make it configurable
</anti_patterns>

<success_criteria>
Successfully deployed when:
- ✓ Application runs in production environment
- ✓ ENV variables configured securely
- ✓ Health check endpoint responds
- ✓ Logs are accessible
- ✓ Monitoring and alerts set up
- ✓ Error tracking configured
- ✓ Performance metrics collected
- ✓ Deployment documented
- ✓ Rollback plan in place
- ✓ Team knows how to access logs and metrics
</success_criteria>
