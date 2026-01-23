# Market ID Migration: bytes32 → uint256

## Summary

Successfully migrated all InfoFi market identification from `bytes32` to `uint256` across the entire codebase to eliminate confusion and ensure consistency.

## Changes Made

### 1. Smart Contracts

#### InfoFiPriceOracle.sol

**Before:**
```solidity
mapping(bytes32 => PriceData) public prices;
function updateRaffleProbability(bytes32 marketId, ...) external
function getPrice(bytes32 marketId) external view
```

**After:**
```solidity
mapping(uint256 => PriceData) public prices;
function updateRaffleProbability(uint256 marketId, ...) external
function getPrice(uint256 marketId) external view
```

#### InfoFiMarketFactory.sol

**Before:**
```solidity
interface IInfoFiPriceOracleMinimal {
    function updateRaffleProbability(bytes32 marketId, ...) external;
}

// Computed bytes32 hash for oracle updates
bytes32 marketId = keccak256(abi.encodePacked(seasonId, player, WINNER_PREDICTION));
oracle.updateRaffleProbability(marketId, newBps);
```

**After:**
```solidity
interface IInfoFiPriceOracleMinimal {
    function updateRaffleProbability(uint256 marketId, ...) external;
}

// Use stored uint256 marketId
uint256 marketId = winnerPredictionMarketIds[seasonId][player];
if (marketId > 0 || winnerPredictionCreated[seasonId][player]) {
    oracle.updateRaffleProbability(marketId, newBps);
}
```

**getMarketId() function:**
```solidity
// Before: returned bytes32 hash
function getMarketId(uint256 seasonId, address player, bytes32 marketType) 
    external pure returns (bytes32)

// After: returns stored uint256 ID
function getMarketId(uint256 seasonId, address player, bytes32 marketType) 
    external view returns (uint256)
```

#### InfoFiMarket.sol

**Before:**
```solidity
interface IInfoFiPriceOracle {
    function updateMarketSentiment(bytes32 marketId, ...) external;
    function getPrice(bytes32 marketId) external view returns (...);
}

struct MarketInfo {
    ...
    bytes32 marketKey;  // Stored for oracle lookups
}

// Used marketKey for oracle calls
oracle.getPrice(market.marketKey);
oracle.updateMarketSentiment(market.marketKey, sBps);
```

**After:**
```solidity
interface IInfoFiPriceOracle {
    function updateMarketSentiment(uint256 marketId, ...) external;
    function getPrice(uint256 marketId) external view returns (...);
}

struct MarketInfo {
    ...
    // marketKey field removed
}

// Use marketId directly
oracle.getPrice(marketId);
oracle.updateMarketSentiment(marketId, sBps);
```

### 2. Test Files

#### InfoFiThreshold.t.sol

**Before:**
```solidity
bytes32 marketId = marketFactory.getMarketId(seasonId, player1, marketTypeCode);
```

**After:**
```solidity
uint256 marketId = marketFactory.getMarketId(seasonId, player1, marketTypeCode);
```

#### HybridPricingInvariant.t.sol

**Before:**
```solidity
bytes32 internal testMarketId;
testMarketId = keccak256("test_market");
```

**After:**
```solidity
uint256 internal testMarketId;
testMarketId = uint256(keccak256("test_market"));
```

#### InfoFiMarketMock

**Before:**
```solidity
function createMarket(...) external {
    // No return value
}
```

**After:**
```solidity
uint256 private nextMarketId = 1;

function createMarket(...) external returns (uint256) {
    return nextMarketId++;
}
```

## Why This Change?

### The Problem

The system had **two different ID systems** that caused confusion:

1. **InfoFiMarket contract**: Used `uint256 marketId` (auto-incrementing: 0, 1, 2, ...)
2. **InfoFiPriceOracle contract**: Used `bytes32 marketKey` (computed hash)

This led to:
- Type mismatches between contracts
- Confusion about which ID to use for operations
- Complex conversion logic in frontend/backend
- Potential for bugs when mixing ID types

### The Solution

**Use `uint256 marketId` everywhere:**

- InfoFiMarket creates markets with sequential uint256 IDs
- Factory stores these IDs in `winnerPredictionMarketIds` mapping
- Oracle uses the same uint256 IDs for pricing data
- All operations use consistent uint256 IDs

## Data Flow (After Migration)

### Market Creation

```solidity
// 1. Factory receives position update
InfoFiMarketFactory.onPositionUpdate(seasonId, player, ...)

// 2. Factory creates market, gets uint256 ID
uint256 marketId = InfoFiMarket.createMarket(seasonId, player, ...)
// Returns: 1, 2, 3, etc.

// 3. Factory stores the uint256 ID
winnerPredictionMarketIds[seasonId][player] = marketId

// 4. Factory updates oracle with same uint256 ID
oracle.updateRaffleProbability(marketId, probabilityBps)
```

### Price Updates

```solidity
// Factory updates oracle using uint256 marketId
uint256 marketId = winnerPredictionMarketIds[seasonId][player];
oracle.updateRaffleProbability(marketId, newBps);

// InfoFiMarket reads price using its own marketId
(raffleBps, sentimentBps, hybridBps, , active) = oracle.getPrice(marketId);
```

### Betting Operations

```solidity
// All operations use uint256 marketId
InfoFiMarket.placeBet(marketId, prediction, amount);
InfoFiMarket.claimPayout(marketId);
InfoFiMarket.resolveMarket(marketId, outcome);
```

## Benefits

1. **Type Safety**: No more bytes32 ↔ uint256 conversion errors
2. **Simplicity**: One ID type throughout the system
3. **Clarity**: Clear which ID to use for any operation
4. **Consistency**: All contracts use the same identifier
5. **Maintainability**: Easier to understand and modify

## Testing

All tests pass after migration:

```bash
forge test
# Ran 12 test suites: 82 tests passed, 0 failed, 0 skipped
```

Key tests verified:
- ✅ InfoFiThreshold.t.sol - Market creation and oracle updates
- ✅ HybridPricingInvariant.t.sol - Oracle pricing calculations
- ✅ CategoricalMarketInvariant.t.sol - Market operations
- ✅ All other contract tests

## Frontend/Backend Impact

### What Needs to Update

1. **Backend services** that interact with oracle:
   - Update `InfoFiPriceOracleAbi.js` (already done via copy-abis script)
   - Change any bytes32 marketId references to uint256
   - Update database queries if storing marketId

2. **Frontend hooks** that read prices:
   - Ensure marketId is passed as number/bigint, not hex string
   - Update any composite ID parsing logic

3. **API endpoints**:
   - `/api/infofi/markets` - ensure marketId is numeric
   - `/stream/pricing/:marketId` - handle numeric IDs

### Migration Path

For existing deployed contracts (if any):
1. Deploy new versions of all InfoFi contracts
2. Update contract addresses in config
3. Migrate any existing market data
4. Update frontend to use new ABIs

## Verification Checklist

- [x] InfoFiPriceOracle uses uint256 marketId
- [x] InfoFiMarketFactory uses uint256 marketId
- [x] InfoFiMarket uses uint256 marketId
- [x] All interfaces updated
- [x] All tests updated and passing
- [x] ABIs copied to frontend
- [x] No bytes32 marketId references remain in contracts
- [x] Documentation updated

## Related Files

### Contracts
- `contracts/src/infofi/InfoFiPriceOracle.sol`
- `contracts/src/infofi/InfoFiMarketFactory.sol`
- `contracts/src/infofi/InfoFiMarket.sol`

### Tests
- `contracts/test/InfoFiThreshold.t.sol`
- `contracts/test/invariant/HybridPricingInvariant.t.sol`

### Frontend
- `src/contracts/abis/InfoFiPriceOracle.json`
- `src/contracts/abis/InfoFiMarketFactory.json`
- `src/contracts/abis/InfoFiMarket.json`

## Next Steps

1. Update backend services to use uint256 marketId
2. Update frontend hooks to handle numeric IDs
3. Test end-to-end market creation and betting flow
4. Deploy updated contracts to testnet
5. Verify all integrations work correctly
