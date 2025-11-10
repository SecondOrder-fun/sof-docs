# SimpleFPMM Listener Architecture - What's Missing

## Current State

`marketCreatedListener.js` currently:

- ✅ Listens for `MarketCreated` events from InfoFiMarketFactory
- ✅ Extracts fpmmAddress from event
- ✅ Stores fpmmAddress in database
- ❌ **Does NOT interact with the SimpleFPMM contract**

## The Problem

Once a SimpleFPMM market is created on-chain, the backend should be:

1. **Listening to SimpleFPMM Trade events** - Track all betting activity
2. **Reading SimpleFPMM state** - Get current prices, reserves, fees
3. **Monitoring liquidity changes** - Track LP provider activity
4. **Calculating market sentiment** - Based on trade flow and volume
5. **Updating oracle prices** - Feed sentiment to InfoFiPriceOracle

Currently, **NONE of this is happening**.

## What Should Happen

### Flow After Market Creation

```text
MarketCreated Event (fpmmAddress extracted)
    ↓
Store fpmmAddress in database
    ↓
START LISTENING TO SimpleFPMM CONTRACT:
    ├─ Trade events → Calculate sentiment → Update oracle
    ├─ LiquidityAdded events → Track LP activity
    ├─ LiquidityRemoved events → Track LP exits
    └─ Periodically read getPrices() → Cache prices
```

### SimpleFPMM Events to Listen For

**From InfoFiFPMMV2.sol (SimpleFPMM contract)**:

- **Trade** (indexed trader, buyYes, amountIn, amountOut)
  - Emitted when user buys/sells outcome tokens
  - Used to calculate market sentiment
  - Indicates direction of market belief

- **LiquidityAdded** (indexed provider, amount, lpTokens)
  - Emitted when LP adds liquidity
  - Indicates market confidence

- **LiquidityRemoved** (indexed provider, lpTokens, amount)
  - Emitted when LP removes liquidity
  - Indicates market uncertainty

### SimpleFPMM State to Read

**From InfoFiFPMMV2.sol (SimpleFPMM contract)**:

```solidity
// View functions to call
getPrices() → (yesPrice, noPrice)  // Current prices in basis points
yesReserve()                        // YES outcome reserve
noReserve()                         // NO outcome reserve
feesCollected()                     // Accumulated fees
totalSupply()                       // LP token supply
```

## Implementation Plan

### Phase 1: Create FPMM Trade Listener

**File**: `backend/src/listeners/fpmmTradeListener.js` (NEW)

```javascript
export async function startFpmmTradeListener(fpmmAddress, logger) {
  // Listen to Trade events on SimpleFPMM
  // For each trade:
  //   1. Extract: trader, buyYes, amountIn, amountOut
  //   2. Calculate sentiment adjustment
  //   3. Call oracleCallService.updateMarketSentiment()
  //   4. Store trade in database for analytics
}
```

### Phase 2: Create FPMM Price Reader

**File**: `backend/src/services/fpmmPriceService.js` (NEW)

```javascript
export async function readFpmmPrices(fpmmAddress, logger) {
  // Call getPrices() on SimpleFPMM
  // Call yesReserve() and noReserve()
  // Cache prices in database
  // Return: { yesPrice, noPrice, yesReserve, noReserve }
}
```

### Phase 3: Update Market Created Listener

**File**: `backend/src/listeners/marketCreatedListener.js` (MODIFY)

After storing fpmmAddress, immediately:

- Start FPMM trade listener for this market
- Start periodic price reader for this market
- Store unwatch functions for cleanup

### Phase 4: Integrate with Oracle

Connect FPMM sentiment to InfoFiPriceOracle:

```javascript
// When Trade event detected:
const sentiment = calculateSentiment(trade);
await oracleCallService.updateMarketSentiment(
  fpmmAddress,
  sentiment,
  logger
);
```

## Sentiment Calculation

**Current approach** (from tradeListener.js):

```javascript
// Base: 5000 bps (neutral)
// Long (YES) trades: increase sentiment
// Short (NO) trades: decrease sentiment
// Amount-based: larger trades have more impact
// Capped at 0-10000 bps range
```

**Better approach** (based on reserves):

```javascript
// Sentiment = (noReserve / (yesReserve + noReserve)) * 10000
// If more NO reserve → market bearish → lower sentiment
// If more YES reserve → market bullish → higher sentiment
// Automatically reflects current market state
```

## Database Schema Needed

### fpmm_markets table (NEW)

```sql
CREATE TABLE fpmm_markets (
  id BIGSERIAL PRIMARY KEY,
  season_id BIGINT NOT NULL,
  player_address VARCHAR(42) NOT NULL,
  fpmm_address VARCHAR(42) NOT NULL UNIQUE,
  condition_id BYTEA NOT NULL,
  yes_reserve NUMERIC(38,18),
  no_reserve NUMERIC(38,18),
  yes_price INTEGER,  -- basis points
  no_price INTEGER,   -- basis points
  total_liquidity NUMERIC(38,18),
  fees_collected NUMERIC(38,18),
  last_price_update TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### fpmm_trades table (NEW)

```sql
CREATE TABLE fpmm_trades (
  id BIGSERIAL PRIMARY KEY,
  fpmm_address VARCHAR(42) NOT NULL,
  trader_address VARCHAR(42) NOT NULL,
  buy_yes BOOLEAN NOT NULL,
  amount_in NUMERIC(38,18) NOT NULL,
  amount_out NUMERIC(38,18) NOT NULL,
  transaction_hash VARCHAR(66),
  block_number BIGINT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Critical Issues This Solves

1. **No market sentiment tracking** - Can't calculate hybrid prices without sentiment
2. **No liquidity monitoring** - Don't know if market is healthy
3. **No price discovery** - Can't read on-chain prices
4. **No trade analytics** - Can't analyze market behavior
5. **Oracle never gets updated** - Sentiment component always 0 or default

## Timeline

- **Phase 1-2**: Create FPMM listeners and price reader (~200 lines)
- **Phase 3**: Update marketCreatedListener to wire everything (~50 lines)
- **Phase 4**: Integrate with oracle (~30 lines)
- **Database**: Create tables and indexes (~100 lines SQL)

**Total**: ~380 lines of code + database schema

## Next Steps

1. Create `fpmmTradeListener.js` to listen for Trade events
2. Create `fpmmPriceService.js` to read on-chain prices
3. Update `marketCreatedListener.js` to start FPMM listeners
4. Create database tables for FPMM data
5. Wire sentiment updates to oracle
6. Add tests for FPMM listener integration
