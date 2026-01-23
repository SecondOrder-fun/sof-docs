# InfoFi Position Recording Implementation

## Summary

Implemented complete position recording system for InfoFi prediction markets with historical sync capability.

## Database Changes

### Schema Updates

1. **Added `tx_hash` column to `infofi_positions`**

   - Enables idempotent position recording
   - Prevents duplicate entries from same transaction
   - Indexed for fast lookups

2. **Added `player_id` column to `infofi_positions`**

   - Foreign key to `players` table
   - Auto-populated by database trigger
   - NULL if player doesn't exist yet (flexible design)

3. **Added sync tracking to `infofi_markets`**

   - `last_synced_block`: Tracks last blockchain block synced
   - `last_synced_at`: Timestamp of last successful sync
   - Enables incremental historical syncing

4. **Created `user_market_positions` view**
   - Aggregates positions by user/market/outcome
   - Includes player information via JOIN
   - Provides totals, averages, and trade counts

### Final Schema

```sql
-- Positions table (raw trade data)
CREATE TABLE infofi_positions (
    id BIGSERIAL PRIMARY KEY,
    market_id BIGINT NOT NULL REFERENCES infofi_markets(id),
    user_address VARCHAR(42) NOT NULL,
    player_id BIGINT REFERENCES players(id),  -- Auto-populated
    outcome VARCHAR(10) NOT NULL,              -- 'YES', 'NO', etc.
    amount NUMERIC NOT NULL,
    price NUMERIC NULL,
    tx_hash VARCHAR(66) NULL,                  -- For idempotency
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Aggregated view
CREATE VIEW user_market_positions AS
SELECT
    market_id,
    user_address,
    player_id,
    outcome,
    COUNT(*) as num_trades,
    SUM(amount) as total_amount,
    AVG(price) as avg_price,
    MIN(created_at) as first_trade_at,
    MAX(created_at) as last_trade_at,
    array_agg(tx_hash) as tx_hashes
FROM infofi_positions
GROUP BY market_id, user_address, player_id, outcome;
```

## Backend Implementation

### 1. Position Service (`backend/src/services/infoFiPositionService.js`)

**Core Functions:**

- `recordPosition()` - Record individual trade (idempotent via tx_hash)
- `syncMarketPositions()` - Backfill historical trades from blockchain
- `syncAllActiveMarkets()` - Sync all active markets
- `getUserPositions()` - Get user's raw positions
- `getAggregatedPosition()` - Get aggregated totals per outcome
- `getNetPosition()` - Get net YES/NO position for binary markets
- `isUserHedging()` - Detect if user has multiple outcome positions

**Key Features:**

- Idempotent recording via tx_hash check
- Historical sync from last_synced_block
- Automatic price calculation (amountIn/amountOut)
- Error handling without crashing
- Support for multi-outcome markets

### 2. Trade Listener Updates (`backend/src/listeners/tradeListener.js`)

**Changes:**

- Import `infoFiPositionService`
- After sentiment update, record position to database
- Extract correct Trade event args: `trader`, `buyYes`, `amountIn`, `amountOut`
- Graceful error handling (log but don't crash listener)

**Flow:**

```
Trade Event → Update Sentiment → Record Position → Log Success/Error
```

### 3. API Endpoints (`backend/fastify/routes/infoFiRoutes.js`)

**New Endpoints:**

```
GET  /api/infofi/positions/:userAddress
     - Get all positions for user
     - Query: ?marketId=X (optional filter)

GET  /api/infofi/positions/:userAddress/aggregated
     - Get aggregated positions by outcome
     - Query: ?marketId=X (required)

GET  /api/infofi/positions/:userAddress/net
     - Get net YES/NO position
     - Query: ?marketId=X (required)
     - Returns: { yes, no, net, isHedged, numTradesYes, numTradesNo }

POST /api/infofi/markets/:fpmmAddress/sync
     - Sync historical trades for specific market
     - Query: ?fromBlock=X (optional)
```

## Features

### ✅ Real-Time Recording

- Every trade automatically recorded to database
- Triggered by Trade event listener
- Idempotent (won't duplicate if event replays)

### Historical Sync

- **Automatic on startup** - Backfills all active markets when backend starts
- Incremental sync from last_synced_block
- Manual trigger via API endpoint (per-market)
- Catches missed trades after crashes/downtime

### Multi-Position Support

- User can hold YES and NO simultaneously (hedging)
- Multiple trades on same outcome tracked separately
- Aggregation view for total positions
- Ready for multi-outcome markets (OUTCOME_A, OUTCOME_B, etc.)

### ✅ Player Integration

- Automatic linking to players table via trigger
- Works even if player doesn't exist yet
- View includes player data for easy queries

## Usage Examples

### Query User Positions

```javascript
// Get all positions
const positions = await fetch("/api/infofi/positions/0x2146...d5ab");

// Get aggregated totals
const aggregated = await fetch(
  "/api/infofi/positions/0x2146...d5ab/aggregated?marketId=23"
);

// Get net position
const net = await fetch("/api/infofi/positions/0x2146...d5ab/net?marketId=23");
// Returns: { yes: '125', no: '50', net: '75', isHedged: true }
```

### Sync Historical Trades

```javascript
// Automatic on startup - no action needed!
// Backend syncs all active markets when it starts

// Manual sync for specific market (if needed)
await fetch("/api/infofi/markets/0xe1b4.../sync", { method: "POST" });
```

### Direct Service Usage

```javascript
import { infoFiPositionService } from "./services/infoFiPositionService.js";

// Record position
await infoFiPositionService.recordPosition({
  fpmmAddress: "0xe1b4...",
  trader: "0x2146...",
  buyYes: false,
  amountIn: 100000000000000000000n, // 100 SOF
  amountOut: 250000000000000000000n, // 250 shares
  txHash: "0x7256...",
});

// Check if hedging
const isHedging = await infoFiPositionService.isUserHedging("0x2146...", 23);
```

## Testing

### Manual Test

1. Place bet on frontend
2. Check database: `SELECT * FROM infofi_positions ORDER BY created_at DESC LIMIT 1;`
3. Verify tx_hash, player_id, outcome, amount, price

### Sync Test

1. Stop backend
2. Place multiple bets
3. Restart backend (sync happens automatically)
4. Check logs for "Historical sync complete" message
5. Verify all trades recorded in database

## Next Steps

- [ ] Create unit tests for position service
- [ ] Add integration tests for Trade listener
- [ ] Monitor production for missed trades
- [ ] Add metrics/alerting for sync failures

## Notes

- Console.error statements in service are intentional (backend logging)
- Trigger auto-populates player_id, no code changes needed
- View automatically updates when positions table changes
- Multi-outcome markets supported by design (just use different outcome strings)
