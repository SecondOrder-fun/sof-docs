# InfoFi Integration Audit Report

**Date:** 2025-01-12
**Status:** CRITICAL - Markets not propagating to Supabase

## Executive Summary

The InfoFi market creation system is **partially working** but has a **critical gap** in the data flow from smart contracts to the database. Markets are being created on-chain but are NOT being propagated to Supabase, breaking the frontend display and prediction market functionality.

## Root Cause Analysis

### The Problem Chain

1. ✅ **Smart Contract Events ARE Firing** - `InfoFiMarketFactory.MarketCreated` events are emitted when positions cross 1% threshold
2. ✅ **Backend Listener IS Running** - `infofiListener.js` is started and watching for events
3. ❌ **Database Schema Mismatch** - The listener expects `player_id` but the schema has `player_address`
4. ❌ **Missing `raffle_id` Column** - Schema uses `season_id` but listener tries to insert `raffle_id`
5. ❌ **No Error Visibility** - Database insertion failures are silently caught and logged

---

## Detailed Audit by Layer

### Layer 1: Smart Contracts ✅ WORKING

#### Raffle.sol (Lines 288-298)
```solidity
// Emit InfoFi position update and call factory
emit PositionUpdate(seasonId, participant, oldTickets, newTicketsLocal, state.totalTickets);
if (infoFiFactory != address(0)) {
    IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(
        seasonId,
        participant,
        oldTickets,
        newTicketsLocal,
        state.totalTickets
    );
}
```

**Status:** ✅ Correctly emits `PositionUpdate` event and calls InfoFi factory

#### SOFBondingCurve.sol (Lines 244-258)
```solidity
// Trigger InfoFi market creation when crossing 1% upward
if (infoFiFactory != address(0) && newBps >= 100 && oldBps < 100) {
    // Best-effort call; do not revert the buy on external failure
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(
        raffleSeasonId,
        msg.sender,
        oldTickets,
        newTickets,
        totalTickets
    ) {
        // no-op
    } catch {
        // swallow error to avoid impacting user buy
    }
}
```

**Status:** ✅ Correctly triggers InfoFi factory on 1% threshold crossing

#### InfoFiMarketFactory.sol (Lines 150-172)
```solidity
// Upward crossing of 1% threshold and not created yet
if (newBps >= 100 && oldBps < 100 && !winnerPredictionCreated[seasonId][player]) {
    winnerPredictionCreated[seasonId][player] = true;

    // Create a per-player prediction market on the shared InfoFiMarket
    string memory q = "Will this player win the raffle season?";
    address marketAddr = address(infoFiMarket);
    uint256 createdMarketId = 0;
    try infoFiMarket.createMarket(seasonId, player, q, betToken) returns (uint256 marketIdU256) {
        winnerPredictionMarkets[seasonId][player] = marketAddr;
        winnerPredictionMarketIds[seasonId][player] = marketIdU256;
        createdMarketId = marketIdU256;
    } catch {
        // If creation fails, still mark as created to avoid spamming
        winnerPredictionMarkets[seasonId][player] = address(0);
        winnerPredictionMarketIds[seasonId][player] = 0;
    }
    _seasonPlayers[seasonId].push(player);

    emit MarketCreated(seasonId, player, WINNER_PREDICTION, createdMarketId, newBps, marketAddr);
}
```

**Status:** ✅ Emits `MarketCreated` event with correct parameters

**Event Signature:**
```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 indexed marketType,
    uint256 marketId,
    uint256 probabilityBps,
    address marketAddress
);
```

---

### Layer 2: Backend Event Listener ⚠️ PARTIALLY WORKING

#### infofiListener.js (Lines 23-74)

**Current Implementation:**
```javascript
const unwatch = client.watchContractEvent({
  address: chain.infofiFactory,
  abi: InfoFiMarketFactoryAbi,
  eventName: 'MarketCreated',
  onLogs: async (logs) => {
    for (const log of logs) {
      try {
        const seasonId = Number(log.args.seasonId);
        const player = String(log.args.player);
        const probabilityBps = Number(log.args.probabilityBps);
        const MARKET_TYPE = 'WINNER_PREDICTION';

        // Ensure we have a canonical player id for downstream lookups
        const playerId = await db.getOrCreatePlayerIdByAddress(player);

        // Idempotency: skip if exists
        const exists = await db.hasInfoFiMarket(seasonId, playerId, MARKET_TYPE);
        if (exists) {
          continue;
        }

        const market = await db.createInfoFiMarket({
          season_id: seasonId,
          player_id: playerId,           // ❌ PROBLEM: Schema expects player_address
          player_address: player,
          market_type: MARKET_TYPE,
          initial_probability_bps: probabilityBps,
          current_probability_bps: probabilityBps,
          is_active: true,
          is_settled: false
        });
```

**Problems Identified:**

1. **Schema Mismatch - `player_id` vs `player_address`**
   - Listener tries to insert `player_id` (bigint reference to players table)
   - Schema expects `player_address` (varchar(42)) as the primary identifier
   - The `players` table may not exist or may not be properly referenced

2. **Column Name Mismatch - `season_id` vs `raffle_id`**
   - Listener correctly uses `season_id`
   - But some backend code references `raffle_id` (see supabaseClient.js line 123)
   - This creates confusion and potential query failures

3. **Silent Failure**
   - Errors are caught and logged but not surfaced to monitoring
   - No alerts when market creation fails
   - Frontend has no way to know markets are missing

---

### Layer 3: Database Schema ❌ CRITICAL ISSUES

#### Current Schema (2025-08-19_infofi_schema.sql)

```sql
create table if not exists infofi_markets (
  id bigserial primary key,
  season_id bigint not null,              -- ✅ Correct
  player_address varchar(42),             -- ⚠️ Should be NOT NULL
  market_type varchar(50) not null,       -- ✅ Correct
  contract_address varchar(42),           -- ✅ Correct
  initial_probability_bps integer not null, -- ✅ Correct
  current_probability_bps integer not null, -- ✅ Correct
  is_active boolean default true,         -- ✅ Correct
  is_settled boolean default false,       -- ✅ Correct
  settlement_time timestamptz,            -- ✅ Correct
  winning_outcome boolean,                -- ✅ Correct
  created_at timestamptz default now(),   -- ✅ Correct
  updated_at timestamptz default now()    -- ✅ Correct
);
```

**Problems:**

1. **Missing `player_id` column** - Listener tries to insert this but it doesn't exist
2. **Missing `raffle_id` column** - Some backend code references this
3. **`player_address` should be NOT NULL** - Every market must have a player
4. **No foreign key to `players` table** - If we're using `player_id`, we need the FK
5. **No foreign key to `raffles` table** - `season_id` should reference `raffles(id)`

#### Missing `players` Table

The listener calls `db.getOrCreatePlayerIdByAddress(player)` but the `players` table schema is not in the migration file. This table needs to exist:

```sql
create table if not exists players (
  id bigserial primary key,
  address varchar(42) not null unique,
  created_at timestamptz default now()
);
```

---

### Layer 4: Backend Database Service ⚠️ INCONSISTENT

#### supabaseClient.js Issues

**Line 53-66: `getInfoFiMarketByComposite`**
```javascript
async getInfoFiMarketByComposite(raffleId, playerId, marketType) {
  const { data, error } = await this.client
    .from('infofi_markets')
    .select('*')
    .eq('raffle_id', raffleId)  // ❌ Should be 'season_id'
    .eq('player_id', playerId)   // ❌ Should be 'player_address'
    .eq('market_type', marketType)
    .limit(1)
    .maybeSingle();
```

**Line 119-127: `getInfoFiMarketsByRaffleId`**
```javascript
async getInfoFiMarketsByRaffleId(raffleId) {
  const { data, error } = await this.client
    .from('infofi_markets')
    .select('*')
    .eq('raffle_id', raffleId)  // ❌ Should be 'season_id'
    .order('created_at', { ascending: false });
```

**Inconsistencies:**

1. Uses `raffle_id` instead of `season_id` in queries
2. Uses `player_id` instead of `player_address` in queries
3. These queries will ALWAYS fail because columns don't exist
4. No error handling for column mismatch

---

### Layer 5: Frontend ⚠️ WORKING BUT RECEIVING EMPTY DATA

#### useInfoFiMarkets.js (Lines 10-14)
```javascript
async function fetchMarkets() {
  const res = await fetch('/api/infofi/markets')
  if (!res.ok) throw new Error(`Failed to fetch markets (${res.status})`)
  const data = await res.json()
  return data?.markets || {}
}
```

**Status:** ✅ Correctly fetches from backend API

**Problem:** Backend returns empty `{}` because:
1. No markets in database (insertion fails)
2. Query uses wrong column names (`raffle_id` instead of `season_id`)

---

## Critical Path Analysis

### Current Flow (BROKEN)

```
1. User buys tickets → SOFBondingCurve.buyTokens()
2. Position crosses 1% → InfoFiMarketFactory.onPositionUpdate()
3. Market created on-chain → emit MarketCreated(seasonId, player, ...)
4. Backend listener catches event → infofiListener.js
5. ❌ FAILS: db.createInfoFiMarket() tries to insert player_id (doesn't exist)
6. ❌ FAILS: Error caught and logged, no retry
7. Frontend queries /api/infofi/markets
8. ❌ FAILS: Query uses raffle_id (doesn't exist)
9. Frontend receives empty {} object
10. No markets displayed to users
```

### Expected Flow (FIXED)

```
1. User buys tickets → SOFBondingCurve.buyTokens()
2. Position crosses 1% → InfoFiMarketFactory.onPositionUpdate()
3. Market created on-chain → emit MarketCreated(seasonId, player, ...)
4. Backend listener catches event → infofiListener.js
5. ✅ db.createInfoFiMarket() inserts with correct columns
6. ✅ Market row created in infofi_markets table
7. ✅ Pricing cache initialized in market_pricing_cache table
8. Frontend queries /api/infofi/markets
9. ✅ Query uses season_id and returns markets
10. ✅ Markets displayed to users with live odds
```

---

## Environment Configuration Issues

### Missing Environment Variables

The backend needs these addresses configured in `.env`:

```bash
# Local Anvil
INFOFI_FACTORY_ADDRESS_LOCAL=0x...
INFOFI_ORACLE_ADDRESS_LOCAL=0x...

# Testnet
INFOFI_FACTORY_ADDRESS_TESTNET=0x...
INFOFI_ORACLE_ADDRESS_TESTNET=0x...
```

**Current Status:** Unknown (can't read .env due to .gitignore)

**Risk:** If these are not set, the listener won't start at all

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     SMART CONTRACTS (ON-CHAIN)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SOFBondingCurve.buyTokens()                                   │
│         │                                                       │
│         ├─→ Position crosses 1% threshold                      │
│         │                                                       │
│         └─→ InfoFiMarketFactory.onPositionUpdate()            │
│                    │                                            │
│                    ├─→ Create market on-chain                  │
│                    │                                            │
│                    └─→ emit MarketCreated(                     │
│                           seasonId,                             │
│                           player,                               │
│                           marketType,                           │
│                           marketId,                             │
│                           probabilityBps,                       │
│                           marketAddress                         │
│                        )                                        │
│                                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Event emitted to blockchain
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND EVENT LISTENER                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  infofiListener.js                                             │
│         │                                                       │
│         ├─→ client.watchContractEvent()                        │
│         │                                                       │
│         ├─→ Parse event: seasonId, player, probabilityBps      │
│         │                                                       │
│         ├─→ ❌ db.getOrCreatePlayerIdByAddress(player)         │
│         │      └─→ PROBLEM: players table may not exist        │
│         │                                                       │
│         ├─→ ❌ db.createInfoFiMarket({                         │
│         │         season_id: seasonId,                         │
│         │         player_id: playerId,  ← WRONG COLUMN         │
│         │         player_address: player,                      │
│         │         ...                                          │
│         │      })                                              │
│         │      └─→ FAILS: player_id column doesn't exist       │
│         │                                                       │
│         └─→ Error caught and logged (silent failure)           │
│                                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Database insertion fails
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                         SUPABASE DATABASE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  infofi_markets table                                          │
│  ├─ id (bigserial)                                             │
│  ├─ season_id (bigint) ✅                                      │
│  ├─ player_address (varchar) ✅                                │
│  ├─ ❌ NO player_id column                                     │
│  ├─ ❌ NO raffle_id column                                     │
│  └─ ...                                                         │
│                                                                 │
│  ❌ NO markets inserted (insertion fails)                      │
│                                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Empty result set
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      BACKEND API ROUTES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GET /api/infofi/markets                                       │
│         │                                                       │
│         ├─→ ❌ db.getInfoFiMarketsByRaffleId(raffleId)        │
│         │      └─→ FAILS: raffle_id column doesn't exist       │
│         │                                                       │
│         └─→ Returns { markets: {} } (empty)                    │
│                                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Empty response
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                          FRONTEND                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  useInfoFiMarkets()                                            │
│         │                                                       │
│         ├─→ fetch('/api/infofi/markets')                       │
│         │                                                       │
│         ├─→ Receives { markets: {} }                           │
│         │                                                       │
│         └─→ ❌ No markets to display                           │
│                                                                 │
│  InfoFiMarketCard                                              │
│         │                                                       │
│         └─→ ❌ Shows "No markets available"                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Impact Assessment

### User Impact: CRITICAL

- ❌ Users cannot see prediction markets for players
- ❌ Users cannot place bets on InfoFi markets
- ❌ No market odds displayed
- ❌ No arbitrage opportunities visible
- ❌ Core product feature completely broken

### Business Impact: HIGH

- ❌ InfoFi markets are a key differentiator
- ❌ Revenue from prediction market fees = $0
- ❌ User engagement severely limited
- ❌ Cannot demonstrate product to investors/users

### Technical Debt: MEDIUM

- ⚠️ Schema inconsistencies will cause future bugs
- ⚠️ Silent failures make debugging difficult
- ⚠️ No monitoring/alerting for critical failures

---

## Recommended Fixes (Priority Order)

### Priority 1: Database Schema Fixes (CRITICAL)

**File:** `backend/src/db/migrations/2025-08-19_infofi_schema.sql`

**Changes Needed:**

1. Add `players` table if it doesn't exist
2. Decide on primary identifier: `player_id` (FK) OR `player_address` (direct)
3. Add `raffle_id` column OR standardize on `season_id` everywhere
4. Add proper foreign keys
5. Make `player_address` NOT NULL

**Recommendation:** Use `player_address` directly (simpler, no FK dependency)

### Priority 2: Backend Listener Fixes (CRITICAL)

**File:** `backend/src/services/infofiListener.js`

**Changes Needed:**

1. Remove `player_id` from insertion
2. Use `player_address` as primary identifier
3. Add better error handling and logging
4. Add retry logic for transient failures
5. Add monitoring/alerting for failures

### Priority 3: Backend Database Service Fixes (HIGH)

**File:** `backend/shared/supabaseClient.js`

**Changes Needed:**

1. Replace all `raffle_id` with `season_id`
2. Replace all `player_id` with `player_address`
3. Update query methods to match schema
4. Add proper error messages for debugging

### Priority 4: Odds Calculation Updates (HIGH)

**Files:**
- `backend/src/services/raffleListener.js`
- `backend/shared/pricingService.js`

**Changes Needed:**

1. Ensure odds updates use correct column names
2. Verify hybrid pricing calculations
3. Add real-time SSE updates for odds changes

### Priority 5: Frontend Robustness (MEDIUM)

**Files:**
- `src/hooks/useInfoFiMarkets.js`
- `src/components/infofi/InfoFiMarketCard.jsx`

**Changes Needed:**

1. Add better error handling for empty markets
2. Add loading states
3. Add retry logic
4. Add user-friendly error messages

---

## Testing Requirements

### Unit Tests

1. ✅ Test `db.createInfoFiMarket()` with correct schema
2. ✅ Test `db.getInfoFiMarketsBySeasonId()` returns markets
3. ✅ Test `db.getInfoFiMarketByComposite()` with player_address
4. ✅ Test event listener parsing logic

### Integration Tests

1. ✅ Deploy contracts to local Anvil
2. ✅ Buy tickets to cross 1% threshold
3. ✅ Verify `MarketCreated` event emitted
4. ✅ Verify market inserted into database
5. ✅ Verify frontend displays market
6. ✅ Verify odds update in real-time

### E2E Tests

1. ✅ Full raffle lifecycle with InfoFi markets
2. ✅ Multiple players crossing threshold
3. ✅ Odds updates as positions change
4. ✅ Market settlement after raffle ends

---

## Monitoring & Observability

### Metrics to Track

1. **Market Creation Rate** - Markets created per hour
2. **Event Processing Latency** - Time from event to DB insertion
3. **Failed Insertions** - Count of database errors
4. **Empty Market Queries** - Frontend queries returning no data
5. **Listener Health** - Is the event listener running?

### Alerts to Configure

1. ❌ **CRITICAL:** No markets created in 1 hour
2. ❌ **CRITICAL:** Database insertion failure rate > 10%
3. ⚠️ **WARNING:** Event processing latency > 30 seconds
4. ⚠️ **WARNING:** Listener restart detected

---

## Conclusion

The InfoFi integration is **fundamentally sound** in architecture but has **critical implementation gaps** that prevent it from working end-to-end. The primary issues are:

1. **Schema mismatch** between listener expectations and actual database structure
2. **Column naming inconsistency** (`season_id` vs `raffle_id`, `player_id` vs `player_address`)
3. **Silent failures** that hide the problem from developers
4. **Missing foreign keys** and table relationships

**Estimated Fix Time:** 4-6 hours for complete resolution

**Risk Level:** LOW - Fixes are straightforward and well-understood

**Next Steps:** See INFOFI_INTEGRATION_WORKPLAN.md
