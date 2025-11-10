# PositionUpdate Event Listener - Architectural Plan

**Status**: Planning Phase  
**Date**: Oct 25, 2025  
**Architect**: Senior Planning Review

---

## Executive Summary

Implement a backend listener that tracks `PositionUpdate` events from `SOFBondingCurve` and records player win probabilities to the `infofi_markets` table in Supabase. This replaces on-chain position storage with off-chain caching, reducing gas costs and enabling real-time odds updates.

**Key Decision**: Remove on-chain `RafflePositionTracker` entirely. Player holdings are already tracked in `BondingCurve` and `RaffleToken` contracts; we compute probabilities off-chain and cache them.

---

## Architecture Overview

```
SOFBondingCurve.buyTokens() / sellTokens()
    ↓
emit PositionUpdate(seasonId, player, oldTickets, newTickets, totalTickets, newBps)
    ↓
Viem watchContractEvent (polling)
    ↓
positionUpdateListener.onLogs()
    ↓
Update infofi_markets.current_probability_bps
    ↓
Emit SSE update for frontend
```

---

## Data Flow & Integration Points

### 1. Event Source: SOFBondingCurve.PositionUpdate

**Event Signature** (lines 70-77 of SOFBondingCurve.sol):

```solidity
event PositionUpdate(
    uint256 indexed seasonId,
    address indexed player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets,
    uint256 probabilityBps
);
```

**Emission Points**:
- Line 214-221: After `buyTokens()` completes
- Line 307-314: After `sellTokens()` completes

**Event Data Interpretation**:
- `seasonId`: Season identifier (matches `raffleSeasonId` in curve)
- `player`: Player address (indexed for filtering)
- `oldTickets`: Previous ticket count
- `newTickets`: Current ticket count
- `totalTickets`: Total supply after transaction
- `probabilityBps`: Win probability in basis points (0-10000)

### 2. Database Target: infofi_markets Table

**Relevant Columns** (from project-requirements.md):

```sql
CREATE TABLE infofi_markets (
  id BIGSERIAL PRIMARY KEY,
  season_id BIGINT NOT NULL,
  player_address VARCHAR(42) NOT NULL,
  market_type VARCHAR(50),
  initial_probability_bps INTEGER,
  current_probability_bps INTEGER,  -- ← UPDATE THIS
  is_active BOOLEAN,
  created_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ
);
```

**Update Strategy**:
- Find market by `(season_id, player_address, market_type='WINNER_PREDICTION')`
- Update `current_probability_bps` = event's `probabilityBps`
- Update `updated_at` = NOW()
- If no market exists yet, skip (market creation happens separately at 1% threshold)

### 3. Contract Address Discovery

**Source**: `season_contracts` table (already populated by `seasonStartedListener`)

**Query Pattern**:
```sql
SELECT bonding_curve_address 
FROM season_contracts 
WHERE season_id = $1 AND is_active = true
```

---

## Implementation Plan

### Phase 1: Core Listener Implementation

#### Step 1.1: Create `positionUpdateListener.js`

**Location**: `backend/src/listeners/positionUpdateListener.js`

**Responsibilities**:
- Watch `PositionUpdate` events from bonding curve
- Parse event data
- Update `infofi_markets.current_probability_bps`
- Handle errors gracefully
- Log all operations

**Key Functions**:
```javascript
export async function startPositionUpdateListener(
  bondingCurveAddress,
  bondingCurveAbi,
  logger
)
```

**Dependencies**:
- `publicClient` from `viemClient.js` (Viem polling)
- `db` from `supabaseClient.js` (Supabase operations)
- Fastify logger for structured logging

#### Step 1.2: Implement Database Update Method

**Location**: `backend/shared/supabaseClient.js`

**New Method**:
```javascript
async updateMarketProbability(seasonId, playerAddress, newProbabilityBps) {
  // Find market by (season_id, player_address, market_type='WINNER_PREDICTION')
  // Update current_probability_bps and updated_at
  // Return updated record or null if not found
}
```

**SQL Logic**:
```sql
UPDATE infofi_markets
SET current_probability_bps = $1,
    updated_at = NOW()
WHERE season_id = $2 
  AND player_address = $3
  AND market_type = 'WINNER_PREDICTION'
  AND is_active = true
RETURNING *;
```

**Behavior**:
- Only updates if market exists (market creation is separate concern)
- Silently skips if no market found (player hasn't crossed 1% threshold yet)
- Returns updated record for logging/verification

#### Step 1.3: Integrate with Server Startup

**Location**: `backend/fastify/server.js`

**Changes**:
1. Import `startPositionUpdateListener`
2. Import `bondingCurveAbi`
3. Add listener startup in `startListeners()` function
4. Query `season_contracts` to get bonding curve addresses
5. Start listener for each active season
6. Store unwatch functions for cleanup

**Pseudocode**:
```javascript
async function startListeners() {
  // ... existing SeasonStarted listener ...
  
  // Start PositionUpdate listeners for all active seasons
  const activeSeasons = await db.getActiveSeasonContracts();
  
  for (const season of activeSeasons) {
    try {
      const unwatch = await startPositionUpdateListener(
        season.bonding_curve_address,
        bondingCurveAbi,
        app.log
      );
      unwatchPositionUpdates.push(unwatch);
    } catch (error) {
      app.log.error(`Failed to start PositionUpdate listener for season ${season.season_id}`, error);
    }
  }
}
```

---

### Phase 2: Real-Time Updates & Monitoring

#### Step 2.1: SSE Integration (Future)

**Concept**: Stream probability updates to frontend in real-time

**Implementation**:
- Emit event from listener when probability updates
- SSE endpoint broadcasts to connected clients
- Frontend receives live odds updates

**Not in initial scope** - implement after core listener works

#### Step 2.2: Monitoring & Observability

**Logging Strategy**:
- Log every event received with timestamp
- Log every database update with old/new values
- Log errors with full context
- Track update frequency per season

**Metrics to Track**:
- Events received per second
- Database update latency
- Update frequency distribution
- Error rates

---

## Technical Decisions & Rationale

### Decision 1: Remove On-Chain Position Storage

**Why**:
- Player holdings already tracked in `BondingCurve.playerTickets` and `RaffleToken` balances
- Computing probabilities off-chain saves ~50k gas per transaction
- Enables real-time updates without on-chain state mutations
- Simplifies contract architecture (no `RafflePositionTracker` needed)

**Trade-off**:
- Requires backend listener to stay in sync
- Adds dependency on backend availability for odds accuracy
- Mitigated by: polling-based listener with historical event scanning

### Decision 2: Use Viem's watchContractEvent with Polling

**Why**:
- HTTP polling works without WebSocket infrastructure
- Viem handles ABI encoding/decoding automatically
- Type-safe event parsing
- Consistent with existing `seasonStartedListener` pattern

**Configuration**:
```javascript
{
  address: bondingCurveAddress,
  abi: bondingCurveAbi,
  eventName: 'PositionUpdate',
  pollingInterval: 3000,  // 3-second polling
  onLogs: async (logs) => { /* handle events */ }
}
```

### Decision 3: Update Only If Market Exists

**Why**:
- Markets are created separately when player crosses 1% threshold
- Avoids creating orphaned market records
- Keeps concerns separated (market creation vs probability updates)

**Implication**:
- Early position updates (before 1% threshold) are ignored
- Once market created, all subsequent updates are captured
- No data loss (initial probability captured at market creation time)

### Decision 4: Use season_contracts Table for Address Discovery

**Why**:
- Bonding curve addresses already stored by `seasonStartedListener`
- Avoids hardcoding addresses
- Supports unlimited concurrent seasons
- Single source of truth for contract addresses

**Query Pattern**:
```javascript
const activeSeasons = await db.getActiveSeasonContracts();
// Returns: [{ season_id, bonding_curve_address, ... }, ...]
```

---

## Database Schema Requirements

### Existing Table: season_contracts

Already populated by `seasonStartedListener`. Used to discover bonding curve addresses.

```sql
SELECT season_id, bonding_curve_address 
FROM season_contracts 
WHERE is_active = true;
```

### Target Table: infofi_markets

**Columns Used**:
- `season_id` - Join key
- `player_address` - Join key
- `market_type` - Filter for 'WINNER_PREDICTION'
- `current_probability_bps` - Update target
- `updated_at` - Update timestamp
- `is_active` - Filter for active markets

**Index Recommendation**:
```sql
CREATE INDEX idx_infofi_markets_season_player 
ON infofi_markets(season_id, player_address, market_type) 
WHERE is_active = true;
```

---

## Error Handling & Resilience

### Error Scenarios

**Scenario 1: Market Doesn't Exist Yet**
- **Cause**: Player position updated before crossing 1% threshold
- **Handling**: Silently skip (log at debug level)
- **Impact**: No data loss; probability captured when market created

**Scenario 2: Database Connection Fails**
- **Cause**: Supabase unavailable
- **Handling**: Log error, continue listening, retry on next event
- **Impact**: Temporary stale odds; recovers when DB available

**Scenario 3: Event Parsing Fails**
- **Cause**: Malformed event data
- **Handling**: Log error with full event data, continue listening
- **Impact**: Single event skipped; subsequent events processed normally

**Scenario 4: Listener Stops Polling**
- **Cause**: Network timeout or Viem internal error
- **Handling**: Heartbeat monitor detects, logs warning, can trigger restart
- **Impact**: Odds become stale until listener restarted

### Resilience Strategies

1. **Historical Event Scanning** (Future Enhancement)
   - On startup, scan last N blocks for missed events
   - Catch up on any events missed during downtime
   - Ensures consistency after restarts

2. **Heartbeat Monitoring** (Future Enhancement)
   - Track last event timestamp
   - Alert if no events for X minutes
   - Trigger manual investigation

3. **Graceful Degradation**
   - Listener failure doesn't crash server
   - Frontend shows "odds updating" state if stale
   - Manual refresh available

---

## Testing Strategy

### Unit Tests

**Test 1: Event Parsing**
- Mock `PositionUpdate` event
- Verify correct extraction of all fields
- Test edge cases (BigInt conversion, address normalization)

**Test 2: Database Update**
- Mock Supabase response
- Verify correct SQL query construction
- Test update with existing market
- Test skip when market doesn't exist

**Test 3: Error Handling**
- Mock database failure
- Verify error logged, listener continues
- Mock malformed event
- Verify error logged, listener continues

### Integration Tests

**Test 1: End-to-End Event Flow**
- Deploy contracts to Anvil
- Buy tickets to trigger `PositionUpdate`
- Verify event captured by listener
- Verify database updated correctly

**Test 2: Multiple Seasons**
- Create 2+ seasons
- Buy tickets in each
- Verify listener handles all seasons
- Verify correct season_id in updates

**Test 3: High-Frequency Updates**
- Rapid buy/sell transactions
- Verify all events captured
- Verify database consistency
- Check for race conditions

---

## Implementation Timeline

### Week 1: Core Implementation
- Day 1-2: Create `positionUpdateListener.js`
- Day 2-3: Add `updateMarketProbability()` to `supabaseClient.js`
- Day 3-4: Integrate with `server.js`
- Day 4-5: Unit tests

### Week 2: Testing & Deployment
- Day 1-2: Integration tests on Anvil
- Day 2-3: Staging deployment
- Day 3-4: Production deployment
- Day 4-5: Monitoring & observability setup

---

## Success Criteria

✅ Listener successfully captures `PositionUpdate` events  
✅ Database updates within 5 seconds of event emission  
✅ No events lost during normal operation  
✅ Graceful error handling (listener doesn't crash)  
✅ Supports unlimited concurrent seasons  
✅ Probability updates visible on frontend within 10 seconds  
✅ Zero data corruption or orphaned records  

---

## Future Enhancements (Post-MVP)

1. **Historical Event Scanning**
   - Recover missed events on startup
   - Ensure consistency after downtime

2. **Real-Time SSE Streaming**
   - Push probability updates to frontend
   - Sub-second odds updates

3. **Probability Aggregation**
   - Track probability history per player
   - Compute volatility metrics
   - Detect manipulation attempts

4. **Cross-Season Analytics**
   - Compare probability patterns across seasons
   - Identify player behavior trends
   - Optimize market creation thresholds

---

## Appendix: Code Structure

### File Organization

```
backend/
├── src/
│   ├── listeners/
│   │   ├── seasonStartedListener.js      (existing)
│   │   └── positionUpdateListener.js     (NEW)
│   ├── lib/
│   │   └── viemClient.js                 (existing)
│   └── abis/
│       └── BondingCurveAbi.js            (existing)
├── shared/
│   ├── supabaseClient.js                 (modify: add updateMarketProbability)
│   └── redisClient.js                    (existing)
└── fastify/
    └── server.js                         (modify: integrate listener)
```

### Dependencies

**Already Available**:
- `viem` - Event watching and ABI parsing
- `@supabase/supabase-js` - Database operations
- `fastify` - Server framework

**No New Dependencies Required**

---

## Questions for Review

1. Should we implement historical event scanning now or defer to Phase 2?
2. What's the acceptable latency for probability updates? (5s? 10s? 30s?)
3. Should we track probability history per player for analytics?
4. What monitoring/alerting thresholds should we set?

