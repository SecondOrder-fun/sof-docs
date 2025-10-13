# InfoFi Probability Calculation Audit

## Problem Statement

**Observed Behavior:**
- Player has 15,000 tickets
- Total tickets sold: 15,000
- Expected win probability: 100%
- **Actual displayed probability: 15%**

## System Architecture Review

### 1. Smart Contract Layer ✅ CORRECT

#### InfoFiMarketFactory.sol (Lines 138-139)
```solidity
uint256 oldBps = (oldTickets * 10000) / totalTickets;
uint256 newBps = (newTickets * 10000) / totalTickets;
```

**Calculation:** `(15000 * 10000) / 15000 = 10000 basis points = 100%` ✅

#### Raffle.sol getSeasonDetails() (Line 371)
```solidity
totalTickets = state.totalTickets;  // Returns actual sold tickets, not max supply
```

**Verified onchain:** Returns `15000` (0x3a98) ✅

#### InfoFiPriceOracle.sol
**Verified onchain for marketId=0:**
- `raffleProbabilityBps`: 10000 (100%) ✅
- `marketSentimentBps`: 0 (0%) ✅  
- `hybridPriceBps`: 7000 (70%) ✅
- Formula: `(70% × 100%) + (30% × 0%) = 70%` ✅

**Smart contracts are calculating correctly!**

### 2. Frontend Layer ❌ ISSUE FOUND

#### Data Flow Analysis

**Step 1: Market Discovery** (`src/services/onchainInfoFi.js`)
```javascript
// listSeasonWinnerMarkets() returns:
{
  id: "0",
  seasonId: 1,
  raffle_id: 1,
  player: "0x7099...79C8",
  market_type: "WINNER_PREDICTION"
  // ❌ NO current_probability field!
}
```

**Step 2: Market Display** (`src/components/infofi/InfoFiMarketCard.jsx`)

The component tries multiple sources for probability (lines 290-320):

1. **bps.hybrid** - From `useHybridPriceLive(market?.id)` hook
2. **market?.current_probability** - From market object (undefined!)
3. **directProbabilityBps** - Calculated from player balance / total supply
4. **directBps** - Inline calculation

#### Issue #1: Missing current_probability in Market Data

`listSeasonWinnerMarkets()` doesn't fetch probability from anywhere. The market object has NO probability data.

#### Issue #2: Direct Calculation May Use Wrong Total

Lines 276-288 in InfoFiMarketCard.jsx:
```javascript
const directProbabilityBps = React.useMemo(() => {
  if (!isWinnerPrediction || playerTicketBalance?.data == null || totalTicketSupply == null) {
    return null;
  }
  if (totalTicketSupply === 0n) return 0;

  const playerBalance = playerTicketBalance.data;  // 15000
  const totalSupply = totalTicketSupply;           // Should be 15000

  const bps = Number((playerBalance * 10000n) / totalSupply);
  return Math.max(0, Math.min(10000, bps));
}, [isWinnerPrediction, playerTicketBalance?.data, totalTicketSupply]);
```

**This SHOULD calculate correctly IF `totalTicketSupply` is 15000.**

#### Issue #3: totalTicketSupply Source

Lines 145-156:
```javascript
const totalTicketSupply = React.useMemo(() => {
  if (!seasonDetailsQuery?.data) return null;
  const raw = Array.isArray(seasonDetailsQuery.data)
    ? seasonDetailsQuery.data[3] ?? seasonDetailsQuery.data.totalTickets
    : seasonDetailsQuery.data?.totalTickets;
  // ...
}, [seasonDetailsQuery?.data]);
```

This reads from `getSeasonDetails()` which returns 15000 ✅

### 3. Root Cause Hypothesis

**Possible Issues:**

A. **Oracle not being queried** - `useHybridPriceLive` may not be working
B. **Calculation using wrong value** - Some code path is using maxSupply instead of totalTickets
C. **Display bug** - Calculation is correct but display shows wrong value
D. **Stale data** - Frontend cached old value when there were 100k max supply

## Verification Steps Needed

1. ✅ Check smart contract calculations - CORRECT
2. ✅ Check onchain oracle values - CORRECT (70% hybrid)
3. ❌ Check what `useHybridPriceLive` returns
4. ❌ Check what `totalTicketSupply` actually contains in browser
5. ❌ Check what `playerTicketBalance` actually contains
6. ❌ Check which code path in `percent` useMemo is being used

## Next Steps

1. Add logging to InfoFiMarketCard to see actual values
2. Check if `useHybridPriceLive` is fetching oracle data
3. Verify `seasonDetailsQuery` is returning correct data
4. Check browser DevTools to see actual rendered values

## Expected Fix

Once we identify which value is wrong, the fix will likely be:

**Option A:** Ensure `useHybridPriceLive` properly fetches from oracle
**Option B:** Fix `listSeasonWinnerMarkets` to include probability from oracle
**Option C:** Ensure `totalTicketSupply` uses actual sold tickets not max supply
**Option D:** Fix calculation priority in `percent` useMemo

## Documentation References

- `instructions/project-requirements.md` Line 78: "Raffle position probability is computed on-chain from Raffle totals (tickets/totalTickets)"
- `instructions/data-schema.md` Line 41: `winProbabilityBps: number; // (ticketCount / totalTickets) * 10000`
- Both confirm: **Use actual sold tickets, not max supply**
