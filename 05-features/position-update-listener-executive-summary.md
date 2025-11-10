# PositionUpdate Event Listener - Executive Summary

**Status**: Revised Plan Ready for Implementation  
**Date**: Oct 25, 2025  
**Critical Revision**: Cascading probability updates for all players

---

## The Problem We Solved

**Original Misconception**: Update only the player who bought/sold tickets.

**Critical Realization**: When ANY player's position changes, ALL players' win probabilities change because:

```
Win Probability = (Player Tickets / Total Tickets) × 10000
```

When total supply changes, the denominator changes for everyone.

**Example**:
```
Season 1: 1000 total tickets
- Player A: 400 tickets → 40% win probability
- Player B: 300 tickets → 30% win probability  
- Player C: 300 tickets → 30% win probability

Player A buys 100 tickets → 1100 total tickets
- Player A: 500 tickets → 45.45% (CHANGED ↑)
- Player B: 300 tickets → 27.27% (CHANGED ↓)
- Player C: 300 tickets → 27.27% (CHANGED ↓)

All three markets must update!
```

---

## Solution Architecture

### Data Flow

```
┌─────────────────────────────────────────────────────────┐
│ Player buys/sells tickets                               │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ SOFBondingCurve emits PositionUpdate event              │
│ (seasonId, player, oldTickets, newTickets,             │
│  totalTickets, probabilityBps)                         │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Backend Listener (Viem polling)                         │
│ - Captures event                                        │
│ - Extracts: seasonId, totalTickets                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Query All Participants                                  │
│ Raffle.getParticipants(seasonId) → [A, B, C]          │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Fetch Each Player's Position                            │
│ For each player: Raffle.getParticipantPosition()       │
│ Result: [{player: A, ticketCount: 500},               │
│          {player: B, ticketCount: 300},               │
│          {player: C, ticketCount: 300}]               │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Calculate All Probabilities                             │
│ A: (500 / 1100) × 10000 = 4545 bps                    │
│ B: (300 / 1100) × 10000 = 2727 bps                    │
│ C: (300 / 1100) × 10000 = 2727 bps                    │
│ Sum: 10000 ✓                                           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Batch Update Database                                   │
│ UPDATE infofi_markets                                   │
│ SET current_probability_bps = CASE                      │
│   WHEN player_address = A THEN 4545                     │
│   WHEN player_address = B THEN 2727                     │
│   WHEN player_address = C THEN 2727                     │
│ WHERE season_id = 1                                     │
│   AND market_type = 'WINNER_PREDICTION'                │
│   AND is_active = true                                  │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Frontend Receives Update (via SSE - future)             │
│ All 3 players see live odds update                      │
└─────────────────────────────────────────────────────────┘
```

---

## Implementation Overview

### Three Components to Build

#### 1. Event Listener: `positionUpdateListener.js`

**Responsibility**: Listen for `PositionUpdate` events and orchestrate updates

**Key Logic**:
```javascript
// When event received:
1. Extract seasonId and totalTickets from event
2. Query Raffle.getParticipants(seasonId)
3. For each participant, fetch their ticket count
4. Call db.updateAllPlayerProbabilities(seasonId, totalTickets, positions)
5. Log: "Updated X markets in season Y"
```

**Error Handling**:
- Market doesn't exist → Skip silently (log debug)
- Contract read fails → Log error, continue with other seasons
- Database update fails → Log error, retry on next event

#### 2. Database Method: `updateAllPlayerProbabilities()`

**Responsibility**: Update all player probabilities in database

**Algorithm**:
```javascript
for each player in playerPositions:
  newBps = Math.round((player.ticketCount / totalTickets) * 10000)
  UPDATE infofi_markets
  SET current_probability_bps = newBps, updated_at = NOW()
  WHERE season_id = seasonId
    AND player_address = player
    AND market_type = 'WINNER_PREDICTION'
    AND is_active = true
  
  if market exists:
    updatedCount++
  else:
    skip (player hasn't crossed 1% threshold yet)

return updatedCount
```

**SQL**:
```sql
UPDATE infofi_markets
SET current_probability_bps = $1,
    updated_at = NOW()
WHERE season_id = $2 
  AND player_address = $3
  AND market_type = 'WINNER_PREDICTION'
  AND is_active = true;
```

#### 3. Server Integration: `server.js`

**Responsibility**: Start listener for each active season on startup

**Logic**:
```javascript
// On server startup:
1. Query season_contracts table for all active seasons
2. For each season:
   - Get bonding_curve_address
   - Start positionUpdateListener(bondingCurveAddress, ...)
   - Store unwatch function
3. On shutdown:
   - Call all unwatch functions
   - Stop all listeners gracefully
```

---

## Implementation Checklist

### Phase 1: Core Implementation (Week 1)

- [ ] Create `backend/src/listeners/positionUpdateListener.js`
  - [ ] Import dependencies (publicClient, db, logger)
  - [ ] Implement event listening logic
  - [ ] Implement participant fetching
  - [ ] Implement error handling
  - [ ] Add comprehensive logging

- [ ] Add `updateAllPlayerProbabilities()` to `backend/shared/supabaseClient.js`
  - [ ] Implement probability calculation
  - [ ] Implement batch update logic
  - [ ] Handle missing markets gracefully
  - [ ] Return update count

- [ ] Integrate with `backend/fastify/server.js`
  - [ ] Import listener and ABI
  - [ ] Query season_contracts on startup
  - [ ] Start listener for each season
  - [ ] Store unwatch functions
  - [ ] Add graceful shutdown

- [ ] Unit Tests
  - [ ] Test probability calculation
  - [ ] Test batch update logic
  - [ ] Test error scenarios

### Phase 2: Testing & Deployment (Week 2)

- [ ] Integration Tests
  - [ ] End-to-end multi-player update
  - [ ] Rapid sequential transactions
  - [ ] Large player base (50+ players)
  - [ ] Race condition testing

- [ ] Performance Testing
  - [ ] Measure latency (target: <5 seconds)
  - [ ] Measure database query time
  - [ ] Measure contract read time
  - [ ] Identify bottlenecks

- [ ] Deployment
  - [ ] Staging environment testing
  - [ ] Production deployment
  - [ ] Monitoring setup
  - [ ] Alert configuration

---

## Key Technical Decisions

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **Fetch all positions on every event** | Simple, reliable, correct | ~50k gas per player (off-chain cost) |
| **Use Viem polling** | HTTP works without WebSocket; consistent pattern | 3-second latency |
| **Update only if market exists** | Avoid orphaned records; markets created separately | Early updates ignored (no data loss) |
| **Batch database updates** | Efficient; single transaction | Slightly more complex SQL |
| **Use season_contracts table** | Already populated; supports unlimited seasons | One more DB query per listener |

---

## Performance Characteristics

### Gas & Latency

| Operation | Cost | Time |
|-----------|------|------|
| Event emission | ~5k gas | Included in tx |
| Contract read (getParticipants) | Off-chain | ~100ms |
| Contract read per player (getParticipantPosition) | Off-chain | ~50ms per player |
| Database update per market | Off-chain | ~10ms per market |
| **Total latency (10 players)** | **Off-chain** | **~600ms** |
| **Total latency (50 players)** | **Off-chain** | **~2.5s** |
| **Total latency (100 players)** | **Off-chain** | **~5s** |

**User Cost**: Zero (all operations are off-chain)

### Scalability

- **MVP Target**: 100 players per season
- **Acceptable Latency**: <5 seconds
- **Future Optimization**: Caching (reduces to <500ms for 100 players)

---

## Success Criteria

✅ Captures all `PositionUpdate` events  
✅ **Updates ALL players' probabilities on each event**  
✅ Database updates within 5 seconds  
✅ Zero events lost during normal operation  
✅ Graceful error handling (listener never crashes)  
✅ Supports unlimited concurrent seasons  
✅ Probability updates visible on frontend within 10 seconds  
✅ **Probabilities sum to 10000 (100%) after each update**  
✅ Scales to 100+ players per season  
✅ Zero data corruption or orphaned records  

---

## Dependencies

**Already Available** (no new packages needed):
- `viem` - Event watching & ABI parsing
- `@supabase/supabase-js` - Database operations
- `fastify` - Server framework

---

## Future Enhancements

### Phase 2: Optimization

1. **Position Caching**
   - Cache player positions in memory
   - Update cache on events
   - Reduces contract reads by 90%
   - Latency: <500ms for 100 players

2. **Batch SQL Updates**
   - Use CASE statement for single query
   - Reduces database round-trips
   - Latency: <100ms for 100 players

3. **Active Player Tracking**
   - Only update players with markets
   - Reduces updates from N to M (M << N)
   - Scales to 1000+ players

### Phase 3: Real-Time

1. **SSE Broadcasting**
   - Stream updates to frontend in real-time
   - Sub-second odds updates
   - Live probability visualization

2. **Probability History**
   - Track probability changes over time
   - Volatility metrics
   - Behavior analytics

3. **Cross-Season Analytics**
   - Identify player behavior trends
   - Optimize market creation thresholds
   - Predict participation patterns

---

## Files to Create/Modify

```
backend/
├── src/
│   ├── listeners/
│   │   ├── seasonStartedListener.js      (existing)
│   │   └── positionUpdateListener.js     (NEW)
│   ├── lib/
│   │   └── viemClient.js                 (existing)
│   └── abis/
│       ├── RaffleAbi.js                  (existing)
│       └── BondingCurveAbi.js            (existing)
├── shared/
│   ├── supabaseClient.js                 (MODIFY: add updateAllPlayerProbabilities)
│   └── redisClient.js                    (existing)
└── fastify/
    └── server.js                         (MODIFY: integrate listener)
```

---

## Questions & Answers

**Q: Why update all players?**  
A: Win probability depends on total supply. When total changes, all probabilities change.

**Q: How do we know which players to update?**  
A: Query `Raffle.getParticipants(seasonId)` to get all players in the season.

**Q: What if a player doesn't have a market yet?**  
A: We silently skip them. Markets are created separately when crossing 1% threshold.

**Q: Won't fetching all positions be slow?**  
A: It's off-chain, so no cost to users. For <100 players, <5 seconds is acceptable. We can optimize later with caching.

**Q: What about race conditions?**  
A: Each event triggers independent updates. Database handles concurrent updates correctly with proper indexes.

**Q: How do we verify correctness?**  
A: After each update, probabilities should sum to 10000 (100%). We can add validation in tests.

---

## Ready for Implementation

This plan is comprehensive, technically sound, and ready for immediate implementation. All architectural decisions are documented with clear rationale.

**Next Steps**:
1. Review plan with team
2. Approve implementation approach
3. Begin Phase 1 development
4. Test on Anvil with multi-player scenarios

