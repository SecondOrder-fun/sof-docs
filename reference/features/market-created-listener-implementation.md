# MarketCreated Listener Implementation Plan

**Objective**: Update `marketCreatedListener.js` to listen to BOTH InfoFiMarketFactory's `MarketCreated` event AND the bonding curve's position event, validate data consistency, and only persist to database after both events are received.

---

## Architecture Overview

### Current State
- ❌ Only listens to InfoFiMarketFactory `MarketCreated` event
- ❌ Stores fpmmAddress immediately without validation
- ❌ No cross-event validation

### Desired State
- ✅ Listen to bonding curve position event (triggers market creation)
- ✅ Listen to InfoFiMarketFactory `MarketCreated` event (confirms creation)
- ✅ Validate data consistency between both events
- ✅ Only persist to database after both events received
- ✅ Handle race conditions and timeouts

---

## Event Signatures

### Event 1: Bonding Curve Position Event
**Source**: SOFBondingCurve contract (via Raffle.recordParticipant)
**Event**: `PositionUpdate` or similar
**Parameters**:
- `seasonId` (uint256)
- `player` (address)
- `newTickets` (uint256)
- `totalTickets` (uint256)

**Triggers**: Market creation when `newTickets/totalTickets >= 1%`

### Event 2: InfoFiMarketFactory MarketCreated Event
**Source**: InfoFiMarketFactory contract
**Event**: `MarketCreated`
**Parameters** (from line 70-76 of InfoFiMarketFactory.sol):
- `seasonId` (uint256, indexed)
- `player` (address, indexed)
- `marketType` (bytes32, indexed) = `keccak256("WINNER_PREDICTION")`
- `conditionId` (bytes32)
- `fpmmAddress` (address) ← **The critical data we need**

---

## Implementation Plan (5 Phases)

### Phase 1: Create Event State Manager
**File**: `backend/src/services/marketCreationStateManager.js` (NEW)

**Purpose**: Track pending market creations waiting for both events

**Responsibilities**:
- Store pending market creation state (seasonId, player, position data)
- Track which events have been received (position event, MarketCreated event)
- Validate data consistency between events
- Handle timeouts (if MarketCreated not received within X seconds)
- Provide methods to query and update state

**Key Methods**:
```javascript
// Initialize pending market creation
initiatePending(seasonId, player, positionData)

// Record position event received
recordPositionEvent(seasonId, player, positionData)

// Record MarketCreated event received
recordMarketCreatedEvent(seasonId, player, marketCreatedData)

// Check if both events received
areBothEventsReceived(seasonId, player)

// Validate consistency between events
validateConsistency(seasonId, player)

// Get pending market data
getPendingMarket(seasonId, player)

// Clean up after persistence
removePending(seasonId, player)

// Handle timeouts
checkForTimeouts()
```

**Data Structure**:
```javascript
{
  [seasonId]: {
    [player]: {
      positionEvent: {
        received: boolean,
        data: { seasonId, player, newTickets, totalTickets, ... }
      },
      marketCreatedEvent: {
        received: boolean,
        data: { seasonId, player, marketType, conditionId, fpmmAddress }
      },
      createdAt: timestamp,
      lastUpdated: timestamp
    }
  }
}
```

---

### Phase 2: Update Position Event Listener
**File**: `backend/src/listeners/positionUpdateListener.js` (MODIFY)

**Changes**:
1. When position crosses 1% threshold:
   - Call `stateManager.initiatePending(seasonId, player, positionData)`
   - Log: "Position event received, waiting for MarketCreated event"
   - Do NOT persist to database yet

2. Add error handling for state manager

**Key Addition**:
```javascript
// After threshold crossing detected
const stateManager = require('../services/marketCreationStateManager');
stateManager.recordPositionEvent(seasonId, player, {
  newTickets,
  totalTickets,
  probabilityBps
});
```

---

### Phase 3: Update MarketCreated Listener
**File**: `backend/src/listeners/marketCreatedListener.js` (MODIFY)

**Current Logic** (lines 33-71):
- Extracts: seasonId, player, marketType, conditionId, fpmmAddress
- Stores immediately to database

**New Logic**:
1. Extract event data (same as before)
2. Call `stateManager.recordMarketCreatedEvent(seasonId, player, eventData)`
3. Check if both events received: `stateManager.areBothEventsReceived(seasonId, player)`
4. If YES:
   - Validate consistency: `stateManager.validateConsistency(seasonId, player)`
   - If valid: Persist to database
   - If invalid: Log error, emit alert event
5. If NO:
   - Log: "MarketCreated event received, waiting for position event"
   - Set timeout for X seconds
   - If timeout: Emit alert, mark as failed

**Key Addition**:
```javascript
const stateManager = require('../services/marketCreationStateManager');

// After extracting event data
stateManager.recordMarketCreatedEvent(seasonId, player, {
  marketType,
  conditionId,
  fpmmAddress
});

// Check if both events received
if (stateManager.areBothEventsReceived(seasonId, player)) {
  // Validate consistency
  const isValid = stateManager.validateConsistency(seasonId, player);
  if (isValid) {
    // Persist to database
    await db.updateMarketContractAddress(...);
    stateManager.removePending(seasonId, player);
  } else {
    // Log validation error
    logger.error("Data mismatch between events");
  }
}
```

---

### Phase 4: Add Timeout and Cleanup Handler
**File**: `backend/fastify/server.js` (MODIFY)

**Purpose**: Periodically check for stale pending markets and clean them up

**Implementation**:
1. Use Fastify `onReady` hook to start cleanup interval
2. Check for timeouts every 30 seconds
3. For stale markets (>60 seconds old):
   - Log warning
   - Emit alert event
   - Remove from pending state
4. Use Fastify `onClose` hook to clear interval

**Key Addition**:
```javascript
// In server.js onReady hook
const stateManager = require('./services/marketCreationStateManager');
const cleanupInterval = setInterval(() => {
  stateManager.checkForTimeouts();
}, 30000);

// In onClose hook
clearInterval(cleanupInterval);
```

---

### Phase 5: Add Data Validation Logic
**File**: `backend/src/services/marketCreationStateManager.js` (ADD)

**Validation Rules**:
1. ✅ `seasonId` matches between both events
2. ✅ `player` address matches between both events
3. ✅ `marketType` is `WINNER_PREDICTION` (keccak256 hash)
4. ✅ `fpmmAddress` is valid Ethereum address (not zero address)
5. ✅ `conditionId` is valid bytes32 (not zero)
6. ✅ Position event received BEFORE MarketCreated event (optional, but good to track)

**Validation Method**:
```javascript
validateConsistency(seasonId, player) {
  const pending = this.getPendingMarket(seasonId, player);
  
  if (!pending.positionEvent.received || !pending.marketCreatedEvent.received) {
    return false; // Both events not received
  }
  
  const pos = pending.positionEvent.data;
  const market = pending.marketCreatedEvent.data;
  
  // Validate matches
  if (pos.seasonId !== market.seasonId) return false;
  if (pos.player !== market.player) return false;
  if (market.fpmmAddress === '0x0000000000000000000000000000000000000000') return false;
  if (market.conditionId === '0x0000000000000000000000000000000000000000000000000000000000000000') return false;
  
  return true;
}
```

---

## Database Schema Changes

### New Table: `market_creation_pending` (Optional, for audit trail)

```sql
CREATE TABLE market_creation_pending (
  id BIGSERIAL PRIMARY KEY,
  season_id BIGINT NOT NULL,
  player_address VARCHAR(42) NOT NULL,
  position_event_received BOOLEAN DEFAULT false,
  position_event_data JSONB,
  market_created_event_received BOOLEAN DEFAULT false,
  market_created_event_data JSONB,
  validation_status VARCHAR(50),  -- 'pending', 'valid', 'invalid', 'timeout'
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  resolved_at TIMESTAMPTZ,
  UNIQUE(season_id, player_address)
);
```

---

## Error Handling & Edge Cases

### Case 1: MarketCreated Event Arrives First
- ✅ Handled: State manager stores it, waits for position event
- Timeout: 60 seconds

### Case 2: Position Event Arrives First
- ✅ Handled: State manager stores it, waits for MarketCreated event
- Timeout: 60 seconds

### Case 3: Data Mismatch Between Events
- ✅ Log error with full details
- ✅ Emit alert event for admin review
- ✅ Do NOT persist to database
- ✅ Mark as failed in state manager

### Case 4: Duplicate Events
- ✅ State manager checks if already recorded
- ✅ Logs duplicate, ignores second occurrence

### Case 5: Timeout (Event Never Arrives)
- ✅ Cleanup handler removes from pending after 60 seconds
- ✅ Logs warning with details
- ✅ Emits alert event

---

## Testing Strategy

### Unit Tests
1. **stateManager.initiatePending()** - Creates pending state
2. **stateManager.recordPositionEvent()** - Records position event
3. **stateManager.recordMarketCreatedEvent()** - Records market created event
4. **stateManager.areBothEventsReceived()** - Checks both events received
5. **stateManager.validateConsistency()** - Validates data consistency
6. **stateManager.checkForTimeouts()** - Handles timeouts

### Integration Tests
1. Position event arrives first, then MarketCreated
2. MarketCreated arrives first, then position event
3. Data mismatch between events
4. Timeout scenario
5. Duplicate events

### End-to-End Tests
1. Full flow: bonding curve → position event → MarketCreated event → database persistence
2. Verify fpmmAddress stored correctly
3. Verify no orphaned pending states

---

## Implementation Order

1. **Create stateManager** (Phase 1) - Foundation
2. **Update positionUpdateListener** (Phase 2) - Record position events
3. **Update marketCreatedListener** (Phase 3) - Record market created events + validation
4. **Add timeout handler** (Phase 4) - Cleanup stale pending states
5. **Add tests** (Phase 5) - Comprehensive coverage

---

## Questions & Clarifications

### Q1: Timeout Duration
**Current Plan**: 60 seconds
**Question**: Is this appropriate? Should it be configurable?

### Q2: State Storage
**Current Plan**: In-memory Map
**Question**: Should we persist pending state to database for recovery after server restart?

### Q3: Alert Events
**Current Plan**: Emit custom events for admin review
**Question**: Should these trigger email/Slack notifications?

### Q4: Retry Logic
**Current Plan**: No automatic retry
**Question**: Should we implement retry mechanism for failed markets?

### Q5: Position Event Source
**Current Plan**: Assume it comes from bonding curve via Raffle
**Question**: Should we validate the exact event source/contract address?

---

## Success Criteria

✅ Both events are captured and stored in state manager
✅ Data consistency validated between events
✅ Database only updated after both events received AND validated
✅ Timeout handling prevents stale pending states
✅ Comprehensive logging for debugging
✅ No orphaned fpmmAddress records
✅ All tests passing

---

## Best Practices Applied

- **Viem watchContractEvent**: Proper event listening with polling support
- **Fastify Lifecycle Hooks**: Using `onReady` and `onClose` for setup/cleanup
- **State Management**: Centralized state manager for consistency
- **Error Handling**: Comprehensive error cases and timeouts
- **Logging**: Detailed logging at each step for debugging
- **Validation**: Multi-step validation before persistence
- **OpenZeppelin Standards**: Following ERC standards for event interpretation
