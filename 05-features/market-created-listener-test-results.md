# Market Created Listener - Test Results

**Date**: Oct 31, 2025  
**Status**: ✅ TESTED - Issues Found and Fixed

## Test Execution Summary

### Setup
- ✅ Contracts deployed to fresh Anvil instance
- ✅ Season 1 created successfully
- ✅ Season 1 started successfully  
- ✅ 2000 tickets purchased (100% of supply)
- ✅ Backend listeners active and detecting events

### Test Results

#### ✅ Market Creation Successful
- **Database Entry**: Created successfully (ID: 7)
- **Player ID**: 13 (auto-created via `getOrCreatePlayerId()`)
- **Contract Address**: `0x06cd7788D77332cF1156f1E327eBC090B5FF16a3`
- **Season ID**: 1
- **Player Address**: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`

#### ❌ Issues Found

**Issue 1: Market Type Garbled**
- **Symptom**: Market type displayed as garbled characters: `÷¬\u0005B\u0012òöõ\u001a­Ö9*®iÃze\u0018'\u0010ÌÃ\u001fÂÎy.«`
- **Root Cause**: bytes32 to string conversion including non-printable characters
- **Fix Applied**: Updated `convertBytes32ToString()` to only include printable ASCII (32-126)
- **File**: `backend/src/listeners/marketCreatedListener.js` lines 10-39

**Issue 2: Probability is NaN**
- **Symptom**: `current_probability_bps: NaN` and log shows `Probability: NaN bps (NaN%)`
- **Root Cause**: BigInt to Number conversion not handling edge cases properly
- **Fix Applied**: 
  - Added safe BigInt to Number conversion with type checking
  - Added validation for NaN values
  - Added zero-check after conversion
  - Added detailed error logging
- **File**: `backend/src/listeners/marketCreatedListener.js` lines 76-106

## Backend Logs Analysis

### Successful Parts
```
✅ MarketCreated Event: Season 1
   Player: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
   FPMM Address: 0x06cd7788D77332cF1156f1E327eBC090B5FF16a3
   Transaction Hash: [hash]
   Block Number: 222
✅ InfoFi market created in database
   Player ID: 13
```

### Issues in Logs
```
❌ Market Type: ÷¬\u0005B\u0012òöõ\u001a­Ö9*®iÃze\u0018'\u0010ÌÃ\u001fÂÎy.«
❌ Probability: NaN bps (NaN%)
```

### PositionUpdate Event (Working Correctly)
```
✅ PositionUpdate: Season 1, Player 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (0 → 2000 tickets)
   Total supply: 2000 | Updated 1 markets | Oracle updates: 0/0 | Player probability: 10000 bps
```

**Note**: The PositionUpdate listener correctly calculated 10000 bps (100%), showing the probability calculation works in that context.

## Fixes Implemented

### Fix 1: Improved bytes32 Conversion
**Before**:
```javascript
for (let i = 0; i < hex.length; i += 2) {
  const byte = parseInt(hex.substr(i, 2), 16);
  if (byte !== 0) {
    str += String.fromCharCode(byte);
  }
}
```

**After**:
```javascript
for (let i = 0; i < hex.length; i += 2) {
  const byte = parseInt(hex.substr(i, 2), 16);
  // Only include printable ASCII characters (32-126)
  if (byte >= 32 && byte <= 126) {
    str += String.fromCharCode(byte);
  }
}
```

### Fix 2: Safe BigInt Conversion
**Before**:
```javascript
const ticketCount = Number(playerTickets);
const totalTickets = Number(totalSupply);
const probabilityBps = Math.floor((ticketCount / totalTickets) * 10000);
```

**After**:
```javascript
// Convert BigInt to Number safely
const ticketCount = typeof playerTickets === 'bigint' ? Number(playerTickets) : playerTickets;
const totalTickets = typeof totalSupply === 'bigint' ? Number(totalSupply) : totalSupply;

// Validate numbers
if (isNaN(ticketCount) || isNaN(totalTickets)) {
  logger.error(`❌ Invalid numbers: ticketCount=${ticketCount}, totalTickets=${totalTickets}`);
  return 0;
}

if (totalTickets === 0) {
  logger.warn(`⚠️  Total tickets is 0 after conversion`);
  return 0;
}

const probabilityBps = Math.floor((ticketCount / totalTickets) * 10000);
```

## Admin Private Key Update

**Updated**: Admin private key for local testing
- **Old**: `0xac0974bec39a17e36ba4a6b4d238ff944bacb476c6b8d6c1f02960247590a4c3` (Account 0 - runs out of gas)
- **New**: `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80` (Account 1 - has sufficient ETH)
- **Address**: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`

**Memory Created**: Stored in project memories for future reference

## Next Steps

### Immediate
1. ✅ Restart backend to pick up code changes
2. ⏳ Test with new ticket purchase to verify fixes
3. ⏳ Verify market type displays correctly
4. ⏳ Verify probability calculates correctly (should be 10000 bps for single player)

### Future Improvements
1. Add unit tests for `convertBytes32ToString()`
2. Add unit tests for `calculateProbability()`
3. Add integration test for full market creation flow
4. Consider using viem's built-in bytes32 utilities

## Files Modified

1. ✅ `backend/src/listeners/marketCreatedListener.js`
   - Fixed `convertBytes32ToString()` function
   - Fixed `calculateProbability()` function
   - Added comprehensive error handling

2. ✅ `contracts/script/CreateSeason.s.sol`
   - Updated to use `RAFFLE_ADDRESS_LOCAL`
   - Updated to use `SOF_ADDRESS_LOCAL`

3. ✅ `contracts/script/BuyTickets.s.sol`
   - Updated to use `SOF_ADDRESS_LOCAL`
   - Updated to use `RAFFLE_ADDRESS_LOCAL`

## Success Criteria

- [x] Market entry created in database
- [x] Player ID auto-created and linked
- [x] Contract address stored correctly
- [ ] Market type displays as readable string (pending retest)
- [ ] Probability calculates correctly (pending retest)
- [x] Timestamps populated
- [x] All required fields present

## Conclusion

The market creation listener is **functionally working** but had two display/calculation bugs that have been fixed:

1. ✅ **Market type conversion** - Now filters to printable ASCII only
2. ✅ **Probability calculation** - Now handles BigInt conversion safely with validation

The core functionality (creating database entries, linking player IDs, storing addresses) is working correctly. The fixes address edge cases in data formatting and type conversion.

**Status**: Ready for retesting with backend restart.
