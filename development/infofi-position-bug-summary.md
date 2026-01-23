# InfoFi Position Display Bug - Complete Fix Summary

## Issue Resolved ✅

**Problem**: Buying a NO position on one InfoFi market incorrectly displayed the same position on ALL markets for that season.

**Status**: **FIXED** (2025-10-05)

## Root Cause

The bug was caused by a chain of issues:

1. **Missing Event Data**: `InfoFiMarketFactory.sol`'s `MarketCreated` event doesn't emit the `marketId` (uint256)
2. **Fallback Logic Failure**: All markets derived the same `effectiveMarketId` using `nextMarketId - 1`
3. **Shared Query Keys**: All markets queried positions for the same market ID, showing identical data

## Solution Implemented

### Files Modified

1. **`/src/services/onchainInfoFi.js`**
   - Added logic to read actual market IDs from `winnerPredictionMarketIds` mapping
   - Each market now gets its unique ID from contract storage

2. **`/src/components/infofi/InfoFiMarketCard.jsx`**
   - Simplified market ID derivation to use ID directly from props
   - Removed complex fallback logic causing the bug

### Test Coverage

Created comprehensive test suite in `/tests/services/infoFiMarketIds.test.js`:

- ✅ Verifies unique market IDs for different players
- ✅ Tests the bug scenario (all markets deriving same ID)
- ✅ Validates fallback to event args when mapping read fails

**All tests passing** (3/3)

## Verification

### Before Fix

```text
Market 1 (Player1): Shows "No 1000 SOF" position
Market 2 (Player2): Shows "No 1000 SOF" position  ❌ WRONG
```

### After Fix

```text
Market 1 (Player1): Shows "No 1000 SOF" position
Market 2 (Player2): Shows no position  ✅ CORRECT
```

## Technical Details

### Market ID Resolution Flow

**Before (Buggy)**:

```text
Event → No marketId → Derive from nextMarketId - 1 → Same ID for all markets
```

**After (Fixed)**:

```text
Event → Read winnerPredictionMarketIds[seasonId][player] → Unique ID per market
```

### Query Key Structure

Each market now uses a unique `effectiveMarketId`:

```javascript
queryKey: ['infofiBet', effectiveMarketId, address, prediction]
```

Where `effectiveMarketId` is the unique uint256 from the contract mapping.

## Long-term Recommendation

Update `InfoFiMarketFactory.sol` to emit `marketId` in the event:

```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 indexed marketType,
    uint256 marketId,        // ADD THIS
    uint256 probabilityBps,
    address marketAddress
);
```

This would eliminate the extra contract read and make events self-contained.

## Documentation

- **Detailed Analysis**: `/docs/03-development/infofi-position-bug-fix.md`
- **Test Suite**: `/tests/services/infoFiMarketIds.test.js`
- **Project Tasks**: Updated in `/instructions/project-tasks.md`

## Impact

- ✅ Each market correctly displays only its own positions
- ✅ No performance degradation (one extra read per market, cached)
- ✅ Full test coverage for regression prevention
- ✅ Build and lint checks passing
