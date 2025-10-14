# Oracle Update Bug: Complete Audit & Fix Plan

## Executive Summary

**Critical Bug Identified**: When a player buys/sells raffle tickets, only THEIR market's oracle is updated, not ALL markets in the season. This causes stale probability data for other players.

**Impact**: HIGH - Breaks core InfoFi functionality, creates incorrect arbitrage opportunities, violates hybrid pricing model

**Root Cause**: `InfoFiMarketFactory.onPositionUpdate()` only updates the oracle for the player whose position changed, not for all other players whose relative probabilities also changed.

**Fix Complexity**: MEDIUM - Requires iterating through all season players and updating their oracles

**Gas Impact**: MODERATE - Additional oracle updates per transaction, but necessary for correctness

## Complete Audit Findings

### 1. Documentation Requirements (project-requirements.md)

**Lines 76-82** specify:
```
Hybrid Pricing Model (70% Raffle Data + 30% Market Sentiment) — On-chain:
- Raffle position probability is computed on-chain from `Raffle` totals (tickets/totalTickets)
- An on-chain `InfoFiPriceOracle` combines both into a hybrid price in basis points
- Live updates are streamed via Server-Sent Events (SSE) as a transport layer for oracle outputs
```

**Requirement**: Oracle MUST reflect current raffle probabilities at all times.

### 2. Current Implementation Flow

```
Player2 buys tickets
  ↓
SOFBondingCurve.buyTokens() called
  ↓
Emits PositionUpdate event
  ↓
Calls InfoFiMarketFactory.onPositionUpdate(seasonId, Player2, oldTickets, newTickets, totalTickets)
  ↓
Factory calculates: newBps = (Player2Tickets * 10000) / totalTickets
  ↓
Factory updates ONLY Player2's oracle: oracle.updateRaffleProbability(marketId=1, 4000)
  ↓
❌ Player1's oracle is NEVER updated (still shows 10000 instead of 6000)
```

### 3. Mathematical Proof of Bug

**Initial State:**
- Player1: 15,000 tickets out of 15,000 total = 100% (10000 bps)
- Player2: 0 tickets = 0%

**After Player2 buys 6,000 tickets:**
- Total tickets: 21,000
- Player1: 15,000 / 21,000 = 71.4% (7143 bps) ✅ SHOULD BE
- Player2: 6,000 / 21,000 = 28.6% (2857 bps) ✅ CORRECT

**Actual Oracle State:**
- Player1 oracle: 10000 bps ❌ WRONG (never updated)
- Player2 oracle: 2857 bps ✅ CORRECT

**This violates the fundamental constraint: Sum of all probabilities MUST equal 100%**

### 4. Code Analysis

#### InfoFiMarketFactory.sol (Lines 115-180)

**Current Logic:**
```solidity
function onPositionUpdate(
    uint256 seasonId,
    address player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets
) external {
    // ... validation ...
    
    uint256 newBps = (newTickets * 10000) / totalTickets;
    
    // ... market creation logic ...
    
    // ❌ BUG: Only updates THIS player's oracle
    uint256 marketId = winnerPredictionMarketIds[seasonId][player];
    if (marketId > 0 || winnerPredictionCreated[seasonId][player]) {
        oracle.updateRaffleProbability(marketId, newBps);
    }
}
```

**Missing Logic:**
- No iteration through `_seasonPlayers[seasonId]`
- No recalculation of other players' probabilities
- No batch oracle updates

### 5. Gas Cost Analysis

**Current Cost (per position update):**
- 1 oracle update: ~5,000 gas

**Proposed Cost (per position update):**
- N oracle updates where N = number of players with markets
- Worst case: 10 players × 5,000 gas = 50,000 gas
- Typical case: 3-5 players × 5,000 gas = 15,000-25,000 gas

**Mitigation Strategies:**
1. Only update players with existing markets (check `winnerPredictionCreated`)
2. Use unchecked math where safe
3. Cache storage reads
4. Consider max player limit per season

### 6. OpenZeppelin Best Practices Review

From Context7 research on OpenZeppelin patterns:

**Batch Operations:**
- ERC-1155 uses `safeBatchTransferFrom` for multiple updates
- Memory optimization: Cache free memory pointer in loops
- Gas optimization: Use unchecked blocks for safe arithmetic

**Storage Access:**
- Minimize SLOAD operations by caching
- Use memory arrays for iteration
- Avoid redundant storage writes

## Complete Fix Plan

### Phase 1: Contract Changes

#### File: `contracts/src/infofi/InfoFiMarketFactory.sol`

**Change 1: Add helper function to update all season players**

```solidity
/**
 * @notice Updates oracle probabilities for all players in a season
 * @dev Called after any position change to maintain probability consistency
 * @param seasonId The season to update
 * @param totalTickets Current total tickets in the season
 */
function _updateAllSeasonProbabilities(uint256 seasonId, uint256 totalTickets) internal {
    if (totalTickets == 0) return;
    
    address[] memory players = _seasonPlayers[seasonId];
    uint256 playerCount = players.length;
    
    // Use unchecked for gas optimization (we know these won't overflow)
    unchecked {
        for (uint256 i = 0; i < playerCount; ++i) {
            address player = players[i];
            
            // Only update if market exists
            if (!winnerPredictionCreated[seasonId][player]) continue;
            
            uint256 marketId = winnerPredictionMarketIds[seasonId][player];
            if (marketId == 0) continue;
            
            // Get current player position
            IRaffleRead.ParticipantPosition memory pos = iRaffle.getParticipantPosition(seasonId, player);
            
            // Calculate new probability
            uint256 newBps = (pos.ticketCount * 10000) / totalTickets;
            
            // Update oracle
            oracle.updateRaffleProbability(marketId, newBps);
        }
    }
}
```

**Change 2: Modify onPositionUpdate to call batch update**

```solidity
function onPositionUpdate(
    uint256 seasonId,
    address player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets
) external {
    // ... existing validation ...
    
    uint256 oldBps = totalTickets > 0 ? (oldTickets * 10000) / totalTickets : 0;
    uint256 newBps = totalTickets > 0 ? (newTickets * 10000) / totalTickets : 0;
    emit ProbabilityUpdated(seasonId, player, oldBps, newBps);
    
    // ... existing market creation logic ...
    
    // ✅ FIX: Update ALL players' probabilities, not just this one
    _updateAllSeasonProbabilities(seasonId, totalTickets);
}
```

**Change 3: Add gas optimization - max players check**

```solidity
// Add to contract state
uint256 public constant MAX_PLAYERS_PER_SEASON = 50; // Reasonable limit

// Add check in onPositionUpdate
require(_seasonPlayers[seasonId].length < MAX_PLAYERS_PER_SEASON, "Factory: max players reached");
```

### Phase 2: Testing

#### File: `contracts/test/InfoFiOracleUpdate.t.sol` (NEW)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/infofi/InfoFiMarketFactory.sol";
import "../src/infofi/InfoFiPriceOracle.sol";

contract InfoFiOracleUpdateTest is Test {
    InfoFiMarketFactory factory;
    InfoFiPriceOracle oracle;
    
    function testOracleUpdatesAllPlayersOnPositionChange() public {
        // Setup: Player1 has 15k tickets, Player2 has 0
        // Player1 oracle should show 100%
        
        // Action: Player2 buys 6k tickets
        // Total now 21k
        
        // Assert: Player1 oracle shows ~71.4% (15k/21k)
        // Assert: Player2 oracle shows ~28.6% (6k/21k)
        // Assert: Sum of probabilities = 100%
    }
    
    function testOracleUpdatesWithThreePlayers() public {
        // Test with 3 players to ensure iteration works
    }
    
    function testGasCostOfBatchUpdate() public {
        // Measure gas cost with 1, 5, 10 players
    }
}
```

### Phase 3: Documentation Updates

#### File: `instructions/project-requirements.md`

Add clarification:
```markdown
### Oracle Update Mechanism (CRITICAL)

When ANY player's position changes:
1. The InfoFiMarketFactory receives onPositionUpdate callback
2. Factory recalculates probabilities for ALL players in the season
3. Oracle is updated for every player with an active market
4. This ensures probabilities always sum to 100%

Gas Cost: O(N) where N = number of players with markets
Mitigation: Max 50 players per season enforced
```

### Phase 4: Migration Strategy

**For Existing Deployments:**

1. Deploy new InfoFiMarketFactory with fix
2. Grant OPERATOR_ROLE to new factory
3. Revoke OPERATOR_ROLE from old factory
4. Call `refreshAllOraclePrices(seasonId)` to fix stale data

**Refresh Function (add to factory):**
```solidity
function refreshAllOraclePrices(uint256 seasonId) external onlyRole(ADMIN_ROLE) {
    (, , , uint256 totalTickets, ) = iRaffle.getSeasonDetails(seasonId);
    _updateAllSeasonProbabilities(seasonId, totalTickets);
}
```

## Implementation Checklist

### Smart Contracts
- [ ] Add `_updateAllSeasonProbabilities()` internal function
- [ ] Modify `onPositionUpdate()` to call batch update
- [ ] Add `MAX_PLAYERS_PER_SEASON` constant
- [ ] Add `refreshAllOraclePrices()` admin function
- [ ] Add gas optimization (unchecked blocks, caching)

### Testing
- [ ] Create `InfoFiOracleUpdate.t.sol` test file
- [ ] Test 2-player scenario (current bug case)
- [ ] Test 3+ player scenarios
- [ ] Test gas costs with varying player counts
- [ ] Test edge cases (player exits, re-enters)
- [ ] Add invariant test: sum of probabilities = 100%

### Documentation
- [ ] Update `project-requirements.md` with oracle update mechanism
- [ ] Create migration guide for existing deployments
- [ ] Update API documentation
- [ ] Add gas cost analysis to docs

### Deployment
- [ ] Deploy new InfoFiMarketFactory
- [ ] Update contract addresses in .env
- [ ] Run migration script to refresh stale oracles
- [ ] Verify all oracle values sum to 100%

## Risk Assessment

### High Risk Areas
1. **Gas costs** - Could make transactions expensive with many players
2. **Reentrancy** - Multiple external calls to oracle
3. **DoS** - Malicious actor creates many small positions

### Mitigations
1. **Gas**: MAX_PLAYERS_PER_SEASON limit (50 players)
2. **Reentrancy**: Oracle is trusted contract, no state changes after calls
3. **DoS**: Minimum position size requirements, 1% threshold for market creation

### Security Considerations
- Factory already has ReentrancyGuard (inherited)
- Oracle updates are idempotent (safe to call multiple times)
- No user funds at risk (only probability calculations)
- Admin function for emergency oracle refresh

## Success Criteria

### Functional
- ✅ All player probabilities sum to 100% at all times
- ✅ Oracle updates happen atomically with position changes
- ✅ No stale probability data

### Performance
- ✅ Gas cost < 100k for typical 5-player season
- ✅ Transaction success rate > 99%
- ✅ No timeout issues with max players

### Testing
- ✅ All unit tests pass
- ✅ Invariant tests pass
- ✅ Gas benchmarks within acceptable range
- ✅ Integration tests with live contracts pass

## Timeline Estimate

- **Contract Changes**: 2 hours
- **Testing**: 3 hours
- **Documentation**: 1 hour
- **Deployment & Verification**: 1 hour
- **Total**: ~7 hours

## Conclusion

This is a **critical bug** that breaks the core InfoFi functionality. The fix is straightforward but requires careful implementation to manage gas costs. The proposed solution follows OpenZeppelin best practices for batch operations and maintains the integrity of the hybrid pricing model.

**Recommendation**: Implement immediately, as this affects all InfoFi market pricing and arbitrage detection.
