# PositionUpdate Event Listener - Revised Architectural Plan

**Status**: Planning Phase - REVISED  
**Date**: Oct 25, 2025  
**Revision**: Critical Issue - Cascading Probability Updates

---

## Executive Summary - REVISED

The original plan had a **critical flaw**: updating only the player who bought/sold tickets. In reality, **ALL players' win probabilities change** when total supply changes, because probability = `(playerTickets / totalTickets) * 10000`.

**Example**:
```
Initial state: 3 players, 1000 total tickets
- Player A: 400 tickets → 40% win probability
- Player B: 300 tickets → 30% win probability
- Player C: 300 tickets → 30% win probability

Player A buys 100 tickets:
- New total: 1100 tickets
- Player A: 500 tickets → 45.45% (CHANGED)
- Player B: 300 tickets → 27.27% (CHANGED)
- Player C: 300 tickets → 27.27% (CHANGED)

All three markets must be updated!
```

---

## Revised Architecture

```
Player buys/sells tickets
    ↓
SOFBondingCurve emits PositionUpdate(seasonId, player, oldTickets, newTickets, totalTickets, newBps)
    ↓
Backend listener (Viem polling) captures event
    ↓
Query: Get ALL players with markets in this season
    ↓
For each player:
  - Recalculate: newProbability = (playerTickets / totalTickets) * 10000
  - UPDATE infofi_markets.current_probability_bps
  - Emit SSE update to frontend
    ↓
All players' odds updated within 5 seconds
```

---

## Critical Implementation Changes

### Change 1: Database Method Must Update ALL Players

**Old Approach** (WRONG):
```javascript
async updateMarketProbability(seasonId, playerAddress, newProbabilityBps) {
  // Update only ONE player
  // ❌ WRONG: Leaves other players with stale odds
}
```

**New Approach** (CORRECT):
```javascript
async updateAllPlayerProbabilities(seasonId, totalTickets) {
  // 1. Get ALL players with active markets in this season
  // 2. For each player, fetch their current ticket count
  // 3. Recalculate: newBps = (playerTickets / totalTickets) * 10000
  // 4. UPDATE all markets in single batch operation
  // 5. Return array of updated markets
}
```

### Change 2: Event Data Provides totalTickets

**Good News**: The `PositionUpdate` event already includes `totalTickets`!

```solidity
event PositionUpdate(
    uint256 indexed seasonId,
    address indexed player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets,  // ← We have this!
    uint256 probabilityBps
);
```

This is the **new total supply** after the transaction, so we can use it directly.

### Change 3: Listener Must Fetch All Player Positions

**Challenge**: We need current ticket counts for ALL players, but the event only tells us about ONE player.

**Solution**: Query `Raffle.getParticipants(seasonId)` to get all players, then fetch each player's position.

```javascript
// In listener's onLogs callback:
const { seasonId, totalTickets } = log.args;

// Get all participants in this season
const participants = await publicClient.readContract({
  address: raffleAddress,
  abi: raffleAbi,
  functionName: 'getParticipants',
  args: [seasonId]
});

// Fetch each player's ticket count
const playerPositions = await Promise.all(
  participants.map(async (player) => {
    const ticketCount = await publicClient.readContract({
      address: raffleAddress,
      abi: raffleAbi,
      functionName: 'getParticipantPosition',
      args: [seasonId, player]
    });
    return { player, ticketCount };
  })
);

// Update all markets
await db.updateAllPlayerProbabilities(seasonId, totalTickets, playerPositions);
```

---

## Detailed Implementation Plan

### Phase 1: Core Listener (Week 1)

#### Step 1.1: Create `positionUpdateListener.js`

**File**: `backend/src/listeners/positionUpdateListener.js`

```javascript
import { publicClient } from '../lib/viemClient.js';
import { db } from '../../shared/supabaseClient.js';

/**
 * Listens for PositionUpdate events and updates ALL players' probabilities
 * @param {string} bondingCurveAddress - BondingCurve contract address
 * @param {object} bondingCurveAbi - BondingCurve ABI
 * @param {string} raffleAddress - Raffle contract address
 * @param {object} raffleAbi - Raffle ABI
 * @param {object} logger - Fastify logger
 * @returns {function} Unwatch function
 */
export async function startPositionUpdateListener(
  bondingCurveAddress,
  bondingCurveAbi,
  raffleAddress,
  raffleAbi,
  logger
) {
  if (!bondingCurveAddress || !raffleAddress) {
    throw new Error('bondingCurveAddress and raffleAddress are required');
  }

  const unwatch = publicClient.watchContractEvent({
    address: bondingCurveAddress,
    abi: bondingCurveAbi,
    eventName: 'PositionUpdate',
    onLogs: async (logs) => {
      for (const log of logs) {
        const { seasonId, player, totalTickets } = log.args;

        try {
          logger.debug(`📊 PositionUpdate: Season ${seasonId}, Player ${player}, Total ${totalTickets}`);

          // Step 1: Get all participants in this season
          const participants = await publicClient.readContract({
            address: raffleAddress,
            abi: raffleAbi,
            functionName: 'getParticipants',
            args: [seasonId]
          });

          if (participants.length === 0) {
            logger.debug(`No participants in season ${seasonId}`);
            return;
          }

          logger.debug(`Found ${participants.length} participants in season ${seasonId}`);

          // Step 2: Fetch ticket count for each participant
          const playerPositions = await Promise.all(
            participants.map(async (addr) => {
              const ticketCount = await publicClient.readContract({
                address: raffleAddress,
                abi: raffleAbi,
                functionName: 'getParticipantPosition',
                args: [seasonId, addr]
              });
              return {
                player: addr,
                ticketCount: typeof ticketCount === 'bigint' ? Number(ticketCount) : ticketCount
              };
            })
          );

          // Step 3: Update all players' probabilities in database
          const updatedCount = await db.updateAllPlayerProbabilities(
            seasonId,
            typeof totalTickets === 'bigint' ? Number(totalTickets) : totalTickets,
            playerPositions
          );

          logger.info(
            `✅ Updated probabilities for ${updatedCount} markets in season ${seasonId}`
          );

        } catch (error) {
          logger.error(
            `❌ Failed to process PositionUpdate for season ${seasonId}`,
            error
          );
          // Continue listening; don't crash on individual failures
        }
      }
    },
    onError: (error) => {
      logger.error('❌ PositionUpdate Listener Error', error);
    }
  });

  logger.info(`✅ Started PositionUpdate listener for ${bondingCurveAddress}`);
  return unwatch;
}
```

#### Step 1.2: Add Database Method

**File**: `backend/shared/supabaseClient.js`

```javascript
/**
 * Update win probabilities for ALL players in a season
 * Called when any player's position changes (buy/sell)
 * 
 * @param {number} seasonId - Season identifier
 * @param {number} totalTickets - New total ticket supply
 * @param {Array} playerPositions - Array of {player, ticketCount}
 * @returns {number} Count of updated markets
 */
async updateAllPlayerProbabilities(seasonId, totalTickets, playerPositions) {
  if (totalTickets === 0) {
    this.logger?.warn(`Cannot update probabilities: totalTickets is 0 for season ${seasonId}`);
    return 0;
  }

  try {
    let updatedCount = 0;

    // Update each player's market
    for (const { player, ticketCount } of playerPositions) {
      // Calculate new probability
      const newProbabilityBps = Math.round((ticketCount * 10000) / totalTickets);

      // Update market if it exists
      const { data, error } = await this.supabase
        .from('infofi_markets')
        .update({
          current_probability_bps: newProbabilityBps,
          updated_at: new Date().toISOString()
        })
        .eq('season_id', seasonId)
        .eq('player_address', player)
        .eq('market_type', 'WINNER_PREDICTION')
        .eq('is_active', true)
        .select();

      if (error) {
        this.logger?.error(
          `Failed to update market for ${player} in season ${seasonId}`,
          error
        );
        continue;
      }

      if (data && data.length > 0) {
        updatedCount++;
        this.logger?.debug(
          `Updated ${player}: ${ticketCount} tickets → ${newProbabilityBps} bps`
        );
      }
      // Silently skip if market doesn't exist (player hasn't crossed 1% threshold)
    }

    return updatedCount;

  } catch (error) {
    this.logger?.error('Error updating player probabilities', error);
    throw error;
  }
}
```

#### Step 1.3: Integrate with Server

**File**: `backend/fastify/server.js`

```javascript
import { startPositionUpdateListener } from "../src/listeners/positionUpdateListener.js";
import bondingCurveAbi from "../src/abis/BondingCurveAbi.js";

// ... existing code ...

let unwatchPositionUpdates = [];

async function startListeners() {
  try {
    const raffleAddress = process.env.RAFFLE_ADDRESS_LOCAL;

    if (!raffleAddress) {
      app.log.warn("⚠️  RAFFLE_ADDRESS_LOCAL not set - listeners will not start");
      return;
    }

    // Start SeasonStarted listener
    unwatchSeasonStarted = await startSeasonStartedListener(raffleAddress, raffleAbi, app.log);

    // Start PositionUpdate listeners for all active seasons
    const activeSeasons = await db.getActiveSeasonContracts();

    for (const season of activeSeasons) {
      try {
        const unwatch = await startPositionUpdateListener(
          season.bonding_curve_address,
          bondingCurveAbi,
          raffleAddress,
          raffleAbi,
          app.log
        );
        unwatchPositionUpdates.push(unwatch);
      } catch (error) {
        app.log.error(
          `Failed to start PositionUpdate listener for season ${season.season_id}`,
          error
        );
      }
    }

  } catch (error) {
    app.log.error("Failed to start listeners:", error);
  }
}

// Graceful shutdown
process.on("SIGINT", async () => {
  app.log.info("Shutting down server...");

  if (unwatchSeasonStarted) {
    unwatchSeasonStarted();
    app.log.info("🛑 Stopped SeasonStarted listener");
  }

  for (const unwatch of unwatchPositionUpdates) {
    unwatch();
  }
  app.log.info(`🛑 Stopped ${unwatchPositionUpdates.length} PositionUpdate listeners`);

  await app.close();
});
```

---

## Gas & Performance Considerations

### Challenge: Fetching All Player Positions

**Problem**: Querying `getParticipantPosition()` for every player on every transaction is expensive.

**Solution Options**:

**Option A: Accept the Cost (Recommended for MVP)**
- Fetch all positions on every event
- ~50k gas per player (off-chain, so no cost to user)
- Acceptable for <100 players per season
- Simple, reliable, no caching issues

**Option B: Cache Player Positions (Future Optimization)**
- Maintain off-chain cache of player positions
- Update cache when events received
- Reduces contract reads by 90%
- Requires cache invalidation logic

**Option C: Hybrid - Fetch Only Active Players (Future)**
- Track which players have markets
- Only fetch positions for those players
- Reduces reads from N to M (where M << N)
- Requires separate tracking

**Recommendation**: Start with Option A. It's simple, correct, and scales to 100+ players. Optimize to Option B/C if profiling shows bottlenecks.

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Player A buys 100 tickets                                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ SOFBondingCurve.buyTokens()                                 │
│ - Update playerTickets[A] = 500                             │
│ - Update totalSupply = 1100                                 │
│ - Emit PositionUpdate(seasonId=1, player=A, totalTickets=1100)
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Backend Listener (positionUpdateListener.js)                │
│ - Receives event: seasonId=1, totalTickets=1100            │
│ - Calls Raffle.getParticipants(1)                          │
│   → Returns [A, B, C]                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Fetch All Player Positions                                  │
│ - A: 500 tickets → 45.45% (500/1100 * 10000)              │
│ - B: 300 tickets → 27.27% (300/1100 * 10000)              │
│ - C: 300 tickets → 27.27% (300/1100 * 10000)              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Database Batch Update                                       │
│ UPDATE infofi_markets                                       │
│ SET current_probability_bps = CASE                          │
│   WHEN player_address = A THEN 4545                         │
│   WHEN player_address = B THEN 2727                         │
│   WHEN player_address = C THEN 2727                         │
│ WHERE season_id = 1 AND market_type = 'WINNER_PREDICTION'  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ SSE Broadcast (Future)                                      │
│ - Stream update to all connected clients                    │
│ - Frontend updates odds for all 3 players                   │
│ - Users see live probability changes                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Database Operations

### SQL: Batch Update All Players

```sql
-- Option 1: Individual updates (simpler, used in implementation)
UPDATE infofi_markets
SET current_probability_bps = $1,
    updated_at = NOW()
WHERE season_id = $2 
  AND player_address = $3
  AND market_type = 'WINNER_PREDICTION'
  AND is_active = true;

-- Option 2: Batch update (more efficient, future optimization)
UPDATE infofi_markets SET
  current_probability_bps = CASE
    WHEN player_address = $1 THEN $2
    WHEN player_address = $3 THEN $4
    WHEN player_address = $5 THEN $6
    ELSE current_probability_bps
  END,
  updated_at = NOW()
WHERE season_id = $7
  AND market_type = 'WINNER_PREDICTION'
  AND is_active = true
  AND player_address IN ($1, $3, $5);
```

### Recommended Index

```sql
CREATE INDEX idx_infofi_markets_season_player 
ON infofi_markets(season_id, player_address, market_type) 
WHERE is_active = true;
```

---

## Error Handling - Revised

| Scenario | Handling | Impact |
|----------|----------|--------|
| Market doesn't exist for player | Skip silently (log debug) | No data loss; initial prob captured at market creation |
| Participant fetch fails | Log error, continue with other seasons | Single season stale; other seasons updated |
| Position fetch fails for one player | Log error, skip that player, update others | One player stale; others updated |
| Database update fails | Log error, retry on next event | Temporary stale odds; recovers on next transaction |
| Listener stops | Heartbeat monitor detects | Odds become stale until restart |

---

## Testing Strategy - Revised

### Unit Tests

**Test 1: Probability Calculation**
```javascript
// Verify correct calculation
const prob = (500 * 10000) / 1100; // Should be 4545
expect(prob).toBe(4545);
```

**Test 2: Batch Update Logic**
```javascript
// Mock 3 players with different positions
// Verify all 3 markets updated with correct probabilities
const updated = await db.updateAllPlayerProbabilities(
  seasonId,
  1100,
  [
    { player: A, ticketCount: 500 },
    { player: B, ticketCount: 300 },
    { player: C, ticketCount: 300 }
  ]
);
expect(updated).toBe(3);
```

### Integration Tests

**Test 1: End-to-End Multi-Player Update**
1. Deploy contracts to Anvil
2. Create season with 3 players
3. Each player buys tickets to cross 1% threshold (creates markets)
4. Player A buys more tickets
5. Verify:
   - PositionUpdate event captured
   - All 3 markets updated
   - Probabilities sum to 10000 (100%)
   - Database consistency maintained

**Test 2: Rapid Sequential Transactions**
1. Player A buys 100 tickets
2. Player B buys 50 tickets (within 2 seconds)
3. Player C buys 75 tickets (within 2 seconds)
4. Verify:
   - All events captured
   - No race conditions
   - Final probabilities correct
   - All markets updated

**Test 3: Large Player Base**
1. Create season with 50 players
2. Each crosses 1% threshold
3. Player 1 buys tickets
4. Verify:
   - All 50 markets updated
   - Latency < 5 seconds
   - No database errors
   - Scalability acceptable

---

## Success Criteria - Revised

✅ Captures all `PositionUpdate` events  
✅ **Updates ALL players' probabilities on each event**  
✅ Database updates within 5 seconds  
✅ Zero events lost during normal operation  
✅ Graceful error handling (listener never crashes)  
✅ Supports unlimited concurrent seasons  
✅ Probability updates visible on frontend within 10 seconds  
✅ **Probabilities sum to 10000 (100%) after each update**  
✅ Zero data corruption or orphaned records  
✅ **Scales to 100+ players per season**  

---

## Implementation Timeline - Revised

### Week 1: Core Implementation
- **Day 1-2**: Create `positionUpdateListener.js` with all-player update logic
- **Day 2-3**: Add `updateAllPlayerProbabilities()` to `supabaseClient.js`
- **Day 3-4**: Integrate with `server.js` and test on Anvil
- **Day 4-5**: Unit tests and error handling

### Week 2: Testing & Deployment
- **Day 1-2**: Integration tests (multi-player, rapid transactions)
- **Day 2-3**: Performance testing (50+ players)
- **Day 3-4**: Staging deployment and monitoring setup
- **Day 4-5**: Production deployment

---

## Key Differences from Original Plan

| Aspect | Original | Revised |
|--------|----------|---------|
| **Scope** | Update single player | Update ALL players |
| **Data Fetching** | Use event data only | Fetch all participants + positions |
| **Database Calls** | 1 update per event | N updates per event (N = players) |
| **Complexity** | Simple | Moderate |
| **Correctness** | ❌ WRONG | ✅ CORRECT |
| **Scalability** | Breaks at 10+ players | Works to 100+ players |

---

## Future Optimizations

1. **Batch SQL Updates** - Use CASE statement for single query
2. **Position Caching** - Cache player positions to reduce contract reads
3. **Active Player Tracking** - Only update players with markets
4. **Probability History** - Track changes for analytics
5. **SSE Broadcasting** - Real-time frontend updates

---

## Appendix: Code Structure

```
backend/
├── src/
│   ├── listeners/
│   │   ├── seasonStartedListener.js      (existing)
│   │   └── positionUpdateListener.js     (NEW - cascading updates)
│   ├── lib/
│   │   └── viemClient.js                 (existing)
│   └── abis/
│       ├── RaffleAbi.js                  (existing)
│       └── BondingCurveAbi.js            (existing)
├── shared/
│   ├── supabaseClient.js                 (modify: add updateAllPlayerProbabilities)
│   └── redisClient.js                    (existing)
└── fastify/
    └── server.js                         (modify: integrate listener)
```

---

## Questions Answered

**Q: Why update all players?**  
A: Win probability = (playerTickets / totalTickets). When totalTickets changes, ALL probabilities change.

**Q: How do we know which players to update?**  
A: Query `Raffle.getParticipants(seasonId)` to get all players in the season.

**Q: What if a player doesn't have a market yet?**  
A: We silently skip them (log at debug level). Markets are created separately when crossing 1% threshold.

**Q: Won't fetching all positions be slow?**  
A: It's off-chain, so no cost to users. For <100 players, it's acceptable. We can optimize later with caching.

**Q: What about race conditions?**  
A: Each event triggers independent updates. Database handles concurrent updates correctly.

