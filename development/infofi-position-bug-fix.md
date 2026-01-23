# InfoFi Position Display Bug Fix

## Issue Description

When a user placed a NO position on one InfoFi market, the same position was incorrectly displayed on ALL markets for that season. This created the false impression that the user had positions in multiple markets when they only had one.

## Root Cause Analysis

### The Problem Chain

1. **Missing Event Data**: The `InfoFiMarketFactory.sol` contract's `MarketCreated` event does not emit the `marketId` (uint256) that is returned from `infoFiMarket.createMarket()`. The event only emits:
   - `seasonId`
   - `player`
   - `marketType` (bytes32)
   - `probabilityBps`
   - `marketAddress`

2. **Frontend Workaround Failure**: The frontend code in `listSeasonWinnerMarketsByEvents()` tried to extract `marketId` from the event args, but since it doesn't exist, it would return `undefined` or `null`.

3. **Incorrect ID Derivation**: In `InfoFiMarketCard.jsx`, when the market ID was missing or looked like a bytes32 hash, the component would derive the "effective" market ID by reading `nextMarketId` from the contract and using `nextMarketId - 1`.

4. **Same ID for All Markets**: Since ALL markets would derive the same `effectiveMarketId` (the last created market's ID), they would all query positions for the SAME market, causing all cards to display identical position data.

### Example of the Bug

```text
Market 1 (Player1): id = 0xe7000b... (bytes32 hash)
  → Derives effectiveMarketId = nextMarketId - 1 = 1

Market 2 (Player2): id = 0xfedc3f... (bytes32 hash)  
  → Derives effectiveMarketId = nextMarketId - 1 = 1

Both markets query positions for marketId=1, showing the same data!
```

## Solution Implemented

### 1. Read Market IDs from Contract Storage

Modified `listSeasonWinnerMarketsByEvents()` in `/src/services/onchainInfoFi.js` to read the actual market IDs from the `winnerPredictionMarketIds` mapping in the factory contract:

```javascript
// Read the actual marketId from the factory's storage mapping
// since the event doesn't emit it
let marketId;
try {
  marketId = await publicClient.readContract({
    address: factory.address,
    abi: MarketIdMappingAbi,
    functionName: 'winnerPredictionMarketIds',
    args: [BigInt(sid), getAddress(player)]
  });
} catch (e) {
  // Fallback: try to extract from event args (in case contract was updated)
  marketId = args.marketId ?? args._marketId;
}
```

### 2. Simplified Market ID Derivation

Simplified the market ID derivation logic in `InfoFiMarketCard.jsx` to directly use the market ID from the props, since it's now guaranteed to be a proper uint256 string:

```javascript
React.useEffect(() => {
  // Simply use the market.id directly - it should now be a proper uint256 string
  // from listSeasonWinnerMarketsByEvents which reads winnerPredictionMarketIds mapping
  if (market?.id != null) {
    setDerivedMid(String(market.id));
  } else {
    setDerivedMid(null);
  }
}, [market?.id]);
```

### 3. Removed Unused Code

Removed the now-unnecessary `MarketMiniAbi` and complex derivation logic that was causing the bug.

## Files Modified

1. `/src/services/onchainInfoFi.js`
   - Added `MarketIdMappingAbi` to read `winnerPredictionMarketIds` mapping
   - Modified `listSeasonWinnerMarketsByEvents()` to fetch actual market IDs from contract storage

2. `/src/components/infofi/InfoFiMarketCard.jsx`
   - Simplified market ID derivation to use the ID directly from props
   - Removed unused `MarketMiniAbi` and complex fallback logic
   - Cleaned up lint warnings

## Long-term Fix Recommendation

The proper fix would be to update the `InfoFiMarketFactory.sol` contract to emit the `marketId` in the `MarketCreated` event:

```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 indexed marketType,
    uint256 marketId,        // ADD THIS
    uint256 probabilityBps,
    address marketAddress
);

// In the onPositionUpdate function:
emit MarketCreated(seasonId, player, WINNER_PREDICTION, marketIdU256, newBps, marketAddr);
```

This would eliminate the need for the extra contract read and make the event data complete.

## Testing

To verify the fix:

1. Create a season with multiple players
2. Have each player buy enough tickets to cross the 1% threshold (creating markets)
3. Place a bet on ONE market
4. Verify that ONLY that market shows the position, not all markets

## Impact

- **Before**: All markets showed the same position data (incorrect)
- **After**: Each market correctly shows only its own position data
- **Performance**: Minimal impact - one additional contract read per market during listing (cached by React Query)
