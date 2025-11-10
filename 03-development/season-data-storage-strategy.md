# Season Data Storage Strategy

**Date**: Oct 25, 2025  
**Context**: Storing BondingCurve and RaffleToken addresses retrieved from `SeasonStarted` events

---

## Problem Statement

When a `SeasonStarted` event fires, we need to:
1. Retrieve BondingCurve and RaffleToken addresses via `getSeasonDetails(seasonId)`
2. **Persist these addresses** for the lifetime of the raffle season
3. **Associate them with seasonId** for multi-season support
4. **Make them accessible** to other backend listeners and services

---

## Proposed Solution: Season Contracts Table

### Schema Design

Create a new `season_contracts` table to store contract addresses per season:

```sql
CREATE TABLE IF NOT EXISTS season_contracts (
  id BIGSERIAL PRIMARY KEY,
  season_id BIGINT NOT NULL UNIQUE,
  bonding_curve_address VARCHAR(42) NOT NULL,
  raffle_token_address VARCHAR(42) NOT NULL,
  raffle_address VARCHAR(42) NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for fast lookups
CREATE INDEX idx_season_contracts_season_id ON season_contracts(season_id);
CREATE INDEX idx_season_contracts_active ON season_contracts(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_season_contracts_bonding_curve ON season_contracts(bonding_curve_address);
CREATE INDEX idx_season_contracts_token ON season_contracts(raffle_token_address);
```

### Why This Design?

| Aspect | Rationale |
|--------|-----------|
| **Separate table** | Clean separation of concerns; season contracts are distinct from InfoFi markets |
| **UNIQUE season_id** | One set of addresses per season; prevents duplicates |
| **Multiple addresses** | Stores all three key addresses (raffle, bonding curve, token) together |
| **is_active flag** | Allows marking seasons as inactive without deleting historical data |
| **Timestamps** | Audit trail for when addresses were stored and last updated |
| **Indexes** | Fast lookups by season_id, bonding_curve, or token address |

---

## Data Flow

### 1. Event Listener Retrieves Data

```javascript
// In seasonStartedListener.js onLogs callback
const seasonDetails = await publicClient.readContract({
  address: raffleAddress,
  abi: raffleAbi,
  functionName: 'getSeasonDetails',
  args: [seasonId],
});

const { bondingCurve, raffleToken } = seasonDetails.config;
```

### 2. Store in Database

```javascript
// Insert into season_contracts table
await db.createSeasonContracts({
  season_id: seasonId,
  bonding_curve_address: bondingCurve,
  raffle_token_address: raffleToken,
  raffle_address: raffleAddress,
  is_active: true
});
```

### 3. Other Listeners Query the Data

```javascript
// In other listeners (e.g., bondingCurveListener)
const seasonContracts = await db.getSeasonContracts(seasonId);
const { bonding_curve_address, raffle_token_address } = seasonContracts;

// Use addresses to set up listeners
await startBondingCurveListener(bonding_curve_address, seasonId);
```

---

## Implementation: Database Service Methods

### Add to `backend/shared/supabaseClient.js`

```javascript
/**
 * Create or update season contract addresses
 */
async createSeasonContracts(data) {
  const { data: result, error } = await this.client
    .from('season_contracts')
    .upsert({
      season_id: data.season_id,
      bonding_curve_address: data.bonding_curve_address,
      raffle_token_address: data.raffle_token_address,
      raffle_address: data.raffle_address,
      is_active: data.is_active !== undefined ? data.is_active : true
    }, {
      onConflict: 'season_id'
    })
    .select()
    .single();

  if (error) throw new Error(error.message);
  return result;
}

/**
 * Get season contract addresses by season ID
 */
async getSeasonContracts(seasonId) {
  const { data, error } = await this.client
    .from('season_contracts')
    .select('*')
    .eq('season_id', seasonId)
    .single();

  if (error && error.code !== 'PGRST116') { // PGRST116 = no rows found
    throw new Error(error.message);
  }
  
  return data || null;
}

/**
 * Get all active season contracts
 */
async getActiveSeasonContracts() {
  const { data, error } = await this.client
    .from('season_contracts')
    .select('*')
    .eq('is_active', true)
    .order('created_at', { ascending: false });

  if (error) throw new Error(error.message);
  return data || [];
}

/**
 * Mark season as inactive
 */
async deactivateSeasonContracts(seasonId) {
  const { data, error } = await this.client
    .from('season_contracts')
    .update({ is_active: false })
    .eq('season_id', seasonId)
    .select()
    .single();

  if (error) throw new Error(error.message);
  return data;
}
```

---

## Integration with Event Listener

### Updated `seasonStartedListener.js`

```javascript
import { publicClient } from '../lib/viemClient.js';
import { db } from '../shared/supabaseClient.js';

export async function startSeasonStartedListener(raffleAddress, raffleAbi, logger) {
  // ... validation code ...

  const unwatch = publicClient.watchContractEvent({
    address: raffleAddress,
    abi: raffleAbi,
    eventName: 'SeasonStarted',
    onLogs: async (logs) => {
      for (const log of logs) {
        const { seasonId } = log.args;
        
        try {
          // 1. Retrieve season details from contract
          const seasonDetails = await publicClient.readContract({
            address: raffleAddress,
            abi: raffleAbi,
            functionName: 'getSeasonDetails',
            args: [seasonId],
          });

          const { bondingCurve, raffleToken } = seasonDetails.config;

          // 2. Store in database
          await db.createSeasonContracts({
            season_id: seasonId,
            bonding_curve_address: bondingCurve,
            raffle_token_address: raffleToken,
            raffle_address: raffleAddress,
            is_active: true
          });

          // 3. Log success
          logger.info(`✅ Season ${seasonId} started`);
          logger.info(`   BondingCurve: ${bondingCurve}`);
          logger.info(`   RaffleToken: ${raffleToken}`);

          // 4. Emit event for other listeners (future enhancement)
          // eventBus.emit('seasonDataRetrieved', { seasonId, bondingCurve, raffleToken });

        } catch (error) {
          logger.error(`❌ Failed to process SeasonStarted for season ${seasonId}`);
          logger.error(`   Error: ${error.message}`);
          // Continue listening; don't crash on individual failures
        }
      }
    },
    onError: (error) => {
      // ... existing error handling ...
    },
    poll: true,
    pollingInterval: 3000,
  });

  logger.info(`🎧 Listening for SeasonStarted events on ${raffleAddress}`);
  return unwatch;
}
```

---

## Multi-Season Support

### Scenario: Multiple Active Seasons

```
Season 1 (Active)
  ├─ BondingCurve: 0xAAA...
  ├─ RaffleToken: 0xBBB...
  └─ Listeners: bondingCurveListener-1, tokenListener-1

Season 2 (Active)
  ├─ BondingCurve: 0xCCC...
  ├─ RaffleToken: 0xDDD...
  └─ Listeners: bondingCurveListener-2, tokenListener-2

Season 3 (Completed)
  ├─ BondingCurve: 0xEEE...
  ├─ RaffleToken: 0xFFF...
  ├─ is_active: FALSE
  └─ Listeners: Stopped
```

### Query Active Seasons

```javascript
// Get all active seasons' contract addresses
const activeSeasons = await db.getActiveSeasonContracts();

// Start listeners for each active season
for (const season of activeSeasons) {
  await startBondingCurveListener(
    season.bonding_curve_address,
    season.season_id
  );
}
```

---

## Event Emission (Future Enhancement)

Once data is stored, emit an event for other listeners:

```javascript
// In seasonStartedListener.js after successful storage
eventBus.emit('seasonDataRetrieved', {
  seasonId,
  bondingCurve,
  raffleToken,
  raffleAddress
});

// In other listeners (e.g., bondingCurveListener.js)
eventBus.on('seasonDataRetrieved', async (data) => {
  logger.info(`Starting bonding curve listener for season ${data.seasonId}`);
  await startBondingCurveListener(data.bondingCurve, data.seasonId);
});
```

---

## Benefits of This Approach

✅ **Persistent Storage**: Addresses survive listener restarts  
✅ **Multi-Season Support**: Each season has its own contract addresses  
✅ **Fast Lookups**: Indexed queries for quick access  
✅ **Audit Trail**: Timestamps track when data was stored  
✅ **Clean Separation**: Distinct table for season contracts  
✅ **Scalable**: Handles unlimited concurrent seasons  
✅ **Event-Driven**: Can trigger downstream listeners  
✅ **Historical Data**: Completed seasons remain queryable  

---

## Migration Path

### Step 1: Create Table
```sql
-- Run via Supabase migration
CREATE TABLE IF NOT EXISTS season_contracts (
  id BIGSERIAL PRIMARY KEY,
  season_id BIGINT NOT NULL UNIQUE,
  bonding_curve_address VARCHAR(42) NOT NULL,
  raffle_token_address VARCHAR(42) NOT NULL,
  raffle_address VARCHAR(42) NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_season_contracts_season_id ON season_contracts(season_id);
CREATE INDEX idx_season_contracts_active ON season_contracts(is_active) WHERE is_active = TRUE;
```

### Step 2: Add Database Methods
Add the four methods above to `supabaseClient.js`

### Step 3: Update Listener
Modify `seasonStartedListener.js` to call `db.createSeasonContracts()`

### Step 4: Update Other Listeners
Modify bonding curve and token listeners to query `season_contracts` table

---

## Testing Strategy

### Unit Test
```javascript
// Mock db.createSeasonContracts
// Verify listener calls it with correct data
// Verify logging output
```

### Integration Test
```javascript
// 1. Deploy contracts
// 2. Start backend
// 3. Create season
// 4. Query season_contracts table
// 5. Verify addresses match deployment output
```

---

## Approval Checklist

- [ ] Schema design aligns with project patterns
- [ ] Database methods follow existing conventions
- [ ] Multi-season support is clear
- [ ] Event emission strategy is acceptable
- [ ] Testing approach is feasible
- [ ] Ready to implement

**Proceed with implementation?**
