# Comprehensive Fix Plan: InfoFi Probability Display Issue

## Executive Summary

**Problem:** Market displays 15% win probability when player has 100% (15k/15k tickets)

**Root Cause:** Frontend cannot access oracle data because:
1. `useHybridPriceLive` depends on non-existent backend SSE/WebSocket streams
2. Direct calculation fallback is not being used correctly
3. Market data from `listSeasonWinnerMarkets` doesn't include probability

**Solution:** Query oracle directly from blockchain, similar to how we fixed market discovery

## Detailed Analysis

### What's Working ✅

1. **Smart Contracts** - All calculations correct
   - InfoFiMarketFactory: `(15000 * 10000) / 15000 = 10000 bps`
   - InfoFiPriceOracle: Stores `raffleProbabilityBps = 10000`, `hybridPriceBps = 7000`
   - Raffle: Returns `totalTickets = 15000`

2. **Onchain Data** - All values correct
   - Player balance: 15000 tickets
   - Total tickets: 15000
   - Oracle probability: 100% raffle, 70% hybrid

### What's Broken ❌

1. **useHybridPriceLive Hook** - Depends on backend streams that don't exist
   - Tries WebSocket connection (no backend)
   - Falls back to SSE stream (no backend)
   - Returns null data

2. **Market Discovery** - Doesn't fetch probability
   - `listSeasonWinnerMarkets` returns basic info only
   - No `current_probability` field
   - Frontend has no initial probability value

3. **Fallback Calculation** - May have bugs
   - Complex priority logic in `percent` useMemo
   - Multiple code paths, unclear which executes
   - Possible stale data or wrong variable usage

## Proposed Solution

### Phase 1: Add Oracle Query to Market Discovery ⭐ PRIMARY FIX

**File:** `src/services/onchainInfoFi.js`

**Change:** Enhance `listSeasonWinnerMarkets` to fetch probability from oracle

```javascript
export async function listSeasonWinnerMarkets({ seasonId, networkKey = 'LOCAL' }) {
  const byEvents = await listSeasonWinnerMarketsByEvents({ seasonId, networkKey });
  
  // NEW: Fetch probability from oracle for each market
  const marketsWithProbability = await Promise.all(
    byEvents.map(async (market) => {
      try {
        const priceData = await readOraclePrice({ 
          marketId: market.id, 
          networkKey 
        });
        
        return {
          ...market,
          current_probability: priceData.hybridPriceBps, // Add hybrid price
          raffle_probability: priceData.raffleProbabilityBps, // Add raffle component
          market_sentiment: priceData.marketSentimentBps, // Add market component
        };
      } catch (error) {
        // If oracle read fails, return market without probability
        return market;
      }
    })
  );
  
  return marketsWithProbability;
}
```

**Impact:**
- ✅ Markets have probability data immediately
- ✅ No backend dependency
- ✅ Uses correct oracle values
- ✅ Minimal code changes

### Phase 2: Replace useHybridPriceLive with Direct Oracle Query

**File:** `src/hooks/useHybridPriceLive.js`

**Change:** Query oracle directly instead of backend streams

```javascript
import { useQuery } from '@tanstack/react-query';
import { readOraclePrice } from '@/services/onchainInfoFi';
import { getStoredNetworkKey } from '@/lib/wagmi';

export function useHybridPriceLive(marketId) {
  const networkKey = getStoredNetworkKey();
  
  const { data, isLoading, error } = useQuery({
    queryKey: ['oraclePrice', marketId, networkKey],
    queryFn: () => readOraclePrice({ marketId, networkKey }),
    enabled: !!marketId,
    staleTime: 5_000,
    refetchInterval: 10_000, // Poll every 10 seconds
  });

  return {
    data: data ? {
      marketId,
      hybridPriceBps: data.hybridPriceBps,
      raffleProbabilityBps: data.raffleProbabilityBps,
      marketSentimentBps: data.marketSentimentBps,
      lastUpdated: data.lastUpdate,
    } : null,
    isLive: !isLoading && !error,
    source: 'blockchain'
  };
}
```

**Impact:**
- ✅ Real-time updates via polling
- ✅ No backend dependency
- ✅ Consistent with market discovery fix
- ✅ Uses proven `readOraclePrice` function

### Phase 3: Simplify InfoFiMarketCard Calculation

**File:** `src/components/infofi/InfoFiMarketCard.jsx`

**Change:** Simplify `percent` calculation to prioritize oracle data

```javascript
const percent = React.useMemo(() => {
  // Priority 1: Hybrid price from oracle (via useHybridPriceLive)
  if (bps.hybrid != null && bps.hybrid > 0) {
    return (bps.hybrid / 100).toFixed(1);
  }
  
  // Priority 2: Current probability from market data (from listSeasonWinnerMarkets)
  if (market?.current_probability != null) {
    const bpsValue = normalizeBps(market.current_probability);
    if (bpsValue != null && bpsValue > 0) {
      return (bpsValue / 100).toFixed(1);
    }
  }
  
  // Priority 3: Direct calculation from player balance
  if (directProbabilityBps != null && directProbabilityBps > 0) {
    return (directProbabilityBps / 100).toFixed(1);
  }
  
  // Fallback: 0%
  return '0.0';
}, [bps.hybrid, market?.current_probability, directProbabilityBps, normalizeBps]);
```

**Impact:**
- ✅ Clear priority order
- ✅ Uses oracle data when available
- ✅ Falls back gracefully
- ✅ Easier to debug

## Implementation Steps

### Step 1: Update onchainInfoFi.js ⭐ DO THIS FIRST

1. Modify `listSeasonWinnerMarkets` to call `readOraclePrice` for each market
2. Add probability fields to returned market objects
3. Handle errors gracefully (return market without probability if oracle fails)

### Step 2: Update useHybridPriceLive.js

1. Remove WebSocket/SSE dependencies
2. Replace with React Query + `readOraclePrice`
3. Keep same return interface for backward compatibility

### Step 3: Simplify InfoFiMarketCard.jsx

1. Simplify `percent` calculation logic
2. Add clear priority comments
3. Remove redundant fallback paths

### Step 4: Test

1. Verify market shows 100% for sole player
2. Verify updates when second player joins
3. Verify hybrid formula (70% raffle + 30% market)
4. Verify fallback when oracle unavailable

## Testing Checklist

- [ ] Single player (100% probability) displays correctly
- [ ] Two players (50%/50%) display correctly  
- [ ] Probability updates when player buys more tickets
- [ ] Probability updates when player sells tickets
- [ ] Hybrid price reflects market sentiment after bets
- [ ] Fallback works if oracle query fails
- [ ] Performance acceptable (no excessive RPC calls)

## Rollback Plan

If issues arise:
1. Revert `listSeasonWinnerMarkets` changes
2. Revert `useHybridPriceLive` changes
3. Markets will show no probability (better than wrong probability)
4. Investigate and fix before re-deploying

## Success Criteria

✅ Market displays 100% when player has all tickets
✅ Probability updates in real-time (within 10 seconds)
✅ No backend dependencies
✅ Uses correct oracle values
✅ Graceful degradation if oracle unavailable

## Related Files

- `src/services/onchainInfoFi.js` - Market discovery & oracle queries
- `src/hooks/useHybridPriceLive.js` - Real-time price updates
- `src/components/infofi/InfoFiMarketCard.jsx` - Display logic
- `contracts/src/infofi/InfoFiPriceOracle.sol` - Source of truth
- `contracts/src/infofi/InfoFiMarketFactory.sol` - Probability updates

## Notes

- This fix aligns with the earlier fix for market discovery (removing backend dependency)
- Oracle is the source of truth, frontend should always query it
- Polling every 10 seconds is acceptable for MVP (can optimize later with events)
- Backend SSE/WS streams can be added later as optimization, not requirement
