# InfoFi Integration Fix Workplan

**Date:** 2025-01-12 (Updated)  
**Estimated Time:** 5-7 hours  
**Priority:** CRITICAL

## Quick Reference

**Root Causes:**
1. **CRITICAL:** Position updates only fire on initial 1% threshold crossing (breaks real-time odds)
2. Schema mismatch between backend listener and database structure  

**Primary Fixes:**
1. **Phase 0:** Remove `oldBps < 100` condition from SOFBondingCurve (DONE ✅)
2. Remove `player_id` from listener insertions
3. Change all `raffle_id` references to `season_id`
4. Use `player_address` as primary identifier
5. Add proper error handling and monitoring

---

## Phase 0: Fix Position Update Logic (30 min) ✅ COMPLETED

### Critical Issue Discovered

The bonding curve only called `InfoFiMarketFactory.onPositionUpdate()` when crossing the 1% threshold **for the first time**. This broke the entire real-time odds system because:

- Markets created at initial probability (e.g., 1.5%)
- Subsequent position increases (e.g., to 3.5%, 5%) never triggered updates
- Odds remained frozen at initial value
- Prediction markets showed stale data
- Arbitrage opportunities invisible

### Fix Applied

**File:** `contracts/src/curve/SOFBondingCurve.sol`

**Changes:**

1. **Line 244-258 (buyTokens):** Removed `oldBps < 100` condition
2. **Line 336-351 (sellTokens):** Added factory notification for sells

**Before:**
```solidity
// Only called on first crossing
if (infoFiFactory != address(0) && newBps >= 100 && oldBps < 100) {
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(...) {
```

**After:**
```solidity
// Always called on ANY position change
if (infoFiFactory != address(0)) {
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(...) {
```

### Benefits

✅ Markets receive updates on EVERY buy/sell  
✅ Real-time odds stay current  
✅ Matches Raffle.sol behavior (which always calls factory)  
✅ Factory handles idempotency internally  
✅ Supports position tracking below 1% threshold  
✅ Enables future analytics and dynamic thresholds

### Testing

**New Test File:** `contracts/test/InfoFiPositionUpdates.t.sol`

**Test Coverage:**
- Multiple position increases above threshold
- Sells trigger position updates
- Multiple players with changing probabilities
- Continuous updates after multiple crossings
- Dropping below and re-crossing threshold
- Gas cost consistency

**Run Tests:**
```bash
cd contracts
forge test --match-contract InfoFiPositionUpdates -vvv
```

### Verification

**Expected Behavior:**
```
Buy 1: 0.5% → ProbabilityUpdated(0, 50) ✅
Buy 2: 1.5% → ProbabilityUpdated(50, 150) ✅ (market created)
Buy 3: 3.5% → ProbabilityUpdated(150, 350) ✅ (market updated)
Buy 4: 5.0% → ProbabilityUpdated(350, 500) ✅ (market updated)
```

---

## Phase 1: Database Schema Fixes (60 min)

### 1.1 Create Players Table

**File:** `backend/src/db/migrations/2025-01-12_add_players_table.sql`

```sql
create table if not exists players (
  id bigserial primary key,
  address varchar(42) not null unique,
  created_at timestamptz default now()
);
create index idx_players_address on players (address);
```

### 1.2 Fix InfoFi Markets Schema

**File:** `backend/src/db/migrations/2025-01-12_fix_infofi_markets.sql`

```sql
-- Make player_address NOT NULL
alter table infofi_markets alter column player_address set not null;

-- Add comment for clarity
comment on column infofi_markets.season_id is 'References raffle season (same as raffle_id)';
```

---

## Phase 2: Backend Listener Fixes (45 min)

### 2.1 Fix infofiListener.js

**File:** `backend/src/services/infofiListener.js` (Lines 45-54)

**Change:**
```javascript
// REMOVE player_id from insertion
const market = await db.createInfoFiMarket({
  season_id: seasonId,
  player_address: player,  // ✅ Use player_address directly
  market_type: MARKET_TYPE,
  initial_probability_bps: probabilityBps,
  current_probability_bps: probabilityBps,
  is_active: true,
  is_settled: false
});
```

### 2.2 Fix hasInfoFiMarket Call

**File:** `backend/src/services/infofiListener.js` (Line 40)

**Change:**
```javascript
// Use player_address instead of playerId
const exists = await db.hasInfoFiMarket(seasonId, player, MARKET_TYPE);
```

---

## Phase 3: Database Service Fixes (60 min)

### 3.1 Fix getInfoFiMarketByComposite

**File:** `backend/shared/supabaseClient.js` (Lines 53-66)

**Change:**
```javascript
async getInfoFiMarketByComposite(seasonId, playerAddress, marketType) {
  const { data, error } = await this.client
    .from('infofi_markets')
    .select('*')
    .eq('season_id', seasonId)           // ✅ Changed from raffle_id
    .eq('player_address', playerAddress)  // ✅ Changed from player_id
    .eq('market_type', marketType)
    .limit(1)
    .maybeSingle();
  if (error && error.code !== 'PGRST116') throw new Error(error.message);
  return data || null;
}
```

### 3.2 Fix getInfoFiMarketsByRaffleId

**File:** `backend/shared/supabaseClient.js` (Lines 119-127)

**Change:**
```javascript
async getInfoFiMarketsBySeasonId(seasonId) {
  const { data, error } = await this.client
    .from('infofi_markets')
    .select('*')
    .eq('season_id', seasonId)  // ✅ Changed from raffle_id
    .order('created_at', { ascending: false });
  if (error) throw new Error(error.message);
  return data;
}

// Backward compatibility alias
async getInfoFiMarketsByRaffleId(raffleId) {
  return this.getInfoFiMarketsBySeasonId(raffleId);
}
```

### 3.3 Update Callers

**Files:**
- `backend/src/services/raffleListener.js` (Line 66)
- `backend/fastify/routes/infoFiRoutes.js` (Line 116)

**Changes:**
```javascript
// raffleListener.js
const market = await db.getInfoFiMarketByComposite(seasonId, player, marketType);

// infoFiRoutes.js
markets = await db.getInfoFiMarketsBySeasonId(raffleId);
```

---

## Phase 4: Testing (90 min)

### 4.1 Database Migration Test

```bash
# Apply migrations
psql $DATABASE_URL -f backend/src/db/migrations/2025-01-12_add_players_table.sql
psql $DATABASE_URL -f backend/src/db/migrations/2025-01-12_fix_infofi_markets.sql

# Verify schema
psql $DATABASE_URL -c "\d infofi_markets"
```

### 4.2 Integration Test

```bash
# Start Anvil
anvil --gas-limit 30000000

# Deploy contracts
cd contracts && forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --private-key $PRIVATE_KEY --broadcast

# Start backend
cd backend && npm run dev

# Buy tickets to trigger market creation
cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" 2000 3500000000000000000000 --rpc-url http://127.0.0.1:8545 --private-key $PRIVATE_KEY

# Verify market in database
psql $DATABASE_URL -c "SELECT * FROM infofi_markets ORDER BY created_at DESC LIMIT 1;"
```

### 4.3 Frontend Test

```bash
# Start frontend
npm run dev

# Navigate to http://localhost:5173/markets
# Verify markets are displayed
```

---

## Phase 5: Monitoring (30 min)

### 5.1 Add Health Check

**File:** `backend/fastify/routes/healthRoutes.js`

```javascript
fastify.get('/health/infofi', async (_request, reply) => {
  const markets = await db.getActiveInfoFiMarkets();
  return reply.send({
    status: 'healthy',
    marketCount: markets.length,
    timestamp: new Date().toISOString()
  });
});
```

### 5.2 Add Logging

**File:** `backend/src/services/infofiListener.js`

```javascript
logger.info({
  seasonId,
  player,
  marketId: market.id,
  probabilityBps
}, '[infofiListener] ✅ Market created successfully');
```

---

## Deployment Checklist

- [ ] Apply database migrations
- [ ] Update backend code
- [ ] Restart backend server
- [ ] Verify listener started (check logs)
- [ ] Run integration test
- [ ] Verify markets in database
- [ ] Verify frontend displays markets
- [ ] Monitor for errors

---

## Rollback Plan

If issues occur:

```bash
# Restore database from backup
pg_restore -d $DATABASE_URL backup_file.sql

# Revert code changes
git revert <commit-hash>

# Restart services
pm2 restart all
```

---

## Success Criteria

✅ Markets created on-chain appear in database within 5 seconds  
✅ Frontend displays markets with correct odds  
✅ No database insertion errors in logs  
✅ Health check returns healthy status  
✅ Real-time odds updates work via SSE

---

## Estimated Timeline

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Database Schema | 60 min | None |
| Backend Listener | 45 min | Phase 1 |
| Database Service | 60 min | Phase 1 |
| Testing | 90 min | Phases 1-3 |
| Monitoring | 30 min | Phases 1-3 |
| **Total** | **4.5 hours** | Sequential |

---

## Next Steps After Fix

1. Add unit tests for database operations
2. Add E2E tests for full market lifecycle
3. Implement market settlement logic
4. Add arbitrage opportunity detection
5. Implement cross-season market analytics
