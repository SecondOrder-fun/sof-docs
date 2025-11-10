# Bonding Curve Listener Fix - Position Update Handler Integration

## Problem

When new players joined a raffle, their markets were created with correct probabilities, but **existing players' probabilities were NOT updated**. This caused markets to show incorrect winning odds.

### Example
- 3 players initially: Each shows 33.33% (correct)
- Player 4 joins: Player 4 shows 25% (correct), but Players 1-3 still show 33.33% (WRONG - should be 25%)

## Root Cause

The `scanHistoricalPositionUpdates()` function in `bondingCurveListener.js` was:
1. ✅ Creating new markets when players crossed the 1% threshold
2. ❌ **NOT calling `positionUpdateHandler.handlePositionUpdate()`** to recalculate ALL players' probabilities

This meant:
- New markets got correct initial probabilities
- Existing markets NEVER got updated when new players joined
- The database showed stale probabilities

## Solution

### File Modified: `backend/src/services/bondingCurveListener.js`

Added `PositionUpdateHandler` call to the historical scan loop:

```javascript
// BEFORE (lines 202-238):
const { seasonId, player, probabilityBps, oldTickets, newTickets, totalTickets } = log.args;
const newBps = Number(probabilityBps);

logger.info(`Event details: season=${seasonId}, player=${player}, probability=${newBps}bps`);

// Only created markets, never updated existing ones
if (newBps >= THRESHOLD_BPS) {
  const exists = await db.hasInfoFiMarket(Number(seasonId), String(player), 'WINNER_PREDICTION');
  if (!exists) {
    await createMarketForPlayer(...);
  }
}

// AFTER (lines 202-239):
const { seasonId, player, probabilityBps, oldTickets, newTickets, totalTickets } = log.args;
const newBps = Number(probabilityBps);
const seasonIdNum = Number(seasonId);
const playerAddr = String(player);

logger.info(`Event details: season=${seasonIdNum}, player=${playerAddr}, probability=${newBps}bps`);

// ✨ NEW: ALWAYS update all player probabilities when ANY position changes
logger.info(`🔄 Updating all player probabilities for season ${seasonIdNum}...`);
const positionHandler = new PositionUpdateHandler(networkKey);
await positionHandler.handlePositionUpdate(seasonIdNum, playerAddr, logger);

// Then check if we need to create a new market
if (newBps >= THRESHOLD_BPS) {
  const exists = await db.hasInfoFiMarket(seasonIdNum, playerAddr, 'WINNER_PREDICTION');
  if (!exists) {
    await createMarketForPlayer(...);
  }
}
```

### Key Changes

1. **Added position update handler import** (already existed)
2. **Call `handlePositionUpdate()` for EVERY event** - This recalculates probabilities for ALL players in the raffle
3. **Then create market if needed** - Market creation happens after probability updates

## How It Works Now

When processing historical `PositionUpdate` events:

1. **Event received**: `PositionUpdate(seasonId=1, player=0x15d3..., newTickets=1000, totalTickets=4000)`
2. **Update ALL players**: `positionUpdateHandler.handlePositionUpdate(1, '0x15d3...')`
   - Fetches all 4 participants from Raffle contract
   - Calculates new probabilities: 25% each (1000/4000)
   - Updates database for ALL 4 players
3. **Create market if needed**: If player crossed 1% threshold, create their market

## Testing

### Manual Rescan Endpoint

Added admin endpoint to manually trigger historical scan:

```bash
POST http://localhost:3000/api/admin/rescan-bonding-curve
Content-Type: application/json

{
  "bondingCurveAddress": "0x...",
  "fromBlock": 0
}
```

This will:
1. Scan all historical `PositionUpdate` events
2. Call `positionUpdateHandler` for each event
3. Update ALL players' probabilities in database
4. Create any missing markets

### Verification Steps

1. **Check database before**:
   ```sql
   SELECT player_address, current_probability_bps 
   FROM infofi_markets 
   WHERE raffle_id = 1;
   ```

2. **Trigger rescan**:
   ```bash
   curl -X POST http://localhost:3000/api/admin/rescan-bonding-curve \
     -H "Content-Type: application/json" \
     -d '{"bondingCurveAddress": "0x..."}'
   ```

3. **Check database after**:
   ```sql
   SELECT player_address, current_probability_bps 
   FROM infofi_markets 
   WHERE raffle_id = 1;
   ```
   
   All players should now show the same probability (e.g., 2500 bps = 25% for 4 players)

4. **Check frontend**: All markets should show correct matching probabilities

## Why The Live Listener Wasn't Working

The live `watchContractEvent` listener (lines 48-130) **already had the position update handler call** (lines 78-79), so it was correct.

The problem was that:
1. Historical events were processed on startup WITHOUT calling the handler
2. This left the database in an inconsistent state
3. Future events would update correctly, but past events created the wrong initial state

## Files Changed

- ✅ `backend/src/services/bondingCurveListener.js` - Added position update handler to historical scan
- ✅ `backend/fastify/routes/adminRoutes.js` - Added `/admin/rescan-bonding-curve` endpoint
- ✅ `backend/src/services/positionUpdateHandler.js` - Added detailed logging (already done)

## Impact

- ✅ **Historical scans now update ALL players** when processing past events
- ✅ **Live events already worked** (position handler was already called)
- ✅ **Manual rescan available** for fixing existing inconsistent data
- ✅ **Detailed logging** shows exactly what's being updated

## Next Steps

1. Restart backend to load the fixed code
2. Call the rescan endpoint to fix existing data
3. Buy new tickets to verify live updates work correctly
4. All players should always show matching probabilities

---

**Fix Date**: 2025-10-25  
**Status**: ✅ Complete  
**Testing**: Ready for manual verification
