# Stale Contract Address Validator Fix

## Problem

The Contract Address Validator was showing stale contract addresses despite the `.env` file containing correct values:

**Validator showed (stale):**

- Season Factory: `0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6`
- SOF Faucet: `0xc5a5C42992dECbae36851359345FE25997F5C42d`
- InfoFi Factory: `0x68B1D87F95878fE05B998F19b66F4baba5De1aed`

**.env contains (correct):**

- Season Factory: `0x0165878A594ca255338adfa4d48449f69242Eb8F`
- SOF Faucet: `0x4A679253410272dd5232B3Ff7cF5dbB88f295319`
- InfoFi Factory: `0x0B306BF915C4d645ff596e518fAf3F9669b97016`

## Root Cause

**Vite caches environment variables at dev server start time.** When `npm run anvil:deploy` updates `.env` with new contract addresses, Vite doesn't automatically reload them. The validator was reading from `import.meta.env` which contained the old cached values.

The flow was:

1. Vite dev server starts → reads `.env` → caches values in `import.meta.env`
2. `npm run anvil:deploy` runs → updates `.env` with new addresses
3. Validator reads `import.meta.env` → gets OLD cached values
4. User sees stale addresses

## Solution

Updated `src/components/dev/ContractAddressValidator.jsx` to:

1. **Fetch fresh addresses from backend** instead of reading from Vite's cached `import.meta.env`
2. **Call `/api/health` endpoint** which reads `.env` at runtime (not cached by Vite)
3. **Validate against fresh addresses** before checking contract bytecode

### Key Changes

**Before:**

```javascript
const contracts = getContractAddresses(netKey);  // ❌ Reads from Vite's cached import.meta.env
```

**After:**

```javascript
const contracts = await fetchBackendContracts();  // ✅ Fetches from backend at runtime
```

The `fetchBackendContracts()` function:

- Calls `/api/health` endpoint
- Backend reads fresh `.env` values (not cached)
- Returns current contract addresses
- Validator uses these fresh values for validation

## How to Test

1. **Restart Vite dev server** (Ctrl+C, then `npm run dev`)
2. **Run deployment script**: `npm run anvil:deploy`
3. **Check validator** in browser dev console
4. **Should now show correct addresses** from `.env`

## Why This Works

- **Backend reads `.env` at runtime** - not cached by Vite
- **Frontend fetches from backend** - gets fresh values every validation
- **No Vite restart needed** - validator auto-refreshes every 5 seconds
- **Automatic detection** - if any address is wrong, validator shows alert

## Files Modified

- `src/components/dev/ContractAddressValidator.jsx` - Now fetches from backend instead of Vite cache

## Prevention

This fix is permanent because:

- ✅ Validator always fetches fresh addresses from backend
- ✅ No dependency on Vite's cached `import.meta.env`
- ✅ Works even if Vite cache is stale
- ✅ Auto-validates every 5 seconds

If you deploy new contracts, the validator will automatically show the new addresses without any manual intervention.
