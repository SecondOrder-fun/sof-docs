# On-Chain InfoFi Market Creation - Complete Flow Trace

**Date**: Oct 25, 2025

**Purpose**: Understand the complete on-chain flow for InfoFi market creation and identify robustness improvements

---

## High-Level Flow

```
Player buys tickets
    ↓
SOFBondingCurve.buyTokens() calls Raffle.recordParticipant()
    ↓
Raffle.recordParticipant() emits PositionUpdate event
    ↓
Raffle.recordParticipant() calls InfoFiMarketFactory.onPositionUpdate()
    ↓
InfoFiMarketFactory checks if player crossed 1% threshold
    ↓
If crossed: InfoFiMarketFactory._createMarket() called
    ↓
_createMarket() calls _createMarketInternal() via try-catch
    ↓
_createMarketInternal() creates Gnosis condition + FPMM market
    ↓
MarketCreated event emitted
```

---

## Detailed Step-by-Step Trace

### Step 1: Player Buys Tickets

**File**: `contracts/src/core/SOFBondingCurve.sol`

```solidity
function buyTokens(uint256 ticketAmount, uint256 maxSof) external {
    // ... bonding curve math ...
    
    // Call raffle to record participant
    raffle.recordParticipant(seasonId, msg.sender, ticketAmount);
}
```

**What happens**:
- User calls `buyTokens()` on bonding curve
- Bonding curve transfers SOF from user
- Bonding curve mints ticket tokens to user
- Bonding curve calls `Raffle.recordParticipant()`

### Step 2: Raffle Records Participant

**File**: `contracts/src/core/Raffle.sol` (lines 217-253)

```solidity
function recordParticipant(uint256 seasonId, address participant, uint256 ticketAmount)
    external
    onlyRole(BONDING_CURVE_ROLE)
{
    require(seasons[seasonId].isActive, "Raffle: season inactive");
    SeasonState storage state = seasonStates[seasonId];
    ParticipantPosition storage pos = state.participantPositions[participant];
    
    uint256 oldTickets = pos.ticketCount;
    uint256 newTicketsLocal = oldTickets + ticketAmount;
    
    // Update position
    if (!pos.isActive) {
        state.participants.push(participant);
        state.totalParticipants++;
        pos.entryBlock = block.number;
        pos.isActive = true;
    }
    
    pos.ticketCount = newTicketsLocal;
    pos.lastUpdateBlock = block.number;
    state.totalTickets += ticketAmount;
    
    // ✅ EMIT EVENT
    emit PositionUpdate(seasonId, participant, oldTickets, newTicketsLocal, state.totalTickets);
    
    // ✅ CALL INFOFI FACTORY
    if (infoFiFactory != address(0)) {
        try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(
            seasonId, participant, oldTickets, newTicketsLocal, state.totalTickets
        ) {
            // Success
        } catch {
            // Silently catch - InfoFi failure doesn't block raffle
        }
    }
}
```

**What happens**:
- Updates participant's ticket count
- Updates total tickets in season
- Emits `PositionUpdate` event
- Calls `InfoFiMarketFactory.onPositionUpdate()` in try-catch
- If InfoFi call fails, raffle continues (graceful degradation)

### Step 3: InfoFiMarketFactory Receives Position Update

**File**: `contracts/src/infofi/InfoFiMarketFactory.sol` (lines 99-117)

```solidity
function onPositionUpdate(
    uint256 seasonId,
    address player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets
) external onlyRole(RAFFLE_ROLE) nonReentrant {
    if (player == address(0)) revert InvalidAddress();
    if (totalTickets == 0) return;
    
    // Calculate probabilities
    uint256 oldBps = (oldTickets * 10000) / totalTickets;
    uint256 newBps = (newTickets * 10000) / totalTickets;
    
    // ✅ EMIT PROBABILITY UPDATE
    emit ProbabilityUpdated(seasonId, player, oldBps, newBps);
    
    // ✅ CHECK THRESHOLD CROSSING
    if (newBps >= THRESHOLD_BPS && oldBps < THRESHOLD_BPS && !marketCreated[seasonId][player]) {
        _createMarket(seasonId, player);
    }
}
```

**What happens**:
- Validates player address and total tickets
- Calculates old and new win probabilities in basis points
- Emits `ProbabilityUpdated` event for all position changes
- Checks if player crossed 1% threshold (100 bps)
- If crossed AND market not already created: calls `_createMarket()`

**Threshold Logic**:
- `THRESHOLD_BPS = 100` (1%)
- Checks: `newBps >= 100 && oldBps < 100 && !marketCreated[seasonId][player]`
- This ensures market is only created ONCE when crossing threshold

### Step 4: Create Market (With Error Handling)

**File**: `contracts/src/infofi/InfoFiMarketFactory.sol` (lines 119-132)

```solidity
function _createMarket(uint256 seasonId, address player) internal {
    // ✅ CHECK TREASURY BALANCE
    if (sofToken.balanceOf(treasury) < INITIAL_LIQUIDITY) {
        emit MarketCreationFailed(seasonId, player, WINNER_PREDICTION, "Insufficient treasury balance");
        return;
    }
    
    // ✅ CALL INTERNAL WITH TRY-CATCH
    try this._createMarketInternal(seasonId, player) {
        // Success
    } catch Error(string memory reason) {
        emit MarketCreationFailed(seasonId, player, WINNER_PREDICTION, reason);
    } catch {
        emit MarketCreationFailed(seasonId, player, WINNER_PREDICTION, "Unknown error");
    }
}
```

**What happens**:
- Checks if treasury has enough SOF for initial liquidity (100 SOF)
- If insufficient: emits `MarketCreationFailed` event and returns
- If sufficient: calls `_createMarketInternal()` via try-catch
- If internal call fails: emits `MarketCreationFailed` event with reason

**Error Handling**:
- Treasury check prevents revert
- Try-catch prevents market creation failure from blocking raffle
- Emits events for all failure modes

### Step 5: Create Market Internal

**File**: `contracts/src/infofi/InfoFiMarketFactory.sol` (lines 134-151)

```solidity
function _createMarketInternal(uint256 seasonId, address player) external {
    require(msg.sender == address(this), "Internal only");
    
    // ✅ STEP 1: PREPARE GNOSIS CONDITION
    bytes32 conditionId = oracleAdapter.preparePlayerCondition(seasonId, player);
    
    // ✅ STEP 2: TRANSFER LIQUIDITY FROM TREASURY
    require(sofToken.transferFrom(treasury, address(this), INITIAL_LIQUIDITY), "Treasury transfer failed");
    
    // ✅ STEP 3: APPROVE FPMM MANAGER
    sofToken.approve(address(fpmmManager), INITIAL_LIQUIDITY);
    
    // ✅ STEP 4: CREATE FPMM MARKET
    (address fpmm,) = fpmmManager.createMarket(seasonId, player, conditionId);
    
    // ✅ STEP 5: SET STATE
    marketCreated[seasonId][player] = true;
    playerConditions[seasonId][player] = conditionId;
    playerMarkets[seasonId][player] = fpmm;
    _seasonPlayers[seasonId].push(player);
    
    // ✅ STEP 6: EMIT EVENT
    emit MarketCreated(seasonId, player, WINNER_PREDICTION, conditionId, fpmm);
}
```

**What happens**:

1. **Prepare Gnosis Condition** (via RaffleOracleAdapter):
   - Creates binary outcome condition: [WIN, LOSE]
   - Question ID = keccak256(seasonId, player)
   - Returns conditionId for this condition

2. **Transfer Liquidity**:
   - Transfers 100 SOF from treasury to this contract
   - Reverts if transfer fails

3. **Approve FPMM Manager**:
   - Approves FPMM manager to spend 100 SOF

4. **Create FPMM Market**:
   - Calls InfoFiFPMMV2.createMarket()
   - Creates SimpleFPMM contract for this market
   - Initializes with 100 SOF liquidity
   - Returns FPMM address

5. **Set State**:
   - `marketCreated[seasonId][player] = true` (prevents duplicate creation)
   - Stores conditionId for later resolution
   - Stores FPMM address for trading
   - Adds player to season's player list

6. **Emit Event**:
   - Emits `MarketCreated` with immutable data (no probabilities)

---

## Robustness Analysis

### Current Strengths ✅

1. **Graceful Degradation**
   - Raffle continues even if InfoFi creation fails
   - Try-catch prevents market creation from blocking raffle

2. **Treasury Protection**
   - Checks balance before attempting transfer
   - Prevents revert from insufficient balance

3. **Duplicate Prevention**
   - `marketCreated` mapping prevents duplicate market creation
   - Threshold check ensures market created only once

4. **Event Logging**
   - All failure modes emit events for monitoring
   - Events help backend track what happened

5. **Access Control**
   - `onlyRole(RAFFLE_ROLE)` ensures only Raffle can call
   - `onlyRole(RESOLVER_ROLE)` for condition preparation

### Potential Issues ⚠️

1. **Silent Failures in Raffle**
   - Raffle catches all InfoFi failures silently
   - Backend might not know market creation failed
   - No way to retry or recover

2. **Treasury Depletion**
   - No mechanism to refill treasury
   - If treasury runs out, all subsequent markets fail
   - No warning or circuit breaker

3. **Condition Already Prepared**
   - If `preparePlayerCondition()` is called twice, it reverts
   - But `_createMarket()` doesn't check if condition already exists
   - Could cause silent failure if called twice

4. **FPMM Creation Failure**
   - If `fpmmManager.createMarket()` fails, caught by try-catch
   - But state is partially updated (condition prepared, SOF transferred)
   - Could leave orphaned conditions

5. **No Retry Mechanism**
   - If market creation fails, no way to retry
   - Player's position is recorded but market not created
   - Asymmetric state between raffle and InfoFi

6. **Approval Race Condition**
   - Approves FPMM manager for exact amount (100 SOF)
   - If approval fails, transfer might still be pending
   - Could cause issues with multiple concurrent markets

---

## Recommended Robustness Improvements

### 1. Add Market Creation Status Tracking

```solidity
enum MarketCreationStatus {
    NotStarted,
    ConditionPrepared,
    LiquidityTransferred,
    MarketCreated,
    Failed
}

mapping(uint256 => mapping(address => MarketCreationStatus)) public marketStatus;
mapping(uint256 => mapping(address => string)) public marketFailureReason;
```

**Benefit**: Backend can query exact failure point and retry appropriately.

### 2. Add Treasury Monitoring

```solidity
event TreasuryLow(uint256 currentBalance, uint256 requiredPerMarket);

function onPositionUpdate(...) external {
    // ... existing code ...
    
    // Warn if treasury getting low
    uint256 balance = sofToken.balanceOf(treasury);
    if (balance < INITIAL_LIQUIDITY * 10) {
        emit TreasuryLow(balance, INITIAL_LIQUIDITY);
    }
}
```

**Benefit**: Frontend/backend can alert admin to refill treasury before depletion.

### 3. Add Atomic Market Creation

```solidity
function _createMarketInternal(uint256 seasonId, address player) external {
    // Check all preconditions first
    require(sofToken.balanceOf(treasury) >= INITIAL_LIQUIDITY, "Insufficient treasury");
    require(playerConditions[seasonId][player] == bytes32(0), "Condition already prepared");
    require(!marketCreated[seasonId][player], "Market already created");
    
    // Then execute atomically
    bytes32 conditionId = oracleAdapter.preparePlayerCondition(seasonId, player);
    require(sofToken.transferFrom(treasury, address(this), INITIAL_LIQUIDITY), "Transfer failed");
    sofToken.approve(address(fpmmManager), INITIAL_LIQUIDITY);
    (address fpmm,) = fpmmManager.createMarket(seasonId, player, conditionId);
    
    // Set all state at end
    marketCreated[seasonId][player] = true;
    playerConditions[seasonId][player] = conditionId;
    playerMarkets[seasonId][player] = fpmm;
    _seasonPlayers[seasonId].push(player);
    
    emit MarketCreated(seasonId, player, WINNER_PREDICTION, conditionId, fpmm);
}
```

**Benefit**: Precondition checks prevent partial state updates.

### 4. Add Retry Mechanism

```solidity
function retryMarketCreation(uint256 seasonId, address player) external onlyRole(ADMIN_ROLE) {
    require(!marketCreated[seasonId][player], "Market already created");
    require(marketStatus[seasonId][player] == MarketCreationStatus.Failed, "Not in failed state");
    
    _createMarket(seasonId, player);
}
```

**Benefit**: Admin can retry failed market creations without player intervention.

### 5. Add Approval Increase Pattern

```solidity
// Instead of:
sofToken.approve(address(fpmmManager), INITIAL_LIQUIDITY);

// Use:
uint256 currentAllowance = sofToken.allowance(address(this), address(fpmmManager));
if (currentAllowance < INITIAL_LIQUIDITY) {
    sofToken.approve(address(fpmmManager), 0);
    sofToken.approve(address(fpmmManager), INITIAL_LIQUIDITY);
}
```

**Benefit**: Prevents approval issues with certain token implementations.

### 6. Add Condition Existence Check

```solidity
function _createMarketInternal(uint256 seasonId, address player) external {
    // Check if condition already prepared
    bytes32 existingCondition = playerConditions[seasonId][player];
    if (existingCondition != bytes32(0)) {
        // Condition already exists, use it
        conditionId = existingCondition;
    } else {
        // Prepare new condition
        conditionId = oracleAdapter.preparePlayerCondition(seasonId, player);
    }
    
    // ... rest of creation ...
}
```

**Benefit**: Handles idempotent retries gracefully.

---

## Implementation Priority

### High Priority (Do First)

1. **Market Creation Status Tracking** - Enables debugging and monitoring
2. **Atomic Market Creation** - Prevents partial state updates
3. **Precondition Checks** - Catches errors early

### Medium Priority (Do Next)

4. **Treasury Monitoring** - Prevents depletion
5. **Retry Mechanism** - Enables recovery from transient failures

### Low Priority (Nice to Have)

6. **Approval Pattern** - Defensive programming
7. **Condition Existence Check** - Idempotency

---

## Testing Strategy

### Unit Tests

```solidity
// Test threshold crossing
function testMarketCreatedOnThresholdCrossing() public

// Test duplicate prevention
function testMarketNotCreatedTwice() public

// Test treasury check
function testMarketNotCreatedInsufficientTreasury() public

// Test graceful failure
function testRaffleContinuesIfMarketCreationFails() public

// Test all failure modes
function testMarketCreationFailureEvents() public
```

### Integration Tests

```javascript
// Test full flow: buy → threshold cross → market created
test('Market created when player crosses 1% threshold')

// Test multiple players
test('Multiple markets created for different players')

// Test treasury depletion
test('Market creation fails when treasury depleted')

// Test recovery
test('Admin can retry failed market creation')
```

---

## Summary

The current on-chain InfoFi market creation flow is **fundamentally sound** but has several **robustness gaps**:

**Strengths**:
- Graceful degradation (raffle continues if InfoFi fails)
- Treasury protection (checks balance)
- Duplicate prevention (threshold check + state flag)
- Good event logging

**Gaps**:
- Silent failures (no way to know what failed)
- No retry mechanism (stuck in failed state)
- Partial state updates possible (condition prepared but market not created)
- Treasury depletion risk (no monitoring)

**Recommended Fix**: Implement market creation status tracking + atomic creation + precondition checks. This provides observability, prevents partial updates, and enables recovery.

