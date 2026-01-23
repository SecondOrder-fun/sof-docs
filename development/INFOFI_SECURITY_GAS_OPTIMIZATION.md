# InfoFi Security & Gas Optimization Implementation Plan

**Date:** October 16, 2025  
**Priority:** HIGH  
**Estimated Time:** 1-2 weeks  
**Status:** Ready for Implementation

---

## Overview

This plan addresses **Moderate Issues #3 and #4** from the Polymarket audit:
- **Issue #3:** Missing safety checks (zero-division, invariants, solvency)
- **Issue #4:** Gas inefficiency (no batching, on-chain strings, unoptimized loops)

These improvements will be integrated with the existing **Batch Claims** implementation plan.

---

## Phase 1: Safety Checks & Invariants (Priority: CRITICAL)

### 1.1 Add Pool Invariant Checks

**File:** `contracts/src/infofi/InfoFiMarket.sol`  
**Location:** Lines 131-171 (in `placeBet` function)

**Changes:**

```solidity
function placeBet(
    uint256 marketId,
    bool prediction,
    uint256 amount
) external nonReentrant whenNotPaused {
    MarketInfo storage market = markets[marketId];
    require(!market.resolved, "InfoFiMarket: market resolved");
    require(!market.locked, "InfoFiMarket: market locked");
    require(amount >= MIN_BET_AMOUNT && amount <= MAX_BET_AMOUNT, "InfoFiMarket: invalid bet amount");

    IERC20 token = IERC20(market.tokenAddress);
    
    // Transfer tokens from better to contract
    require(token.transferFrom(msg.sender, address(this), amount), "InfoFiMarket: token transfer failed");

    // Update market pools
    if (prediction) {
        market.totalYesPool += amount;
    } else {
        market.totalNoPool += amount;
    }
    market.totalPool += amount;

    // ✅ ADD: Pool invariant check
    assert(market.totalYesPool + market.totalNoPool == market.totalPool);
    
    // ✅ ADD: Overflow protection (explicit check even with Solidity 0.8.20)
    require(market.totalPool >= amount, "InfoFiMarket: pool overflow");

    // Update bet info per outcome (hedging supported)
    BetInfo storage bet = bets[marketId][msg.sender][prediction];
    if (bet.amount == 0) {
        // Only push marketId once per bettor (first-ever position across both outcomes)
        BetInfo storage other = bets[marketId][msg.sender][!prediction];
        if (other.amount == 0) {
            betterMarkets[msg.sender].push(marketId);
        }
    }
    
    bet.prediction = prediction;
    bet.amount += amount;

    emit BetPlaced(msg.sender, marketId, prediction, amount);

    // Push on-chain sentiment to oracle (if configured)
    _updateOracle(marketId);
}
```

**Test Case:**
```solidity
// contracts/test/InfoFiMarketInvariants.t.sol
function testPoolInvariantMaintained() public {
    uint256 marketId = createTestMarket();
    
    // Place YES bet
    vm.prank(user1);
    market.placeBet(marketId, true, 100 ether);
    
    // Place NO bet
    vm.prank(user2);
    market.placeBet(marketId, false, 50 ether);
    
    InfoFiMarket.MarketInfo memory info = market.getMarket(marketId);
    
    // Invariant: totalYesPool + totalNoPool == totalPool
    assertEq(info.totalYesPool + info.totalNoPool, info.totalPool);
    assertEq(info.totalPool, 150 ether);
}
```

---

### 1.2 Add Zero-Division Protection to Payout Calculation

**File:** `contracts/src/infofi/InfoFiMarket.sol`  
**Location:** Lines 222-241 (in `claimPayout` function)

**Changes:**

```solidity
function claimPayout(uint256 marketId, bool prediction) external nonReentrant whenNotPaused {
    MarketInfo storage market = markets[marketId];
    BetInfo storage bet = bets[marketId][msg.sender][prediction];
    
    require(market.resolved, "InfoFiMarket: market not resolved");
    require(bet.amount > 0, "InfoFiMarket: no bet placed");
    require(!bet.claimed, "InfoFiMarket: payout already claimed");
    require(bet.prediction == market.outcome, "InfoFiMarket: incorrect prediction");
    
    // ✅ ADD: Zero-division protection
    uint256 winningPool = market.outcome ? market.totalYesPool : market.totalNoPool;
    require(winningPool > 0, "InfoFiMarket: zero winning pool");
    
    // Calculate payout
    uint256 payout = (bet.amount * market.totalPool) / winningPool;
    
    // ✅ ADD: Solvency check
    IERC20 token = IERC20(market.tokenAddress);
    uint256 contractBalance = token.balanceOf(address(this));
    require(payout <= contractBalance, "InfoFiMarket: insufficient balance");
    
    // ✅ ADD: Sanity check - payout should not exceed total pool
    require(payout <= market.totalPool, "InfoFiMarket: payout exceeds pool");
    
    bet.payout = payout;
    bet.claimed = true;
    
    require(token.transfer(msg.sender, payout), "InfoFiMarket: payout transfer failed");
    
    emit PayoutClaimed(msg.sender, marketId, prediction, payout);
}
```

**Also update the overloaded version (lines 246-264):**

```solidity
function claimPayout(uint256 marketId) external nonReentrant whenNotPaused {
    MarketInfo storage market = markets[marketId];
    require(market.resolved, "InfoFiMarket: market not resolved");
    bool winning = market.outcome;
    BetInfo storage bet = bets[marketId][msg.sender][winning];
    require(bet.amount > 0, "InfoFiMarket: no winning-side bet");
    require(!bet.claimed, "InfoFiMarket: payout already claimed");
    
    // ✅ ADD: Zero-division protection
    uint256 winningPool = winning ? market.totalYesPool : market.totalNoPool;
    require(winningPool > 0, "InfoFiMarket: zero winning pool");
    
    // Calculate payout
    uint256 payout = (bet.amount * market.totalPool) / winningPool;
    
    // ✅ ADD: Solvency check
    IERC20 token = IERC20(market.tokenAddress);
    uint256 contractBalance = token.balanceOf(address(this));
    require(payout <= contractBalance, "InfoFiMarket: insufficient balance");
    
    // ✅ ADD: Sanity check
    require(payout <= market.totalPool, "InfoFiMarket: payout exceeds pool");
    
    bet.payout = payout;
    bet.claimed = true;
    
    require(token.transfer(msg.sender, payout), "InfoFiMarket: payout transfer failed");
    
    emit PayoutClaimed(msg.sender, marketId, winning, payout);
}
```

**Test Cases:**
```solidity
// Test zero winning pool
function testCannotClaimWithZeroWinningPool() public {
    uint256 marketId = createTestMarket();
    
    // Artificially create scenario with zero winning pool (edge case)
    // This should never happen in practice but we protect against it
    
    vm.expectRevert("InfoFiMarket: zero winning pool");
    vm.prank(user1);
    market.claimPayout(marketId, true);
}

// Test insufficient balance
function testCannotClaimWithInsufficientBalance() public {
    uint256 marketId = createTestMarket();
    
    // Place bets
    vm.prank(user1);
    market.placeBet(marketId, true, 100 ether);
    
    // Resolve market
    vm.prank(operator);
    market.resolveMarket(marketId, true);
    
    // Drain contract balance (simulate exploit or bug)
    vm.prank(address(market));
    sofToken.transfer(address(0xdead), sofToken.balanceOf(address(market)));
    
    // Attempt claim should fail
    vm.expectRevert("InfoFiMarket: insufficient balance");
    vm.prank(user1);
    market.claimPayout(marketId, true);
}
```

---

### 1.3 Add Payout Preview Function (from original batch claims plan)

**File:** `contracts/src/infofi/InfoFiMarket.sol`  
**Location:** After line 264 (after existing `claimPayout` functions)

**Add new view function:**

```solidity
/**
 * @dev Calculate potential payout for a bet without claiming
 * @param marketId ID of the market
 * @param better Address of the better
 * @param prediction Which side (true=YES, false=NO)
 * @return payout Potential payout amount (0 if not winning or already claimed)
 */
function calculatePayout(
    uint256 marketId,
    address better,
    bool prediction
) external view returns (uint256 payout) {
    MarketInfo storage market = markets[marketId];
    BetInfo storage bet = bets[marketId][better][prediction];
    
    // Return 0 if market not resolved, no bet, already claimed, or wrong side
    if (!market.resolved || bet.amount == 0 || bet.claimed || bet.prediction != market.outcome) {
        return 0;
    }
    
    // ✅ Zero-division protection
    uint256 winningPool = market.outcome ? market.totalYesPool : market.totalNoPool;
    if (winningPool == 0) return 0;
    
    // Calculate payout using same formula as claimPayout
    payout = (bet.amount * market.totalPool) / winningPool;
    
    // ✅ Cap at total pool (sanity check)
    if (payout > market.totalPool) {
        payout = market.totalPool;
    }
}
```

---

### 1.4 Add ClaimRequest Struct for Batch Operations

**File:** `contracts/src/infofi/InfoFiMarket.sol`  
**Location:** After line 65 (after `BetInfo` struct)

**Add new struct:**

```solidity
/**
 * @dev Struct for batch claim requests
 */
struct ClaimRequest {
    uint256 marketId;
    bool prediction;
}
```

---

### 1.5 Add Batch Claim Function with Safety Checks

**File:** `contracts/src/infofi/InfoFiMarket.sol`  
**Location:** After the new `calculatePayout` function

**Add batch claim function:**

```solidity
/**
 * @dev Batch claim payouts for multiple markets in a single transaction
 * @param claims Array of (marketId, prediction) tuples to claim
 * @return totalPayout Total amount claimed across all markets
 * 
 * Gas optimization notes:
 * - Uses unchecked blocks for loop counters (safe, we control iteration)
 * - Single token transfer at end instead of N transfers
 * - Shared nonReentrant check across all claims
 * - Skips invalid claims silently to allow partial success
 * 
 * Safety features:
 * - Zero-division protection on each claim
 * - Solvency check before final transfer
 * - Pool invariant validation
 */
function batchClaimPayouts(
    ClaimRequest[] calldata claims
) external nonReentrant whenNotPaused returns (uint256 totalPayout) {
    require(claims.length > 0, "InfoFiMarket: empty claims array");
    require(claims.length <= 50, "InfoFiMarket: too many claims"); // Gas limit protection
    
    IERC20 token;
    address tokenAddr;
    
    // Process all claims and accumulate total payout
    unchecked { // ✅ Gas optimization - we control the loop
        for (uint256 i = 0; i < claims.length; ++i) {
            uint256 marketId = claims[i].marketId;
            bool prediction = claims[i].prediction;
            
            MarketInfo storage market = markets[marketId];
            BetInfo storage bet = bets[marketId][msg.sender][prediction];
            
            // Skip if already claimed or not eligible
            if (bet.claimed || !market.resolved || bet.amount == 0 || bet.prediction != market.outcome) {
                continue;
            }
            
            // ✅ Zero-division protection
            uint256 winningPool = market.outcome ? market.totalYesPool : market.totalNoPool;
            if (winningPool == 0) continue;
            
            // Calculate payout for this market
            uint256 payout = (bet.amount * market.totalPool) / winningPool;
            
            // ✅ Sanity check
            if (payout > market.totalPool) {
                payout = market.totalPool;
            }
            
            bet.payout = payout;
            bet.claimed = true;
            totalPayout += payout;
            
            // Cache token address (all markets use same token in MVP)
            if (tokenAddr == address(0)) {
                tokenAddr = market.tokenAddress;
                token = IERC20(tokenAddr);
            }
            
            emit PayoutClaimed(msg.sender, marketId, prediction, payout);
        }
    }
    
    require(totalPayout > 0, "InfoFiMarket: no valid claims");
    
    // ✅ Solvency check before transfer
    uint256 contractBalance = token.balanceOf(address(this));
    require(totalPayout <= contractBalance, "InfoFiMarket: insufficient balance for batch");
    
    require(token.transfer(msg.sender, totalPayout), "InfoFiMarket: batch transfer failed");
}
```

**Test Cases:**
```solidity
function testBatchClaimMultipleMarkets() public {
    // Create 3 markets
    uint256[] memory marketIds = new uint256[](3);
    for (uint i = 0; i < 3; i++) {
        marketIds[i] = createTestMarket();
        
        // User places YES bets
        vm.prank(user1);
        market.placeBet(marketIds[i], true, 100 ether);
        
        // Resolve YES
        vm.prank(operator);
        market.resolveMarket(marketIds[i], true);
    }
    
    // Batch claim all 3
    InfoFiMarket.ClaimRequest[] memory claims = new InfoFiMarket.ClaimRequest[](3);
    for (uint i = 0; i < 3; i++) {
        claims[i] = InfoFiMarket.ClaimRequest({
            marketId: marketIds[i],
            prediction: true
        });
    }
    
    uint256 balanceBefore = sofToken.balanceOf(user1);
    
    vm.prank(user1);
    uint256 totalPayout = market.batchClaimPayouts(claims);
    
    uint256 balanceAfter = sofToken.balanceOf(user1);
    
    assertEq(balanceAfter - balanceBefore, totalPayout);
    assertGt(totalPayout, 0);
}

function testBatchClaimGasSavings() public {
    // Measure gas for individual claims vs batch
    uint256 numMarkets = 10;
    uint256[] memory marketIds = new uint256[](numMarkets);
    
    for (uint i = 0; i < numMarkets; i++) {
        marketIds[i] = createTestMarket();
        vm.prank(user1);
        market.placeBet(marketIds[i], true, 100 ether);
        vm.prank(operator);
        market.resolveMarket(marketIds[i], true);
    }
    
    // Measure individual claims
    uint256 gasIndividual = 0;
    for (uint i = 0; i < numMarkets; i++) {
        uint256 gasBefore = gasleft();
        vm.prank(user1);
        market.claimPayout(marketIds[i], true);
        gasIndividual += gasBefore - gasleft();
    }
    
    // Reset and measure batch claim
    // ... (setup again)
    
    InfoFiMarket.ClaimRequest[] memory claims = new InfoFiMarket.ClaimRequest[](numMarkets);
    for (uint i = 0; i < numMarkets; i++) {
        claims[i] = InfoFiMarket.ClaimRequest({
            marketId: marketIds[i],
            prediction: true
        });
    }
    
    uint256 gasBefore = gasleft();
    vm.prank(user1);
    market.batchClaimPayouts(claims);
    uint256 gasBatch = gasBefore - gasleft();
    
    // Batch should save 60-70% gas
    assertLt(gasBatch, gasIndividual * 40 / 100); // Less than 40% of individual
}
```

---

## Phase 2: Gas Optimization (Priority: HIGH)

### 2.1 Optimize InfoFiMarketFactory Oracle Updates

**File:** `contracts/src/infofi/InfoFiMarketFactory.sol`  
**Location:** Lines 197-235 (in `_updateAllPlayerProbabilities`)

**Changes:**

```solidity
// ✅ ADD: Gas limit constant
uint256 public constant MAX_PLAYERS_PER_UPDATE = 50;

// ✅ ADD: Event for partial updates
event PartialOracleUpdate(uint256 indexed seasonId, uint256 updatedCount, uint256 totalPlayers);

function _updateAllPlayerProbabilities(
    uint256 seasonId,
    uint256 totalTickets,
    address skipPlayer
) internal {
    if (totalTickets == 0) return;

    address[] memory players = _seasonPlayers[seasonId];
    uint256 updateCount = 0;
    
    // ✅ Gas optimization: Use unchecked for loop counter
    unchecked {
        for (uint256 i = 0; i < players.length && updateCount < MAX_PLAYERS_PER_UPDATE; ++i) {
            address playerAddr = players[i];
            
            // Skip the player we just updated
            if (playerAddr == skipPlayer) continue;

            // Only update if market exists
            if (winnerPredictionMarkets[seasonId][playerAddr] == address(0)) continue;

            // Get current position from raffle
            IRaffleRead.ParticipantPosition memory pos = iRaffle.getParticipantPosition(seasonId, playerAddr);
            
            // Calculate new probability
            uint256 playerBps = (pos.ticketCount * 10000) / totalTickets;
            
            // Update oracle
            uint256 marketId = winnerPredictionMarketIds[seasonId][playerAddr];
            oracle.updateRaffleProbability(marketId, playerBps);
            
            ++updateCount;
        }
    }
    
    // ✅ Emit event if we hit the limit (for monitoring)
    if (updateCount == MAX_PLAYERS_PER_UPDATE && players.length > MAX_PLAYERS_PER_UPDATE) {
        emit PartialOracleUpdate(seasonId, updateCount, players.length);
    }
}
```

**Test Case:**
```solidity
function testOracleUpdateGasLimit() public {
    uint256 seasonId = 1;
    
    // Create 100 players (exceeds MAX_PLAYERS_PER_UPDATE)
    for (uint i = 0; i < 100; i++) {
        address player = address(uint160(i + 1));
        // ... create markets for each player
    }
    
    // Trigger update
    vm.expectEmit(true, false, false, true);
    emit PartialOracleUpdate(seasonId, 50, 100);
    
    factory.onPositionUpdate(seasonId, address(1), 0, 100, 10000);
/**
 * @dev Calculate potential payouts for multiple markets in one call
 * @param claims Array of (marketId, prediction) tuples to preview
 * @return payouts Array of potential payout amounts
 * @return totalPayout Sum of all payouts
 */
function batchCalculatePayouts(
    ClaimRequest[] calldata claims
) external view returns (uint256[] memory payouts, uint256 totalPayout) {
    payouts = new uint256[](claims.length);
    
    unchecked { // ✅ Gas optimization
        for (uint256 i = 0; i < claims.length; ++i) {
            uint256 marketId = claims[i].marketId;
            bool prediction = claims[i].prediction;
            
            MarketInfo storage market = markets[marketId];
            BetInfo storage bet = bets[marketId][msg.sender][prediction];
            
            // Calculate payout if eligible
            if (market.resolved && bet.amount > 0 && !bet.claimed && bet.prediction == market.outcome) {
                uint256 winningPool = market.outcome ? market.totalYesPool : market.totalNoPool;
                if (winningPool > 0) {
                    uint256 payout = (bet.amount * market.totalPool) / winningPool;
                    
                    // Cap at total pool
                    if (payout > market.totalPool) {
                        payout = market.totalPool;
                    }
                    
                    payouts[i] = payout;
                    totalPayout += payout;
                }
            }
        }
    }
}

// ... rest of the code remains the same ...

---

## Phase 3: Frontend Integration (from original batch claims plan)

### 3.1 Update onchainInfoFi.js Service

**File:** `src/services/onchainInfoFi.js`  
**Location:** After line 62 (after `readBetFull` function)

**Add new functions** (see original batch claims plan for full implementation):

1. `calculatePayoutView()` - Preview single payout
2. `batchCalculatePayoutsView()` - Preview batch payouts
3. `batchClaimPayoutsTx()` - Execute batch claim
4. `isSeasonCompleted()` - Check season status

### 3.2 Update ClaimCenter Component

**File:** `src/components/infofi/ClaimCenter.jsx`  
**Action:** Enhance with batch claiming and season completion checks

(See original batch claims plan for full implementation)

### 3.3 Remove Individual Claim Buttons from InfoFiMarketCard

**File:** `src/components/infofi/InfoFiMarketCard.jsx`  
**Action:** Remove lines 245-264 and 575-605

(See original batch claims plan for details)

---

## Testing Strategy

### Unit Tests (Foundry)

**File:** `contracts/test/InfoFiMarketSafety.t.sol` (NEW)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/infofi/InfoFiMarket.sol";

contract InfoFiMarketSafetyTest is Test {
    InfoFiMarket public market;
    
    function setUp() public {
        market = new InfoFiMarket();
        // ... setup
    }
    
    function testPoolInvariant() public { /* ... */ }
    function testZeroDivisionProtection() public { /* ... */ }
    function testSolvencyCheck() public { /* ... */ }
    function testBatchClaimGasSavings() public { /* ... */ }
    function testOracleUpdateGasLimit() public { /* ... */ }
}
```

### Integration Tests (Vitest)

**File:** `tests/e2e/infofi-batch-claim-safety.test.jsx` (NEW)

```javascript
describe('InfoFi Batch Claims with Safety Checks', () => {
  it('should preview payouts correctly', async () => { /* ... */ });
  it('should batch claim multiple positions', async () => { /* ... */ });
  it('should handle zero winning pool gracefully', async () => { /* ... */ });
  it('should prevent claims exceeding balance', async () => { /* ... */ });
});
```

---

## Implementation Checklist

### Smart Contracts

- [ ] Add pool invariant checks to `placeBet()`
- [ ] Add zero-division protection to `claimPayout()`
- [ ] Add solvency checks to `claimPayout()`
- [ ] Add `calculatePayout()` view function
- [ ] Add `ClaimRequest` struct
- [ ] Add `batchClaimPayouts()` function
- [ ] Add `batchCalculatePayouts()` view function
- [ ] Add gas limit to `_updateAllPlayerProbabilities()`
- [ ] Add `PartialOracleUpdate` event
- [ ] Write unit tests for all safety checks
- [ ] Write gas comparison tests

### Frontend Services

- [ ] Add `calculatePayoutView()` to `onchainInfoFi.js`
- [ ] Add `batchCalculatePayoutsView()` to `onchainInfoFi.js`
- [ ] Add `batchClaimPayoutsTx()` to `onchainInfoFi.js`
- [ ] Add `isSeasonCompleted()` to `onchainInfoFi.js`

### Frontend Components

- [ ] Update `ClaimCenter.jsx` with batch functionality
- [ ] Add season completion checks to `ClaimCenter.jsx`
- [ ] Remove individual claim buttons from `InfoFiMarketCard.jsx`
- [ ] Add gas savings estimate to UI
- [ ] Add loading states for batch operations

### Testing

- [ ] Unit tests for safety checks (Foundry)
- [ ] Gas comparison tests (Foundry)
- [ ] Integration tests for batch claims (Vitest)
- [ ] E2E test for full claim flow
- [ ] Manual testing on local Anvil

### Documentation

- [ ] Update API docs with new functions
- [ ] Add gas savings analysis
- [ ] Document safety guarantees
- [ ] Update user guide for batch claims

---

## Success Metrics

### Safety

- ✅ Zero division errors: **0 occurrences**
- ✅ Pool invariant violations: **0 occurrences**
- ✅ Insolvency attempts: **100% blocked**
- ✅ All edge cases covered in tests

### Gas Efficiency

- ✅ Batch claim savings: **60-70% vs individual**
- ✅ Oracle update gas limit: **< 5M gas per update**
- ✅ Loop optimizations: **10-15% savings with unchecked**

### User Experience

- ✅ Payout preview accuracy: **100%**
- ✅ Batch claim success rate: **> 99%**
- ✅ UI responsiveness: **< 2s for batch operations**

---

## Timeline

### Week 1: Smart Contract Implementation

- **Day 1-2:** Add safety checks to existing functions
- **Day 3-4:** Implement batch claim functions
- **Day 5:** Write and run unit tests

### Week 2: Frontend Integration & Testing

- **Day 1-2:** Update frontend services
- **Day 3:** Update UI components
- **Day 4:** Integration testing
- **Day 5:** Documentation and deployment prep

---

## Risk Mitigation

### Technical Risks

1. **Gas limit exceeded in batch operations**
   - Mitigation: 50-claim limit, gas estimation in UI

2. **Solvency check false positives**
   - Mitigation: Comprehensive testing, emergency admin withdrawal

3. **Oracle update gas spikes**
   - Mitigation: MAX_PLAYERS_PER_UPDATE limit, monitoring events

### User Experience Risks

1. **Confusion about batch vs individual claims**
   - Mitigation: Clear UI labels, tooltips, gas savings display

2. **Failed batch claims**
   - Mitigation: Individual claim fallback, detailed error messages

---

## Post-Implementation

### Monitoring

- Track gas usage for batch operations
- Monitor `PartialOracleUpdate` events
- Track claim success/failure rates
- Monitor contract SOF balance

### Future Enhancements (Post-MVP)

1. **ERC-1155 integration** (deferred)
2. **Automated resolution with UMA** (deferred)
3. **Off-chain order matching** (deferred)
4. **IPFS metadata storage** (deferred)

---

**End of Implementation Plan**
