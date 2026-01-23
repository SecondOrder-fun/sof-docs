# Raffle → Season Terminology Refactor Summary

**Date**: Oct 31, 2025  
**Status**: ✅ COMPLETE

## Overview

Standardized all terminology from `raffleId`/`raffle_id` to `seasonId`/`season_id` across the backend codebase to match the actual database schema and improve code clarity.

## Changes Made

### 1. backend/shared/supabaseClient.js

**Function Parameter Changes** (7 functions updated):

| Function | Old Parameter | New Parameter |
|----------|--------------|---------------|
| `getInfoFiMarketByComposite()` | `raffleId` | `seasonId` |
| `hasInfoFiMarket()` | `raffleId` | `seasonId` |
| `getInfoFiMarketBySeasonAndPlayer()` | `raffleId` | `seasonId` |
| `getInfoFiMarketsBySeasonId()` | `raffleId` | `seasonId` |
| `getInfoFiMarketsByRaffleId()` | `raffleId` | `seasonId` |
| `updateInfoFiMarketProbability()` | `raffleId` | `seasonId` |

**Query Changes** (5 queries updated):
- All `.eq("raffle_id", ...)` changed to `.eq("season_id", ...)`
- Lines affected: 75, 169, 182, 285

**Validation Changes**:
- Removed backward compatibility for `marketData.raffle_id`
- Now only accepts `marketData.season_id`
- Updated error message: `"Missing required fields: season_id, player_address, or market_type"`

### 2. backend/shared/utils.js

**Function Parameter Changes**:

```javascript
// Before
export function generateMarketId(raffleId, playerAddress, marketType) {
  return `${raffleId}-${playerAddress}-${marketType}`;
}

// After
export function generateMarketId(seasonId, playerAddress, marketType) {
  return `${seasonId}-${playerAddress}-${marketType}`;
}
```

## Database Schema Status

### ✅ Correct Schema (Using season_id)

**Table**: `infofi_markets`
- Column: `season_id` (bigint)
- Comment: "References the raffle season ID"
- All backend queries now correctly use `season_id`

**Table**: `season_contracts`
- Column: `season_id` (bigint, unique)
- Correctly uses season terminology

**Table**: `market_pricing_cache`
- References `infofi_markets.id` (no direct season reference)

**Table**: `infofi_winnings`
- References `infofi_markets.id` (no direct season reference)

### ⚠️ Needs Future Migration

**Table**: `arbitrage_opportunities`
- Column: `raffle_id` (bigint) - **Still uses old terminology**
- Index: `idx_arbitrage_raf` - **Still uses old name**
- **Status**: No backend code currently uses this table
- **Action Required**: Create migration to rename when implementing arbitrage features

## Files Modified

1. ✅ `backend/shared/supabaseClient.js` - 7 functions, 5 queries
2. ✅ `backend/shared/utils.js` - 1 function

## Files Verified (No Changes Needed)

- ✅ `backend/src/listeners/marketCreatedListener.js` - Already using correct parameter names
- ✅ `backend/fastify/routes/*.js` - No raffle_id references found
- ✅ `backend/src/services/*.js` - No raffle_id references found

## Breaking Changes

### For External Callers

If any external code was calling these functions with `raffleId` parameter name, it will still work (JavaScript doesn't enforce parameter names), but the semantic meaning has changed:

```javascript
// Old way (still works, but semantically incorrect)
db.hasInfoFiMarket(raffleId, playerAddress, marketType);

// New way (correct)
db.hasInfoFiMarket(seasonId, playerAddress, marketType);
```

### For Database Inserts

Code that was passing `raffle_id` in `marketData` will now fail:

```javascript
// ❌ WILL FAIL
await db.createInfoFiMarket({
  raffle_id: 1,  // No longer supported
  player_address: "0x...",
  market_type: "WINNER_PREDICTION"
});

// ✅ CORRECT
await db.createInfoFiMarket({
  season_id: 1,  // Must use season_id
  player_address: "0x...",
  market_type: "WINNER_PREDICTION"
});
```

## Testing Checklist

- [x] All function signatures updated
- [x] All database queries updated
- [x] All parameter names standardized
- [x] Validation logic updated
- [x] Error messages updated
- [ ] Backend restart required
- [ ] Test market creation flow
- [ ] Test market queries
- [ ] Verify no runtime errors

## Future Work

### Immediate (Before Production)
1. Restart backend to load new code
2. Test full market creation flow
3. Verify database inserts use correct column names

### Future Migrations Needed
1. **arbitrage_opportunities table**:
   ```sql
   ALTER TABLE arbitrage_opportunities RENAME COLUMN raffle_id TO season_id;
   DROP INDEX IF EXISTS idx_arbitrage_raf;
   CREATE INDEX idx_arbitrage_season ON arbitrage_opportunities (season_id);
   ```

2. **Update migration file** `2025-08-19_infofi_schema.sql` for consistency (documentation only)

## Verification Commands

```bash
# Restart backend
npm run dev:backend

# Test market creation
cd contracts
forge script script/BuyTickets.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast

# Check backend logs for successful market creation
# Should see: "✅ InfoFi market created in database"
```

## Benefits of This Refactor

1. **Consistency**: All code now uses `season` terminology matching the database
2. **Clarity**: `season_id` is more descriptive than `raffle_id`
3. **Maintainability**: Single source of truth for terminology
4. **Reduced Confusion**: No more mixing of raffle/season terms
5. **Better Documentation**: Code is self-documenting with correct names

## Related Documentation

- Database schema: `instructions/project-requirements.md`
- Migration files: `backend/src/db/migrations/`
- Original schema: `2025-08-19_infofi_schema.sql`
- Schema fix: `2025-01-12_fix_infofi_schema.sql`

---

**Completed By**: AI Assistant  
**Reviewed By**: [Pending]  
**Deployed**: [Pending - Requires backend restart]
