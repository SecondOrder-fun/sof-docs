# Phase 2 Quick Start - Event Listener Enhancement

**Status**: Ready to implement  
**Estimated Time**: 2-3 hours  
**Priority**: High - Enables real-time oracle updates

---

## What Phase 2 Does

Connects the oracle service to actual blockchain events so that:
- When players buy/sell tickets → Oracle updates raffle probability
- When traders place bets → Oracle updates market sentiment
- When markets are created → FPMM address is stored in database

---

## Three Tasks to Complete

### Task 1: Enhance positionUpdateListener.js

**File**: `backend/src/listeners/positionUpdateListener.js`

**What to add**:
```javascript
import { oracleCallService } from '../services/oracleCallService.js';

// In the event handler:
const fpmmAddress = await db.getFpmmAddress(seasonId, player);
if (fpmmAddress) {
  const result = await oracleCallService.updateRaffleProbability(
    fpmmAddress,
    newProbabilityBps,
    logger
  );
  if (!result.success) {
    logger.warn(`Oracle update failed: ${result.error}`);
  }
}
```

**Key points**:
- Get fpmmAddress from database for each player/season
- Call oracle service with new probability
- Log failures but don't crash

---

### Task 2: Create tradeListener.js (NEW)

**File**: `backend/src/listeners/tradeListener.js`

**What to do**:
1. Listen to Trade events from SimpleFPMM contracts
2. Calculate sentiment from trade volume/direction
3. Call oracle service to update market sentiment

**Template**:
```javascript
import { publicClient } from '../lib/viemClient.js';
import { oracleCallService } from '../services/oracleCallService.js';
import SimpleFpmmAbi from '../abis/SimpleFpmmAbi.js';

export async function startTradeListener(fpmmAddresses, logger) {
  for (const fpmmAddress of fpmmAddresses) {
    publicClient.watchContractEvent({
      address: fpmmAddress,
      abi: SimpleFpmmAbi,
      eventName: 'Trade',
      onLogs: async (logs) => {
        // Calculate sentiment from trade
        const sentiment = calculateSentiment(logs);
        
        // Update oracle
        await oracleCallService.updateMarketSentiment(
          fpmmAddress,
          sentiment,
          logger
        );
      }
    });
  }
}
```

---

### Task 3: Modify marketCreatedListener.js

**File**: `backend/src/listeners/marketCreatedListener.js`

**What to add**:
```javascript
// When MarketCreated event detected:
const fpmmAddress = event.args.fpmmAddress;

// Store in database
await db.updateInfoFiMarket(seasonId, player, {
  fpmm_address: fpmmAddress
});

logger.info(`Market created: ${fpmmAddress}`);
```

**Key points**:
- Extract fpmmAddress from event
- Store in `infofi_markets.fpmm_address` column
- This enables Task 1 to find the address

---

## Database Query Needed

Add to `backend/shared/supabaseClient.js`:

```javascript
async getFpmmAddress(seasonId, playerAddress) {
  const { data, error } = await supabase
    .from('infofi_markets')
    .select('fpmm_address')
    .eq('season_id', seasonId)
    .eq('player_id', playerAddress)
    .eq('market_type', 'WINNER_PREDICTION')
    .single();

  if (error) return null;
  return data?.fpmm_address || null;
}
```

---

## Testing Phase 2

After implementing:

1. **Deploy contracts**: `npm run anvil:deploy`
2. **Start backend**: `npm run dev:backend`
3. **Create season and buy tickets**
4. **Check logs for**: `✅ Oracle updated` or `❌ Oracle update failed`
5. **Verify database**: Check `infofi_markets.fpmm_address` is populated

---

## Success Criteria

- ✅ positionUpdateListener calls oracle on position updates
- ✅ tradeListener calls oracle on trade events
- ✅ marketCreatedListener stores fpmmAddress in database
- ✅ All oracle calls logged with emoji indicators
- ✅ Failures don't crash the system
- ✅ Database queries work correctly

---

## Files to Modify/Create

| File | Action | Priority |
|------|--------|----------|
| `backend/src/listeners/positionUpdateListener.js` | MODIFY | HIGH |
| `backend/src/listeners/tradeListener.js` | CREATE | HIGH |
| `backend/src/listeners/marketCreatedListener.js` | MODIFY | HIGH |
| `backend/shared/supabaseClient.js` | ADD METHOD | HIGH |
| `backend/fastify/server.js` | MODIFY | MEDIUM |

---

## Estimated Effort

- positionUpdateListener: 30 min
- tradeListener: 45 min
- marketCreatedListener: 20 min
- Database method: 15 min
- Testing: 30 min

**Total**: ~2-3 hours

---

## Next After Phase 2

Phase 3: Database & Monitoring
- Create `oracle_call_history` table
- Create `adminAlertService.js`
- Send alerts on failures

Phase 4: Integration & Testing
- Update `server.js` initialization
- Create integration tests
- Create unit tests
