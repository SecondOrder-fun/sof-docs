# User Pages InfoFi Update Summary

## Changes Made

Updated user-facing components to use the new FPMM-based InfoFi market system instead of the deprecated InfoFiMarket contract.

## Files Modified

### 1. `src/components/infofi/BuySellWidget.jsx`

**Changes:**
- Updated `placeBetTx` call to include required `seasonId` and `player` parameters
- Added PropTypes validation for `raffle_id`, `seasonId`, and `player` fields

**Before:**
```javascript
return placeBetTx({ 
  marketId, 
  prediction: outcome === 'YES', 
  amount: amt 
});
```

**After:**
```javascript
return placeBetTx({ 
  marketId, 
  prediction: outcome === 'YES', 
  amount: amt,
  seasonId: market?.raffle_id || market?.seasonId,
  player: market?.player
});
```

### 2. `src/components/infofi/InfoFiMarketCard.jsx`

**Already Updated** (from previous fix)
- Passes `seasonId` and `player` to `placeBetTx`
- Component correctly uses the FPMM system

## Components That Don't Need Changes

### ✅ Account Page (`src/routes/AccountPage.jsx`)
- Uses `PositionsPanel` and `ClaimCenter` components
- These components fetch data from backend API
- Backend API already handles FPMM system correctly
- **No changes needed**

### ✅ User Profile (`src/routes/UserProfile.jsx`)
- Displays user information and positions
- Uses backend API for data fetching
- **No changes needed**

### ✅ Markets Index (`src/routes/MarketsIndex.jsx`)
- Uses `useInfoFiMarkets` hook (plural) which fetches from backend API
- Renders `InfoFiMarketCard` components which are already updated
- **No changes needed**

### ✅ Positions Panel (`src/components/infofi/PositionsPanel.jsx`)
- Fetches positions from backend API
- Backend handles FPMM system
- **No changes needed**

### ✅ Claim Center (`src/components/infofi/ClaimCenter.jsx`)
- Handles claiming winnings
- Uses backend API for settled markets
- **No changes needed**

## Deprecated Components (Not Used)

### ⚠️ `src/hooks/useInfoFiMarket.js` (singular)
- Old hook that directly calls InfoFiMarket contract
- **Not used anywhere in the codebase**
- Can be deleted or marked as deprecated
- Replaced by `useInfoFiMarkets` (plural) which uses backend API

## How It Works Now

### Frontend Flow:
1. **Market Display**: `InfoFiMarketCard` shows market with live pricing from database
2. **Place Bet**: User clicks "Trade" button
3. **FPMM Lookup**: `placeBetTx` calls `InfoFiFPMMV2.getMarket(seasonId, player)` to get FPMM address
4. **Execute Trade**: Calls `SimpleFPMM.buy(buyYes, amountIn, minAmountOut)` on the FPMM contract
5. **Update UI**: React Query invalidates cache and refetches positions

### Backend Flow:
1. **Market Creation**: When player crosses 1% threshold, `InfoFiMarketFactory` creates FPMM
2. **Database Sync**: Backend listens to `MarketCreated` events and stores in database
3. **API Endpoints**: Frontend fetches market data from backend API
4. **Real-time Updates**: Backend tracks FPMM reserves and updates pricing

## Testing Checklist

- [x] `BuySellWidget` passes correct parameters to `placeBetTx`
- [x] PropTypes validation includes all required fields
- [x] `InfoFiMarketCard` already updated with correct parameters
- [ ] Test placing a bet via Account page (if BuySellWidget is used there)
- [ ] Test placing a bet via Markets page
- [ ] Verify positions display correctly after trade
- [ ] Verify claim functionality works for settled markets

## Migration Notes

### What Changed:
- **Old System**: `InfoFiMarket.placeBet(marketId, prediction, amount)`
- **New System**: `SimpleFPMM.buy(buyYes, amountIn, minAmountOut)` via `placeBetTx` wrapper

### Breaking Changes:
- `placeBetTx` now requires `seasonId` and `player` parameters
- Components must pass market data including `raffle_id`/`seasonId` and `player`

### Backward Compatibility:
- Backend API maintains same response format
- Database schema includes both old and new market types
- Frontend components gracefully handle missing data

## Related Documentation

- [InfoFi Market Purchase Fix](./INFOFI_MARKET_PURCHASE_FIX.md) - Complete technical details
- [InfoFi Integration Specs](../02-architecture/infofi-integration/) - Architecture overview
- [Data Schema](../../instructions/data-schema.md) - Database structure

## Next Steps

1. **Test End-to-End**: Create season, buy tickets, place bets via UI
2. **Remove Deprecated Code**: Delete `useInfoFiMarket.js` (singular) if confirmed unused
3. **Update E2E Tests**: Update test scripts to use FPMM system
4. **Documentation**: Update user-facing docs to reflect new system
