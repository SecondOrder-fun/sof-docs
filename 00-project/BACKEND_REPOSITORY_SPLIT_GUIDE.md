# Backend Repository Split Guide

## Overview

This guide documents the process of splitting the SecondOrder.fun backend into a separate repository to enable independent builds and deployments.

## Current Structure

The monorepo currently contains:

- **Frontend**: React/Vite application (`src/`, `public/`, `index.html`)
- **Backend**: Fastify server (`backend/`)
- **Contracts**: Solidity smart contracts (`contracts/`)
- **Shared**: Scripts and configuration

## New Repository Structure

### Frontend Repository (sof-alpha)

```text
sof-alpha/
├── src/                    # React components and hooks
├── public/                 # Static assets
├── contracts/              # Smart contracts (read-only for ABIs)
├── scripts/                # Frontend-specific scripts
├── tests/                  # Frontend tests
├── docs/                   # Documentation
├── instructions/           # Project guidelines
├── .windsurf/             # Windsurf rules
├── index.html
├── package.json
├── vite.config.js
└── README.md
```

### Backend Repository (sof-backend) - NEW

```
sof-backend/
├── src/
│   ├── listeners/         # Event listeners
│   ├── services/          # Business logic
│   ├── lib/              # Utilities (viem, etc.)
│   └── db/               # Database migrations
├── fastify/
│   ├── routes/           # API routes
│   └── server.js         # Server entry point
├── shared/               # Shared utilities
│   ├── auth.js
│   ├── redisClient.js
│   └── supabaseClient.js
├── abis/                 # Contract ABIs (copied from contracts)
├── tests/                # Backend tests
├── scripts/              # Backend-specific scripts
├── .env.example
├── package.json
└── README.md
```

## Migration Steps

### Phase 1: Create Backend Repository

1. **Initialize new repository**

   ```bash
   mkdir sof-backend
   cd sof-backend
   git init
   ```

2. **Copy backend files**

   ```bash
   # From sof-alpha root
   cp -r backend/* sof-backend/
   cp -r backend/shared sof-backend/
   ```

3. **Copy necessary configuration**

   ```bash
   cp .env.example sof-backend/
   cp .gitignore sof-backend/
   ```

4. **Copy backend tests**
   ```bash
   mkdir -p sof-backend/tests
   cp -r tests/backend/* sof-backend/tests/
   cp -r tests/api/* sof-backend/tests/api/
   ```

### Phase 2: Create Backend package.json

Create `sof-backend/package.json`:

```json
{
  "name": "sof-backend",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "node -r dotenv/config fastify/server.js",
    "dev:local": "node scripts/start-backend-with-redis.js",
    "start": "node -r dotenv/config fastify/server.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint . --max-warnings 0",
    "reset:local-db": "node scripts/reset-local-db.js",
    "scan:historical": "node scripts/scan-historical-events.js",
    "copy-abis": "node scripts/copy-abis-from-contracts.js"
  },
  "dependencies": {
    "@fastify/cors": "^11.1.0",
    "@fastify/helmet": "^13.0.1",
    "@fastify/rate-limit": "^10.3.0",
    "@supabase/supabase-js": "^2.53.1",
    "axios": "^1.12.2",
    "dotenv": "^17.2.1",
    "fastify": "^5.4.0",
    "ioredis": "^5.8.0",
    "jsonwebtoken": "^9.0.2",
    "viem": "^2.33.3",
    "ws": "^8.18.3"
  },
  "devDependencies": {
    "eslint": "^8.56.0",
    "vitest": "^1.0.0"
  }
}
```

### Phase 3: Update Backend Configuration

1. **Create backend-specific .env.example**

   ```bash
   # Remove frontend-specific variables
   # Keep only backend variables:
   # - Database URLs
   # - Redis configuration
   # - Contract addresses
   # - RPC URLs
   # - Private keys
   ```

2. **Update import paths**

   - Change `backend/` imports to relative paths
   - Update ABI imports to use local `abis/` directory

3. **Create ABI sync script**
   Create `sof-backend/scripts/copy-abis-from-contracts.js` to sync ABIs from contracts repo

### Phase 4: Update Frontend Repository

1. **Remove backend files**

   ```bash
   rm -rf backend/
   ```

2. **Update package.json**

   - Remove backend-specific scripts
   - Remove backend-only dependencies (fastify, ioredis, etc.)
   - Keep only frontend dependencies

3. **Update scripts**

   - Remove `dev:backend`, `dev:backend:local`, `start:backend`
   - Keep `dev`, `build`, `preview` for frontend

4. **Update .gitignore**
   - Remove backend-specific ignores if any

### Phase 5: Setup Communication

1. **Backend API URL Configuration**

   Frontend `.env`:

   ```bash
   VITE_BACKEND_URL=http://localhost:3000
   VITE_BACKEND_URL_PRODUCTION=https://api.secondorder.fun
   ```

2. **CORS Configuration**

   Backend `fastify/server.js`:

   ```javascript
   fastify.register(cors, {
     origin: [
       "http://localhost:5173", // Local frontend
       "https://secondorder.fun", // Production frontend
     ],
     credentials: true,
   });
   ```

3. **Environment-based URLs**

   Frontend API client:

   ```javascript
   const API_URL = import.meta.env.VITE_BACKEND_URL || "http://localhost:3000";
   ```

## Deployment Strategy

### Development

1. **Start Backend**

   ```bash
   cd sof-backend
   npm install
   npm run dev
   ```

2. **Start Frontend**
   ```bash
   cd sof-alpha
   npm install
   npm run dev
   ```

### Production

#### Backend (Railway/Render)

- Deploy from `sof-backend` repository
- Set environment variables in platform
- Auto-deploy on push to `main` branch

#### Frontend (Vercel/Netlify)

- Deploy from `sof-alpha` repository
- Set `VITE_BACKEND_URL` to production backend URL
- Auto-deploy on push to `main` branch

## Benefits

✅ **Independent Deployments**: Frontend changes don't trigger backend rebuilds
✅ **Faster CI/CD**: Smaller repositories = faster builds
✅ **Clear Separation**: Better code organization
✅ **Scalability**: Can scale frontend/backend independently
✅ **Team Workflow**: Different teams can work on different repos

## Shared Dependencies

### Contract ABIs

- **Source**: Contracts repository
- **Backend**: Copies ABIs via script
- **Frontend**: Copies ABIs via script
- **Sync**: Run `npm run copy-abis` after contract changes

### Environment Variables

- **Shared**: Contract addresses, RPC URLs
- **Backend-only**: Database URLs, Redis, private keys
- **Frontend-only**: Vite variables (VITE\_\*)

## Migration Checklist

### Backend Repository Setup

- [ ] Create new repository
- [ ] Copy backend files
- [ ] Create package.json with backend dependencies
- [ ] Setup .env.example
- [ ] Copy backend tests
- [ ] Create ABI sync script
- [ ] Update import paths
- [ ] Test locally

### Frontend Repository Updates

- [ ] Remove backend directory
- [ ] Update package.json (remove backend deps)
- [ ] Update scripts (remove backend scripts)
- [ ] Add VITE_BACKEND_URL configuration
- [ ] Update API client to use environment URL
- [ ] Test locally with backend running

### Deployment Setup

- [ ] Deploy backend to hosting platform
- [ ] Configure backend environment variables
- [ ] Deploy frontend to hosting platform
- [ ] Configure frontend environment variables
- [ ] Test production deployment
- [ ] Setup CI/CD for both repositories

### Documentation

- [ ] Update README in both repositories
- [ ] Document API endpoints
- [ ] Document environment variables
- [ ] Create deployment guides
- [ ] Update contribution guidelines

## Rollback Plan

If issues arise, you can temporarily:

1. Keep both repositories
2. Use git submodules to reference backend in frontend
3. Gradually migrate traffic to new setup

## Next Steps

1. Review this guide with team
2. Create backend repository
3. Test locally with both repositories
4. Deploy to staging environment
5. Monitor and validate
6. Deploy to production

## Support

For questions or issues during migration:

- Check this guide
- Review deployment logs
- Test API connectivity
- Verify environment variables
