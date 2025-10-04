# Redis Setup: Local Development to Production Guide

## Overview

This guide covers setting up Redis for local development and transitioning to production deployment. It includes configurations for:

- **Local Development**: Redis via Homebrew
- **Production**: Upstash Redis (serverless)
- **Windsurf MCP Integration**: For both environments

## Part 1: Local Development Setup

### 1.1 Install Redis with Homebrew (macOS)

```bash
# Install Redis
brew install redis

# Start Redis as a background service
brew services start redis

# Verify Redis is running
redis-cli ping
# Expected output: PONG

# Check Redis info
redis-cli info server
```

Redis is now running on: `redis://localhost:6379`

### 1.2 Basic Redis Commands

```bash
# Connect to Redis CLI
redis-cli

# Set a key-value pair
SET wallet:0x123 alice.eth

# Get a value
GET wallet:0x123

# List all keys
KEYS *

# Exit
exit
```

### 1.3 Configure Windsurf MCP for Local Redis

Find your uvx path:

```bash
which uvx
# Example output: /Users/yourname/.local/bin/uvx
```

Edit MCP configuration:

- **macOS**: `~/.codeium/windsurf/mcp_config.json`
- **Windows**: `%USERPROFILE%\.codeium\windsurf\mcp_config.json`

Local Development Configuration:

```json
{
  "mcpServers": {
    "redis-mcp-server": {
        "type": "stdio",
        "command": "/Users/yourname/.local/bin/uvx",
        "args": [
            "--from", "redis-mcp-server@latest",
            "redis-mcp-server",
            "--url", "redis://localhost:6379/0"
        ]
    }
  }
}
```

Refresh Windsurf:

1. Save the config file
2. Go to Settings > Cascade > MCP Servers
3. Click Refresh

### 1.4 Test MCP Integration

In Windsurf Cascade:

```text
Store wallet address "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb" with username "alice.eth"
```

Then verify:

```text
Get the username for wallet "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"
```

### 1.5 Managing Redis Service

```bash
# Stop Redis
brew services stop redis

# Restart Redis
brew services restart redis

# Check Redis status
brew services list | grep redis

# View Redis logs
tail -f /usr/local/var/log/redis.log
```

## Part 2: Production Setup with Upstash

### 2.1 Create Upstash Redis Database

1. Go to [console.upstash.com](https://console.upstash.com)
2. Sign up or log in
3. Click "Create Database"
4. Configure:
   - **Name**: your-dapp-production
   - **Type**: Regional or Global
   - **Primary Region**: Choose closest to your users
   - **TLS**: Enabled by default ✓
5. Click "Create"

### 2.2 Get Connection Credentials

From the Upstash dashboard, copy:

- **Endpoint**: `precise-fish-12345.upstash.io`
- **Port**: `6379` (or `6380`)
- **Password**: Click the copy button

Connection URL format:

```text
rediss://default:YOUR_PASSWORD@YOUR_ENDPOINT.upstash.io:PORT
```

Example:

```text
rediss://default:AaBbCc123xyz@precise-fish-12345.upstash.io:6379
```

⚠️ **Note**: Use `rediss://` (with double 's') for TLS/SSL

### 2.3 Configure Windsurf MCP for Production

Update `mcp_config.json`:

```json
{
  "mcpServers": {
    "redis-mcp-server": {
        "type": "stdio",
        "command": "/Users/yourname/.local/bin/uvx",
        "args": [
            "--from", "redis-mcp-server@latest",
            "redis-mcp-server",
            "--url", "rediss://default:YOUR_UPSTASH_PASSWORD@YOUR_ENDPOINT.upstash.io:6379"
        ]
    }
  }
}
```

Refresh Windsurf to apply changes.

## Part 3: Application Code - Environment-Based Configuration

### 3.1 Setup Environment Variables

Create `.env.local` (for local development):

```env
REDIS_URL=redis://localhost:6379
REDIS_TOKEN=
```

Create `.env.production` (for production):

```env
REDIS_URL=rediss://default:YOUR_PASSWORD@YOUR_ENDPOINT.upstash.io:6379
REDIS_TOKEN=YOUR_UPSTASH_REST_TOKEN
```

Add to `.gitignore`:

```text
.env.local
.env.production
```

### 3.2 Application Code (Next.js/React Example)

#### Option A: Using Standard Redis Client (ioredis)

Install:

```bash
npm install ioredis
```

Code:

```javascript
// lib/redis.js
import Redis from 'ioredis';

const getRedisClient = () => {
  const redisUrl = process.env.REDIS_URL || 'redis://localhost:6379';
  
  return new Redis(redisUrl, {
    // Enable TLS for production
    tls: redisUrl.startsWith('rediss://') ? {} : undefined,
    maxRetriesPerRequest: 3,
    retryStrategy(times) {
      const delay = Math.min(times * 50, 2000);
      return delay;
    }
  });
};

export const redis = getRedisClient();
```

#### Option B: Using Upstash REST SDK (Serverless-Friendly)

Install:

```bash
npm install @upstash/redis
```

Code:

```javascript
// lib/redis.js
import { Redis } from '@upstash/redis';

export const redis = process.env.REDIS_TOKEN
  ? new Redis({
      url: process.env.REDIS_URL,
      token: process.env.REDIS_TOKEN,
    })
  : new Redis({
      url: process.env.REDIS_URL || 'redis://localhost:6379',
    });
```

### 3.3 Usage Example

```javascript
// app/api/wallet/route.js
import { redis } from '@/lib/redis';
import { NextResponse } from 'next/server';

export async function POST(request) {
  const { walletAddress, username } = await request.json();
  
  // Store wallet -> username mapping
  await redis.set(`wallet:${walletAddress}`, username);
  
  return NextResponse.json({ 
    success: true,
    message: 'Username stored' 
  });
}

export async function GET(request) {
  const { searchParams } = new URL(request.url);
  const walletAddress = searchParams.get('address');
  
  // Retrieve username
  const username = await redis.get(`wallet:${walletAddress}`);
  
  return NextResponse.json({ 
    walletAddress,
    username 
  });
}
```

## Part 4: Deployment Configuration

### 4.1 Vercel Deployment

#### Option 1: Environment Variables via Vercel Dashboard

1. Go to your project on Vercel
2. Settings > Environment Variables
3. Add:
   - `REDIS_URL`: Your Upstash connection string
   - `REDIS_TOKEN`: Your Upstash REST token (if using REST SDK)
4. Select environments: Production, Preview, Development

#### Option 2: Using Vercel CLI

```bash
# Set production environment variables
vercel env add REDIS_URL production
# Paste your Upstash URL when prompted

vercel env add REDIS_TOKEN production
# Paste your Upstash token when prompted
```

#### Option 3: Vercel + Upstash Integration

1. Go to Vercel Marketplace
2. Search for "Upstash"
3. Click "Add Integration"
4. Follow the setup wizard
5. Environment variables are automatically added

### 4.2 Other Platforms

#### Netlify

```bash
# Set environment variables
netlify env:set REDIS_URL "rediss://..."
netlify env:set REDIS_TOKEN "..."
```

#### Railway

- Add environment variables in project settings
- Railway also supports direct Redis hosting

#### Docker

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Environment variables will be passed at runtime
ENV REDIS_URL=${REDIS_URL}
ENV REDIS_TOKEN=${REDIS_TOKEN}

CMD ["npm", "start"]
```

Run:

```bash
docker run -e REDIS_URL="rediss://..." -e REDIS_TOKEN="..." your-app
```

## Part 5: Switching Between Environments

### 5.1 Development to Production Checklist

Before deploying to production:

- [ ] Update `.env.production` with Upstash credentials
- [ ] Test connection to Upstash locally:

```bash
REDIS_URL="rediss://..." npm run dev
```

- [ ] Update Windsurf MCP config to Upstash URL
- [ ] Set environment variables in hosting platform
- [ ] Deploy application
- [ ] Verify production Redis connection
- [ ] Migrate any existing data (if needed)

### 5.2 Quick Environment Switch Script

Create `scripts/switch-redis.sh`:

```bash
#!/bin/bash

ENV=$1

if [ "$ENV" == "local" ]; then
  cp .env.local .env
  echo "✓ Switched to LOCAL Redis"
  echo "Redis URL: redis://localhost:6379"
elif [ "$ENV" == "production" ]; then
  cp .env.production .env
  echo "✓ Switched to PRODUCTION Redis (Upstash)"
  echo "Check .env for connection details"
else
  echo "Usage: ./switch-redis.sh [local|production]"
  exit 1
fi
```

Usage:

```bash
chmod +x scripts/switch-redis.sh
./scripts/switch-redis.sh local
./scripts/switch-redis.sh production
```

### 5.3 Windsurf MCP Environment Switching

Create two MCP configs:

**Local Config** (`mcp_config.local.json`):

```json
{
  "mcpServers": {
    "redis-mcp-server": {
        "type": "stdio",
        "command": "/Users/yourname/.local/bin/uvx",
        "args": [
            "--from", "redis-mcp-server@latest",
            "redis-mcp-server",
            "--url", "redis://localhost:6379/0"
        ]
    }
  }
}
```

**Production Config** (`mcp_config.production.json`):

```json
{
  "mcpServers": {
    "redis-mcp-server": {
        "type": "stdio",
        "command": "/Users/yourname/.local/bin/uvx",
        "args": [
            "--from", "redis-mcp-server@latest",
            "redis-mcp-server",
            "--url", "rediss://default:PASSWORD@ENDPOINT.upstash.io:6379"
        ]
    }
  }
}
```

Switch script (`scripts/switch-mcp.sh`):

```bash
#!/bin/bash

ENV=$1
MCP_PATH="$HOME/.codeium/windsurf/mcp_config.json"

if [ "$ENV" == "local" ]; then
  cp mcp_config.local.json "$MCP_PATH"
  echo "✓ Windsurf MCP switched to LOCAL Redis"
elif [ "$ENV" == "production" ]; then
  cp mcp_config.production.json "$MCP_PATH"
  echo "✓ Windsurf MCP switched to PRODUCTION Redis"
else
  echo "Usage: ./switch-mcp.sh [local|production]"
  exit 1
fi

echo "Remember to refresh Windsurf MCP servers!"
```

## Part 6: Data Migration

### 6.1 Export from Local Redis

```bash
# Export all data to file
redis-cli --rdb local_backup.rdb

# Or export specific keys as JSON
redis-cli --json GET wallet:* > wallets_backup.json
```

### 6.2 Import to Upstash

#### Option 1: Using redis-cli

```bash
# Connect to Upstash
redis-cli -h YOUR_ENDPOINT.upstash.io -p 6379 -a YOUR_PASSWORD --tls

# Import data
redis-cli -h YOUR_ENDPOINT.upstash.io -p 6379 -a YOUR_PASSWORD --tls --pipe < data.txt
```

#### Option 2: Using Node.js Script

```javascript
// migrate.js
import Redis from 'ioredis';

const local = new Redis('redis://localhost:6379');
const production = new Redis('rediss://default:PASSWORD@ENDPOINT.upstash.io:6379', {
  tls: {}
});

async function migrate() {
  const keys = await local.keys('*');
  
  for (const key of keys) {
    const value = await local.get(key);
    const ttl = await local.ttl(key);
    
    if (ttl > 0) {
      await production.set(key, value, 'EX', ttl);
    } else {
      await production.set(key, value);
    }
    
    console.log(`Migrated: ${key}`);
  }
  
  console.log(`✓ Migrated ${keys.length} keys`);
  process.exit(0);
}

migrate();
```

Run:

```bash
node migrate.js
```

## Part 7: Troubleshooting

### 7.1 Local Redis Issues

**Redis won't start:**

```bash
# Check if port 6379 is in use
lsof -i :6379

# Restart Redis
brew services restart redis

# Check Redis logs
tail -f /usr/local/var/log/redis.log
```

**Connection refused:**

```bash
# Verify Redis is running
brew services list | grep redis

# Test connection
redis-cli ping
```

### 7.2 Upstash Connection Issues

**SSL/TLS errors:**

- Ensure you're using `rediss://` (not `redis://`)
- Verify your password is correct (no special character encoding issues)

**Authentication failed:**

- Copy password again from Upstash dashboard
- Check for trailing spaces in password

**Timeout errors:**

- Check your network/firewall
- Verify the endpoint URL is correct

### 7.3 MCP Server Issues

**Server not showing in Windsurf:**

```bash
# Check uvx is installed
uvx --version

# Test MCP server manually
uvx --from redis-mcp-server@latest redis-mcp-server --url redis://localhost:6379/0

# Check Windsurf logs (macOS)
tail -f ~/Library/Logs/Codeium/Windsurf/mcp-server-redis-mcp-server.log
```

**Tools not appearing:**

- Refresh MCP servers in Windsurf settings
- Restart Windsurf
- Verify `mcp_config.json` syntax is valid JSON

## Part 8: Best Practices

### 8.1 Security

- ✅ Never commit credentials to version control
- ✅ Use environment variables for all credentials
- ✅ Enable TLS/SSL in production (Upstash has this by default)
- ✅ Use different passwords for dev/staging/prod
- ✅ Rotate credentials periodically
- ✅ Use Redis ACL for fine-grained permissions

### 8.2 Performance

- ✅ Use connection pooling in production
- ✅ Set appropriate TTLs for cached data
- ✅ Monitor memory usage
- ✅ Use Redis pipelining for batch operations
- ✅ Consider Redis clustering for high traffic

### 8.3 Monitoring

**Local Development:**

```bash
# Monitor Redis in real-time
redis-cli monitor

# Check memory usage
redis-cli info memory

# Check connected clients
redis-cli client list
```

**Production (Upstash):**

- Use Upstash dashboard for metrics
- Set up alerts for high memory usage
- Monitor request rates and latency

## Quick Reference

### Commands Cheat Sheet

```bash
# Local Redis
brew services start redis          # Start Redis
brew services stop redis           # Stop Redis
redis-cli ping                     # Test connection
redis-cli                         # Enter CLI

# Windsurf MCP
which uvx                         # Find uvx path
code ~/.codeium/windsurf/mcp_config.json  # Edit config
# Refresh in Windsurf UI after changes

# Environment
cp .env.local .env                # Use local
cp .env.production .env           # Use production
```

### Connection URLs

```bash
# Local
redis://localhost:6379/0

# Upstash (Production)
rediss://default:PASSWORD@ENDPOINT.upstash.io:6379
```

## Support Resources

- **Redis Documentation**: [https://redis.io/docs/](https://redis.io/docs/)
- **Upstash Documentation**: [https://upstash.com/docs/redis](https://upstash.com/docs/redis)
- **Redis MCP GitHub**: [https://github.com/redis/mcp-redis](https://github.com/redis/mcp-redis)
- **Windsurf MCP Docs**: [https://docs.windsurf.com/windsurf/cascade/mcp](https://docs.windsurf.com/windsurf/cascade/mcp)

---

**Last Updated**: 2025-10-04
