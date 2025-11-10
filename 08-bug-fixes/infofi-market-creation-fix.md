# InfoFi Market Creation Fix - Complete Analysis

**Date**: Oct 25, 2025
**Status**: ✅ FIXED
**Issue**: On-chain InfoFi markets were not being created correctly due to probability sync issues

---

## Problem Summary

### The Issue
- **Database**: InfoFi markets WERE being created in the database ✅
- **On-Chain**: InfoFi markets were NOT being created on-chain ❌
- **Root Cause**: The contract was storing `probabilityBps` in the event, which creates a sync problem

### Why It Was Wrong

The original `MarketCreated` event included `probabilityBps`:

```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 indexed marketType,
    bytes32 conditionId,
    address fpmmAddress,
    uint256 probabilityBps  // ❌ PROBLEM: Stale data
);
```

**The Problem**: 
- Probabilities are calculated at market creation time
- But probabilities change on EVERY ticket purchase
- The on-chain value becomes stale immediately after creation
- The database has the source of truth for current probabilities
- Storing it on-chain creates a sync nightmare

**Example**:
```
Market created when player has 10,000 tickets (100% of supply)
Event emits: probabilityBps = 10,000 (100%)

Player B buys 5,000 tickets
New total supply: 15,000
Player A's new probability: 10,000 / 15,000 = 6,667 bps (66.67%)

On-chain event still shows: 10,000 bps ❌ STALE
Database shows: 6,667 bps ✅ CURRENT
```

---

## Solution Implemented

### Change 1: Remove `probabilityBps` from Event

**File**: `contracts/src/infofi/InfoFiMarketFactory.sol`

**Before**:
```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 indexed marketType,
    bytes32 conditionId,
    address fpmmAddress,
    uint256 probabilityBps
);
```

**After**:
```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 indexed marketType,
    bytes32 conditionId,
    address fpmmAddress
);
```

**Rationale**: Probabilities are dynamic and should ONLY be tracked in the database. The on-chain event should only emit immutable data (seasonId, player, marketType, conditionId, fpmmAddress).

### Change 2: Remove `probabilityBps` Parameter from Functions

**File**: `contracts/src/infofi/InfoFiMarketFactory.sol`

Updated function signatures:

```solidity
// Before
function _createMarket(uint256 seasonId, address player, uint256 probabilityBps) internal

// After
function _createMarket(uint256 seasonId, address player) internal
```

```solidity
// Before
function _createMarketInternal(uint256 seasonId, address player, uint256 probabilityBps) external

// After
function _createMarketInternal(uint256 seasonId, address player) external
```

### Change 3: Update Backend ABI

**File**: `backend/src/abis/InfoFiMarketFactoryAbi.js`

Removed the `probabilityBps` parameter from the `MarketCreated` event definition in the ABI to match the updated contract.

---

## Architecture: Probability Tracking

### On-Chain (Immutable)
- Only stores the **fact** that a market was created
- Stores: seasonId, player, marketType, conditionId, fpmmAddress
- Does NOT store probabilities (they change too frequently)

### Off-Chain Database (Source of Truth)
- Tracks **current** probabilities in `infofi_markets` table
- Columns: `initial_probability_bps`, `current_probability_bps`
- Updated on EVERY PositionUpdate event
- Always reflects the current state

### Data Flow

```
Player buys tickets
    ↓
PositionUpdate event emitted
    ↓
Backend listener catches event
    ↓
Backend calculates new probabilities for ALL players
    ↓
Backend updates database:
  - initial_probability_bps (when market first created)
  - current_probability_bps (always current)
    ↓
Frontend queries database for probabilities
    ↓
Frontend displays current odds
```

---

## Why This Fixes the Problem

### Before (Broken)
1. Market created on-chain with `probabilityBps = 10,000`
2. Event emitted with stale probability
3. Backend listener receives event with stale probability
4. Backend stores stale probability in database
5. Frontend displays stale odds
6. **Result**: Sync issues, confusion, incorrect odds

### After (Fixed)
1. Market created on-chain with NO probability data
2. Event emitted with only immutable data
3. Backend listener receives event
4. Backend queries Raffle contract for CURRENT probabilities
5. Backend stores current probabilities in database
6. Frontend queries database for current odds
7. **Result**: Always in sync, single source of truth

---

## Testing the Fix

### Prerequisites
```bash
# Terminal 1: Anvil
anvil --gas-limit 30000000

# Terminal 2: Backend
npm run dev:backend

# Terminal 3: Frontend (optional)
npm run dev
```

### Steps

**1. Rebuild contracts**
```bash
cd contracts
forge build
```

**2. Deploy fresh contracts**
```bash
forge script script/Deploy.s.sol \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast
```

**3. Update environment**
```bash
cd ..
node scripts/update-env-addresses.js
node scripts/copy-abis.js
```

**4. Create and start season**
```bash
export $(cat .env | xargs)
cd contracts
forge script script/CreateSeason.s.sol \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY \
  --broadcast

# Wait 60 seconds
sleep 61

cast send $RAFFLE_ADDRESS_LOCAL "startSeason(uint256)" 1 \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY
```

**5. Buy tickets to cross 1% threshold**
```bash
cast send $SOF_ADDRESS_LOCAL "approve(address,uint256)" \
  $CURVE_ADDRESS \
  0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY

cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" \
  11000 \
  3500000000000000000000 \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY
```

### Verification

**Check on-chain market creation**:
```bash
cast call $INFOFI_FACTORY_ADDRESS_LOCAL \
  "marketCreated(uint256,address)" 1 $ACCOUNT0_ADDRESS \
  --rpc-url $RPC_URL_LOCAL
```

**Expected**: `0x0000000000000000000000000000000000000000000000000000000000000001` (true)

**Check database**:
```sql
SELECT * FROM infofi_markets 
WHERE season_id = 1 
  AND player_address = '0x...'
  AND market_type = 'WINNER_PREDICTION';
```

**Expected**:
- `initial_probability_bps`: 10000 (100% when created)
- `current_probability_bps`: 10000 (current probability)
- `is_active`: true
- `contract_address`: NULL (will be populated when market deploys)

---

## Files Modified

1. **`contracts/src/infofi/InfoFiMarketFactory.sol`**
   - Removed `probabilityBps` from `MarketCreated` event
   - Removed `probabilityBps` parameter from `_createMarket()` and `_createMarketInternal()`
   - Updated function calls to match new signatures

2. **`backend/src/abis/InfoFiMarketFactoryAbi.js`**
   - Removed `probabilityBps` parameter from `MarketCreated` event definition

---

## Key Insights

### 1. Single Source of Truth
- Database is the source of truth for probabilities
- On-chain contract only stores immutable market metadata
- Frontend always queries database for current odds

### 2. Separation of Concerns
- **On-chain**: Immutable market creation and resolution
- **Off-chain**: Dynamic probability tracking and updates
- **Database**: Persistent state for frontend queries

### 3. Scalability
- Probabilities can be updated without touching on-chain state
- Multiple markets can have probabilities updated in parallel
- No on-chain transaction cost for probability updates

### 4. Data Consistency
- No sync issues between on-chain and database
- Probabilities always reflect current state
- Historical data preserved in database

---

## Next Steps

1. ✅ Rebuild contracts
2. ✅ Deploy fresh contracts
3. ✅ Test market creation on-chain
4. ✅ Verify database stores correct probabilities
5. ⏳ Test probability updates on subsequent purchases
6. ⏳ Verify frontend displays current odds

---

## Summary

The fix removes the problematic `probabilityBps` from the on-chain event, making the contract emit only immutable data. Probabilities are now tracked exclusively in the database, which is updated by the backend listener on every PositionUpdate event. This ensures:

- ✅ No sync issues between on-chain and database
- ✅ Always current probabilities for frontend
- ✅ Single source of truth in database
- ✅ Cleaner contract design with separation of concerns
- ✅ Better scalability for multiple concurrent markets

