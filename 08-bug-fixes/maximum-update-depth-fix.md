# Maximum Update Depth Error Fix

## Problem

React error: "Maximum update depth exceeded" causing infinite re-render loop in InfoFiMarketCard component.

## Root Cause

Two issues in the probability display code:

### Issue 1: Unnecessary State Updates

The `useEffect` in `InfoFiMarketCard.jsx` (lines 51-100) was calling `setBps()` on every execution, even when values hadn't changed. This caused:

1. React Query polls oracle every 10 seconds
2. `priceData` changes (new object reference)
3. `useEffect` runs
4. `setBps()` called with same values
5. Component re-renders
6. Loop repeats infinitely

### Issue 2: Non-Memoized Return Object

The `useHybridPriceLive` hook was returning a new object on every render:

```javascript
return {
  data: data ? { /* new object every time */ } : null,
  isLive: !isLoading && !error,
  source: 'blockchain'
};
```

This caused the `priceData` dependency in InfoFiMarketCard's `useEffect` to change on every render.

## Solution

### Fix 1: Conditional State Updates

Added checks to only update state when values actually change:

```javascript
setBps((prev) => {
  // Calculate new values...
  
  // Only update if values actually changed
  if (prev.hybrid === hybrid && prev.raffle === raffle && prev.market === marketProb) {
    return prev; // Return same object reference = no re-render
  }
  
  return { hybrid, raffle, market: marketProb };
});
```

### Fix 2: Memoize Return Object

Used `useMemo` to stabilize the return object:

```javascript
const priceData = useMemo(() => {
  if (!data) return null;
  return {
    marketId,
    hybridPriceBps: data.hybridPriceBps,
    raffleProbabilityBps: data.raffleProbabilityBps,
    marketSentimentBps: data.marketSentimentBps,
    lastUpdated: data.lastUpdate,
  };
}, [data, marketId]);

return {
  data: priceData, // Stable reference
  isLive: !isLoading && !error,
  source: 'blockchain'
};
```

## Files Changed

1. **`src/hooks/useHybridPriceLive.js`**
   - Added `useMemo` import
   - Memoized the data object to prevent unnecessary re-renders
   - Return object now has stable reference when data unchanged

2. **`src/components/infofi/InfoFiMarketCard.jsx`**
   - Added conditional checks before calling `setBps()`
   - Only updates state when values actually change
   - Returns previous state object if no changes (prevents re-render)

## Impact

- ✅ Eliminates infinite render loop
- ✅ Improves performance (fewer unnecessary renders)
- ✅ Maintains correct probability display
- ✅ React Query polling still works (updates when oracle changes)

## Testing

1. Navigate to `/markets` page
2. Should load without errors
3. Probability should display correctly (70%)
4. No console errors about maximum update depth
5. Component should update when oracle values change (every 10s)

## Related Issues

This fix complements the probability calculation fix from 2025-10-13. Both issues stemmed from the same root cause: replacing backend streams with direct blockchain queries required careful handling of React re-render cycles.

## Prevention

**Best Practices Applied:**

1. **Always check if state actually changed** before calling setState
2. **Memoize objects** returned from hooks to prevent unnecessary re-renders
3. **Use stable references** in useEffect dependencies
4. **Test with React DevTools Profiler** to catch excessive re-renders

## Date

2025-10-13
