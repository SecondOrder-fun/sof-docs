# InfoFi Position Updates - Critical Design Documentation

**Date:** 2025-01-12  
**Status:** IMPLEMENTED ✅

## Critical Design Principle

**All position changes MUST trigger InfoFi updates**, not just threshold crossings. This is fundamental to the platform's value proposition of real-time information markets.

## The Problem (Fixed)

### Original Broken Implementation

```solidity
// ❌ BROKEN: Only called on first crossing
if (infoFiFactory != address(0) && newBps >= 100 && oldBps < 100) {
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(...) {
```

**Impact:**
- Markets created at initial probability (e.g., 1.5%)
- Subsequent increases (e.g., to 3.5%, 5%) never triggered updates
- Odds frozen at initial value
- Prediction markets showed stale data
- Arbitrage opportunities invisible
- **Entire real-time odds system broken**

## The Solution (Implemented)

### Fixed Implementation

```solidity
// ✅ FIXED: Always called on ANY position change
if (infoFiFactory != address(0)) {
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(...) {
```

**Files Changed:**
- `contracts/src/curve/SOFBondingCurve.sol` (Lines 244-258, 336-351)

**Changes:**
1. Removed `oldBps < 100` condition from `buyTokens()`
2. Added factory notification to `sellTokens()`

## Data Flow

### Complete Position Update Chain

```
User Action (Buy/Sell)
    ↓
SOFBondingCurve state update
    ↓
emit PositionUpdate(seasonId, player, oldTickets, newTickets, totalTickets, probabilityBps)
    ↓
InfoFiMarketFactory.onPositionUpdate() [ALWAYS CALLED]
    ↓
Market creation (if first time crossing 1%) OR Market update (all other times)
    ↓
InfoFiPriceOracle.updateRaffleProbability(marketId, newBps)
    ↓
Hybrid price calculation (70% raffle + 30% sentiment)
    ↓
Backend listener catches event
    ↓
Database updated (infofi_markets, market_pricing_cache)
    ↓
SSE broadcast to frontend
    ↓
UI updates with new odds
```

## Expected Behavior

### Scenario: Player Buys Multiple Times

```
Buy 1: 500 tickets (0.5%)
  → PositionUpdate(0, 50)
  → Factory called, no market created (below 1%)
  → Position tracked for analytics

Buy 2: 1000 more tickets (1.5% total)
  → PositionUpdate(50, 150)
  → Factory called, market CREATED
  → Oracle updated to 150 bps (1.5%)
  → Frontend shows market with 1.5% odds

Buy 3: 2000 more tickets (3.5% total)
  → PositionUpdate(150, 350)
  → Factory called, market UPDATED
  → Oracle updated to 350 bps (3.5%)
  → Frontend shows updated 3.5% odds

Buy 4: 1500 more tickets (5.0% total)
  → PositionUpdate(350, 500)
  → Factory called, market UPDATED
  → Oracle updated to 500 bps (5.0%)
  → Frontend shows updated 5.0% odds
```

### Scenario: Player Sells Tickets

```
Current: 5000 tickets (5.0%)

Sell 1: 2000 tickets (3.0% remaining)
  → PositionUpdate(500, 300)
  → Factory called, market UPDATED
  → Oracle updated to 300 bps (3.0%)
  → Frontend shows updated 3.0% odds

Sell 2: 2000 tickets (1.0% remaining)
  → PositionUpdate(300, 100)
  → Factory called, market UPDATED
  → Oracle updated to 100 bps (1.0%)
  → Frontend shows updated 1.0% odds
```

## Benefits

✅ **Real-Time Odds**: Markets always show current win probability  
✅ **Accurate Arbitrage**: Price differences between layers visible immediately  
✅ **Analytics**: Can track positions below 1% threshold  
✅ **Consistency**: Matches Raffle.sol behavior (which always calls factory)  
✅ **Future-Proof**: Supports dynamic thresholds and advanced strategies  
✅ **Gas Efficient**: Try/catch ensures user transactions never fail  

## Testing

**Test File:** `contracts/test/InfoFiPositionUpdates.t.sol`

**Coverage:**
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

## Key Identifiers

- **season_id**: Raffle season identifier (NOT raffle_id)
- **player_address**: Ethereum address (NOT player_id)
- **probability_bps**: Win probability in basis points (100 bps = 1%)
- **market_type**: WINNER_PREDICTION, POSITION_SIZE, BEHAVIORAL

## References

- **OpenZeppelin Best Practices**: Emit events after state changes
- **Solidity Patterns**: Always notify external systems of state changes
- **InfoFi Design**: Continuous information flow for real-time markets
- **Audit Report**: `docs/INFOFI_INTEGRATION_AUDIT.md`
- **Workplan**: `docs/INFOFI_INTEGRATION_WORKPLAN.md`
