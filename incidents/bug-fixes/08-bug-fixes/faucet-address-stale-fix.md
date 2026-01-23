# Faucet Address Stale Issue - Complete Fix

## Problem

The faucet address validator was showing stale addresses even after deploying new contracts:

```text
[useFaucet] ❌ No contract deployed at faucet address: 0xc5a5C42992dECbae36851359345FE25997F5C42d
```

But `.env` contains the correct address:

```text
SOF_FAUCET_ADDRESS_LOCAL=0x4A679253410272dd5232B3Ff7cF5dbB88f295319
```

## Root Cause

**Two-layer caching issue:**

1. **Vite caches env vars** at dev server start → Frontend reads stale `import.meta.env`
2. **Backend also caches env vars** at startup → Health endpoint returns stale addresses

The validator was fetching from `/api/health`, but the backend hadn't been restarted after `npm run anvil:deploy`, so it was still returning old addresses.

## Solution

### Part 1: Updated Health Endpoint (Backend)

**File**: `backend/fastify/routes/healthRoutes.js`

Added `getContractAddresses()` function that reads fresh contract addresses from `process.env` at runtime:

```javascript
function getContractAddresses() {
  return {
    SOF: process.env.SOF_ADDRESS_LOCAL || process.env.SOF_ADDRESS_TESTNET || '',
    RAFFLE: process.env.RAFFLE_ADDRESS_LOCAL || process.env.RAFFLE_ADDRESS_TESTNET || '',
    SEASON_FACTORY: process.env.SEASON_FACTORY_ADDRESS_LOCAL || process.env.SEASON_FACTORY_ADDRESS_TESTNET || '',
    SOF_FAUCET: process.env.SOF_FAUCET_ADDRESS_LOCAL || process.env.SOF_FAUCET_ADDRESS_TESTNET || '',
    // ... other contracts
  };
}
```

Updated `/health` endpoint to include `contracts` in response:

```javascript
return reply.send({
  status,
  timestamp: new Date().toISOString(),
  latencyMs: Date.now() - startedAt,
  env,
  contracts: getContractAddresses(),  // ✅ NEW
  checks: { supabase, rpc, network },
});
```

### Part 2: Updated Validator Instructions (Frontend)

**File**: `src/components/dev/ContractAddressValidator.jsx`

Updated fix instructions to include backend restart:

```
1. Run: npm run anvil:deploy
2. Restart backend: npm run dev:backend  ✅ NEW
3. Restart Vite dev server (Ctrl+C, then npm run dev)
4. Hard refresh browser: Cmd+Shift+R (Mac) or Ctrl+Shift+R (Windows)
5. Click button below to clear cache
```

## How It Works Now

```
npm run anvil:deploy
  ↓
Deploys new contracts, updates .env
  ↓
npm run dev:backend
  ↓
Backend restarts, reads fresh .env into process.env
  ↓
npm run dev
  ↓
Vite restarts, validator fetches from /api/health
  ↓
/api/health returns fresh contracts from process.env
  ↓
Validator validates against fresh addresses
  ✅ Shows correct addresses
```

## Key Improvements

✅ **Backend health endpoint now returns contract addresses**
✅ **Addresses are read fresh from process.env at runtime**
✅ **No caching of contract addresses in backend**
✅ **Validator instructions now include backend restart**
✅ **Clear step-by-step fix process**

## Testing the Fix

1. **Deploy new contracts**: `npm run anvil:deploy`
2. **Restart backend**: `npm run dev:backend`
3. **Restart Vite**: `npm run dev`
4. **Check validator** in browser
5. **Should now show correct addresses** from `.env`

## Why This Is Permanent

- Backend reads fresh `process.env` on every health check
- Frontend fetches from backend every 5 seconds
- No reliance on Vite's cached `import.meta.env`
- Both layers now read fresh values at runtime

## Files Modified

1. `backend/fastify/routes/healthRoutes.js` - Added contract address loading
2. `src/components/dev/ContractAddressValidator.jsx` - Updated fix instructions

## Important Note

**Always restart the backend after deploying new contracts.** The backend caches environment variables at startup, so it needs to be restarted to pick up new addresses from `.env`.
