# Trade Listener Error Fix

**Date**: Oct 31, 2025  
**Status**: ✅ FIXED

## Error Message

```json
{"level":50,"time":1761898276152,"pid":21913,"hostname":"Alpeia.local","msg":"❌ Failed to start Trade listeners: logger instance is required"}
```

## Root Cause

The `startTradeListener` function expects **3 parameters**:

1. `fpmmAddresses` (array of contract addresses)
2. `fpmmAbi` (SimpleFPMM contract ABI) ⬅️ **MISSING**
3. `logger` (logger instance)

But in `server.js` it was being called with only **2 parameters**:

```javascript
// ❌ WRONG - Missing fpmmAbi
const unwatchFunctions = await startTradeListener(
  activeFpmmAddresses,
  app.log  // This was passed as 2nd param, but function expects it as 3rd
);
```

The function's validation threw the error because `app.log` was being received as the `fpmmAbi` parameter (2nd position), leaving `logger` (3rd position) as `undefined`.

## Solution

**File**: `backend/fastify/server.js`

1. **Import SimpleFPMMAbi** (line 14):

```javascript
import simpleFpmmAbi from "../src/abis/SimpleFPMMAbi.js";
```

2. **Pass all 3 parameters** (lines 211-215):

```javascript
// ✅ CORRECT - All 3 parameters
const unwatchFunctions = await startTradeListener(
  activeFpmmAddresses,
  simpleFpmmAbi,  // Added missing ABI parameter
  app.log
);
```

## Files Modified

- `backend/fastify/server.js` - Added import and fixed function call

## Testing

After restarting the backend, you should see:

```text
✅ Trade listeners started for X FPMM contract(s)
```

Instead of:

```text
❌ Failed to start Trade listeners: logger instance is required
```

## Prevention

This type of error happens when function signatures change but call sites aren't updated. Always check:

1. Function signature in the implementation
2. All call sites match the signature
3. Parameter order is correct
4. Required imports are present
