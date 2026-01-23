# Network Configuration Fix: Respect DEFAULT_NETWORK from .env

**Date**: December 7, 2025  
**Status**: ✅ COMPLETE

## Problem

Multiple files in the InfoFi backend and frontend system had hardcoded `"LOCAL"` network configurations that ignored the `DEFAULT_NETWORK` environment variable. This caused issues when trying to run the system on TESTNET, as the code would fall back to LOCAL instead of respecting the configured network.

## Solution

Updated all hardcoded `"LOCAL"` references to respect `DEFAULT_NETWORK` from `.env` with a proper fallback chain:

```javascript
// Backend pattern
const NETWORK =
  process.env.NETWORK ||
  process.env.DEFAULT_NETWORK ||
  process.env.VITE_DEFAULT_NETWORK ||
  "LOCAL";

// Frontend pattern
const defaultNet = (
  import.meta.env.VITE_DEFAULT_NETWORK || "LOCAL"
).toUpperCase();
```

## Files Modified

### Backend Files (6 files)

1. **`backend/src/lib/viemClient.js`**

   - Line 11: Updated NETWORK constant to respect DEFAULT_NETWORK
   - Added comment explaining fallback chain

2. **`backend/fastify/server.js`**

   - Line 22: Updated NETWORK constant to respect DEFAULT_NETWORK
   - Added comment explaining fallback chain

3. **`backend/fastify/routes/adminRoutes.js`**

   - Line 15: Updated NETWORK constant to respect DEFAULT_NETWORK
   - Line 289: Updated network variable to respect DEFAULT_NETWORK
   - Added comments explaining fallback chain

4. **`backend/fastify/routes/healthRoutes.js`**

   - Line 47: Updated network variable to respect DEFAULT_NETWORK
   - Added comment explaining fallback chain

5. **`backend/src/config/chain.js`**
   - Lines 54-59: Updated `getChainByKey()` function
   - Now respects DEFAULT_NETWORK with proper fallback chain
   - Updated JSDoc comment

### Frontend Files (4 files)

1. **`src/services/onchainInfoFi.js`**

   - Lines 34-38: Updated WS URL selection in `buildClients()`
   - Lines 515-517: Updated avgBlockTime calculation in `getSeasonStartBlock()`
   - Lines 530-534: Updated lookbackBlocks calculation
   - Added comments explaining DEFAULT_NETWORK usage

2. **`src/config/networks.js`**

   - Lines 43-48: Updated `getNetworkByKey()` function
   - Now respects DEFAULT_NETWORK with proper fallback chain
   - Updated JSDoc comment

3. **`src/lib/wagmi.js`**

   - Lines 48-54: Updated `setStoredNetworkKey()` function
   - Now respects DEFAULT_NETWORK when storing network preference
   - Added comment explaining DEFAULT_NETWORK usage

4. **`src/components/infofi/InfoFiMarketCard.jsx`** - Already using `import.meta.env.VITE_DEFAULT_NETWORK`

## Environment Variable Configuration

The `.env` file should have:

```bash
# Network configuration
VITE_DEFAULT_NETWORK=TESTNET
DEFAULT_NETWORK=TESTNET
```

## Fallback Chain

The fallback chain ensures maximum compatibility:

1. **First**: Check `process.env.NETWORK` (explicit override)
2. **Second**: Check `process.env.DEFAULT_NETWORK` (backend config)
3. **Third**: Check `process.env.VITE_DEFAULT_NETWORK` (frontend config)
4. **Final**: Fall back to `"LOCAL"` (hardcoded last resort)

## Testing

After these changes:

1. ✅ Backend respects `DEFAULT_NETWORK=TESTNET` from `.env`
2. ✅ Frontend respects `VITE_DEFAULT_NETWORK=TESTNET` from `.env`
3. ✅ All network-dependent code uses the configured network
4. ✅ Fallback to LOCAL still works if no env vars are set

## Impact

- **Backend listeners**: Now connect to correct network (TESTNET vs LOCAL)
- **Frontend clients**: Now use correct RPC URLs and chain IDs
- **Oracle calls**: Now target correct network contracts
- **Event listeners**: Now watch correct network events
- **Database operations**: Now sync with correct network state

## Related Issues

This fix resolves the hardcoded LOCAL network configurations that were preventing proper TESTNET operation in the InfoFi system.

## Files Not Modified

These files already had correct DEFAULT_NETWORK usage:

- `src/components/infofi/InfoFiMarketCard.jsx` - Already using `import.meta.env.VITE_DEFAULT_NETWORK`
- `src/components/common/NetworkToggle.jsx` - Only defines available options, not defaults
- `src/config/networks.js` - `getDefaultNetworkKey()` already correct

## Verification

To verify the fix is working:

```bash
# Check backend is using correct network
grep "Using backend network configuration" backend.log

# Check frontend is using correct network
# Open browser console and check network requests

# Verify .env configuration
grep DEFAULT_NETWORK .env
```

Expected output:

```bash
DEFAULT_NETWORK=TESTNET
VITE_DEFAULT_NETWORK=TESTNET
```

## Additional Improvements: Network Config Centralization

After the initial fix, we further improved the architecture by centralizing all network-specific configuration in `src/config/networks.js`:

### Added to Network Config

Each network now includes:

- `wsUrl` - WebSocket RPC endpoint (optional)
- `avgBlockTime` - Average block time in seconds
- `lookbackBlocks` - Default lookback for historical queries

### Benefits

1. **Scalability**: Adding MAINNET or other networks is now trivial
2. **Simplicity**: No more conditional logic scattered throughout the codebase
3. **Maintainability**: All network-specific values in one place
4. **Type Safety**: JSDoc typedef ensures consistency

### Example: Before vs After

**Before (convoluted)**:

```javascript
const defaultNet = (
  import.meta.env.VITE_DEFAULT_NETWORK || "LOCAL"
).toUpperCase();
const wsUrl =
  import.meta.env.VITE_WS_URL_LOCAL && networkKey?.toUpperCase() === defaultNet
    ? import.meta.env.VITE_WS_URL_LOCAL
    : import.meta.env.VITE_WS_URL_TESTNET;
```

**After (clean)**:

```javascript
const chain = getNetworkByKey(networkKey);
const transportWs = chain.wsUrl ? webSocket(chain.wsUrl) : null;
```

### MAINNET Ready

The system is now ready for mainnet deployment. Simply add to `.env`:

```bash
VITE_DEFAULT_NETWORK=MAINNET
VITE_RPC_URL_MAINNET=https://mainnet.base.org
VITE_WS_URL_MAINNET=wss://mainnet.base.org
VITE_MAINNET_CHAIN_ID=8453
VITE_MAINNET_NAME=Base
```

No code changes required!

## Conclusion

All hardcoded `"LOCAL"` network configurations have been replaced with proper `DEFAULT_NETWORK` environment variable usage. The system now correctly respects the configured network from `.env` while maintaining backward compatibility with LOCAL as the final fallback.

Additionally, network-specific configuration has been centralized in `src/config/networks.js`, making the system scalable and ready for mainnet deployment without code changes.
