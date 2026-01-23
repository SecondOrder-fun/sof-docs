# Deployment Guide for Split Repositories

## Backend Deployment (Railway)

### 1. Create Railway Project

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login
railway login

# Initialize project
cd sof-backend
railway init
```

### 2. Configure Environment Variables

In Railway dashboard, add all variables from `BACKEND_ENV_EXAMPLE.md`:

**Required Variables:**

- `PORT=3000`
- `NODE_ENV=production`
- `SUPABASE_URL`
- `SUPABASE_SERVICE_ROLE_KEY`
- `REDIS_HOST`, `REDIS_PORT`
- `RPC_URL_MAINNET`
- `PRIVATE_KEY_MAINNET`
- All contract addresses
- `JWT_SECRET`
- `CORS_ORIGIN=https://secondorder.fun`

### 3. Deploy

```bash
railway up
```

### 4. Get Backend URL

```bash
railway domain
# Example: https://sof-backend-production.up.railway.app
```

## Frontend Deployment (Vercel)

### 1. Install Vercel CLI

```bash
npm install -g vercel
```

### 2. Deploy

```bash
cd sof-alpha
vercel
```

### 3. Configure Environment Variables

In Vercel dashboard:

```bash
VITE_BACKEND_URL=https://your-backend-url.railway.app
VITE_RPC_URL_MAINNET=https://mainnet.base.org
VITE_CHAIN_ID_MAINNET=8453
# Add all VITE_* contract addresses
```

### 4. Redeploy

```bash
vercel --prod
```

## Alternative: Backend on Render

### 1. Create Web Service

- Go to render.com
- New > Web Service
- Connect GitHub repository
- Select `sof-backend` repo

### 2. Configure

**Build Command:** `npm install`
**Start Command:** `npm start`
**Environment:** Node

### 3. Add Environment Variables

Same as Railway configuration above.

## Alternative: Frontend on Netlify

### 1. Deploy

```bash
npm install -g netlify-cli
cd sof-alpha
netlify deploy --prod
```

### 2. Configure

**Build Command:** `npm run build`
**Publish Directory:** `dist`

### 3. Environment Variables

Add in Netlify dashboard under Site Settings > Environment Variables.

## Health Checks

### Backend Health Check

```bash
curl https://your-backend-url/api/health
```

Expected response:

```json
{
  "status": "ok",
  "timestamp": "2024-12-11T08:00:00.000Z",
  "contracts": { ... }
}
```

### Frontend Health Check

Visit: `https://your-frontend-url`

Should load without errors and connect to backend.

## Monitoring

### Backend Logs

**Railway:**

```bash
railway logs
```

**Render:**
Check logs in dashboard

### Frontend Logs

**Vercel:**
Check deployment logs in dashboard

**Netlify:**
Check function logs in dashboard

## Rollback

### Backend

**Railway:**

```bash
railway rollback
```

**Render:**
Use dashboard to rollback to previous deployment

### Frontend

**Vercel:**

```bash
vercel rollback
```

**Netlify:**
Use dashboard to rollback

## CI/CD Setup

### Backend (GitHub Actions)

Create `.github/workflows/deploy-backend.yml`:

```yaml
name: Deploy Backend

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
      - run: npm install
      - run: npm test
      - uses: railway/deploy@v1
        with:
          railway-token: ${{ secrets.RAILWAY_TOKEN }}
```

### Frontend (GitHub Actions)

Create `.github/workflows/deploy-frontend.yml`:

```yaml
name: Deploy Frontend

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
      - run: npm install
      - run: npm run build
      - uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
```

## Cost Estimates

### Railway (Backend)

- Hobby: $5/month
- Pro: $20/month (recommended)

### Vercel (Frontend)

- Hobby: Free
- Pro: $20/month

### Render (Backend Alternative)

- Free tier available
- Starter: $7/month

### Netlify (Frontend Alternative)

- Free tier available
- Pro: $19/month

## Security Checklist

- [ ] All private keys in environment variables (not code)
- [ ] CORS configured with specific origins
- [ ] Rate limiting enabled
- [ ] Helmet security headers enabled
- [ ] JWT secret is strong and unique
- [ ] HTTPS enabled on all endpoints
- [ ] Environment variables not exposed in frontend
- [ ] Database credentials secured
- [ ] Redis password set (if applicable)
