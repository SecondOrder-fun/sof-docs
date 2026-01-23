# InfoFi Final Architecture Decision & Implementation Plan

**Date:** October 16, 2025  
**Status:** FINAL DECISION - Ready for Implementation  
**Based on:** LMSR Analysis + Polymarket Audit + Pseudo-Categorical Research

---

## Executive Summary

After comprehensive analysis of three key documents:
1. **LMSR Implementation Analysis** - Industry research on LMSR vs alternatives
2. **Polymarket Audit** - Gap analysis against industry standards
3. **Pseudo-Categorical Architecture** - Our unique season-level design

**FINAL DECISION: Hybrid FPMM + Pseudo-Categorical Architecture**

### Why NOT Pure LMSR?

The LMSR analysis reveals critical findings:

❌ **LMSR is being abandoned by major platforms**
- Augur V2 abandoned LMSR entirely
- Polymarket migrated from LMSR to CLOB
- Gnosis now offers both LMSR and FPMM (users prefer FPMM)

❌ **Capital inefficiency**
- Requires upfront funding: b = F / ln(2) ≈ 0.693F
- $100 market needs $69.31 locked permanently
- 100 concurrent markets = $6,931 locked capital

❌ **Guaranteed losses**
- Maximum loss = 0.693b for binary markets
- Subsidization required for sustainability
- No path to profitability without fees

❌ **Implementation complexity**
- Requires fixed-point math libraries (ABDKMath64x64)
- Exponential/logarithm operations (25,000 gas overhead)
- Numerical stability challenges (overflow protection)

✅ **LMSR's ONLY advantage: Bounded loss**
- Good for treasury budgeting
- But FPMM with proper liquidity management achieves same goal

### Why FPMM (Fixed Product Market Maker)?

✅ **Industry proven**
- Gnosis production deployment
- Polkamarkets, xParametric using it
- Battle-tested contracts available

✅ **Gas efficient**
- ~20,000 gas cheaper than LMSR per trade
- Simple arithmetic (x × y = k)
- No exponentials or logarithms

✅ **Sustainable economics**
- User liquidity provision possible
- Fee collection supports operations
- No guaranteed losses

✅ **Simpler implementation**
- Elementary math only
- Lower audit surface area
- Easier to maintain

✅ **Our pseudo-categorical design works perfectly with FPMM**
- Season-level liquidity pool
- Cross-market price effects maintained
- Simpler than LMSR categorical implementation

---

## Final Architecture: Pseudo-Categorical FPMM

### Core Design

**One FPMM instance per season** managing multiple player markets:

```
Season 1 FPMM:
├─ Shared liquidity pool (e.g., 10,000 SOF)
├─ Player A market (YES/NO reserves)
├─ Player B market (YES/NO reserves)
├─ Player C market (YES/NO reserves)
└─ Invariant: Σ(YES_i × NO_i) = k for all players
```

### Mathematical Foundation

**FPMM Formula (per player market):**
```
x × y = k

Where:
- x = YES token reserve
- y = NO token reserve
- k = constant product

Price: P(YES) = y / (x + y)
```

**Pseudo-Categorical Extension:**
```
For season with N players:
- Each player has independent (x_i, y_i) reserves
- Shared liquidity: L = Σ(x_i + y_i)
- Trading in one market affects others through shared pool
```

### Why This Works

1. **Reflects Reality**
   - All players compete in same raffle
   - Only ONE can win
   - Shared liquidity reflects shared outcome space

2. **Cross-Market Effects**
   - Trading in Player A affects available liquidity for Player B
   - Prices adjust naturally across all markets
   - Information propagates through liquidity changes

3. **Capital Efficiency**
   - One pool serves all markets
   - No per-market funding requirements
   - Dynamic liquidity allocation

4. **Gas Efficiency**
   - Simple arithmetic operations
   - No complex math libraries
   - ~120,000 gas per trade on L2

---

## Critical Findings from Polymarket Audit

### 🔴 CRITICAL: ERC-1155 Integration

**Current State:** Custom BetInfo struct, no tokenization

**Industry Standard:** Gnosis Conditional Token Framework (CTF)

**Impact:**
- ❌ No composability (can't trade positions)
- ❌ No DeFi integration (can't use as collateral)
- ❌ No secondary markets
- ❌ No flash loan arbitrage

**Decision:** **DEFER to Phase 3** (Month 2)
- **Rationale:** MVP can launch without ERC-1155
- Focus on core functionality first
- Add composability once proven

### 🟡 MODERATE: Resolution Mechanism

**Current State:** Manual operator resolution

**Industry Standard:** UMA Optimistic Oracle with challenge period

**Decision:** **Implement two-step resolution** (Phase 1)
- Proposal + 2-hour challenge period
- Bond requirement for proposals
- Escalation to admin if challenged
- Maintains decentralization while practical

### 🟡 MODERATE: Gas Optimization

**Current State:** All operations on-chain, no batching

**Industry Standard:** Off-chain matching + batch settlement

**Decision:** **Phased approach**
- Phase 1: Add batch operations
- Phase 2: Optimize storage packing
- Phase 4: Consider hybrid CLOB (if volume justifies)

---

## Implementation Plan: 4-Phase Approach

### Phase 1: Core FPMM + Security (Weeks 1-3)

**Smart Contracts:**

1. **SeasonFPMM.sol** - Season-level FPMM market maker
```solidity
contract SeasonFPMM {
    uint256 public immutable seasonId;
    uint256 public totalLiquidity;
    
    struct PlayerMarket {
        uint256 yesReserve;
        uint256 noReserve;
        uint256 k; // Constant product
        bool isActive;
    }
    
    mapping(uint256 => PlayerMarket) public playerMarkets; // playerId => market
    
    // FPMM pricing
    function getPrice(uint256 playerId, bool isYes) public view returns (uint256) {
        PlayerMarket memory market = playerMarkets[playerId];
        if (isYes) {
            return (market.noReserve * 1e18) / (market.yesReserve + market.noReserve);
        } else {
            return (market.yesReserve * 1e18) / (market.yesReserve + market.noReserve);
        }
    }
    
    // Calculate cost to buy shares
    function calcBuyCost(uint256 playerId, bool isYes, uint256 amount) 
        public view returns (uint256) 
    {
        PlayerMarket memory market = playerMarkets[playerId];
        
        if (isYes) {
            // Buy YES: need to maintain x' × y = k
            // x' = x - amount (YES tokens decrease)
            // y' = k / x'
            // cost = y' - y
            uint256 newYesReserve = market.yesReserve - amount;
            uint256 newNoReserve = market.k / newYesReserve;
            return newNoReserve - market.noReserve;
        } else {
            // Buy NO: similar logic
            uint256 newNoReserve = market.noReserve - amount;
            uint256 newYesReserve = market.k / newNoReserve;
            return newYesReserve - market.yesReserve;
        }
    }
    
    // Execute trade
    function buyShares(uint256 playerId, bool isYes, uint256 amount) 
        external 
        nonReentrant 
        returns (uint256 cost) 
    {
        cost = calcBuyCost(playerId, isYes, amount);
        
        // Transfer collateral from user
        require(collateralToken.transferFrom(msg.sender, address(this), cost));
        
        // Update reserves
        PlayerMarket storage market = playerMarkets[playerId];
        if (isYes) {
            market.yesReserve -= amount;
            market.noReserve += cost;
        } else {
            market.noReserve -= amount;
            market.yesReserve += cost;
        }
        
        // Verify invariant
        assert(market.yesReserve * market.noReserve == market.k);
        
        // Mint outcome tokens to user
        _mint(msg.sender, getPositionId(playerId, isYes), amount);
        
        emit SharesPurchased(msg.sender, playerId, isYes, amount, cost);
    }
}
```

2. **Enhanced InfoFiMarket.sol** - Integrate SeasonFPMM
```solidity
contract InfoFiMarket {
    mapping(uint256 => SeasonFPMM) public seasonFPMMs;
    
    function buyShares(uint256 marketId, bool isYes, uint256 amount) external {
        Market storage market = markets[marketId];
        SeasonFPMM fpmm = seasonFPMMs[market.seasonId];
        
        uint256 cost = fpmm.buyShares(market.playerId, isYes, amount);
        
        // Update user position tracking
        // ... existing logic ...
    }
}
```

3. **Two-Step Resolution** - Add challenge mechanism
```solidity
struct Resolution {
    bool proposed;
    bool outcome;
    uint256 proposedAt;
    address proposer;
    uint256 bond;
    bool challenged;
}

uint256 public constant CHALLENGE_PERIOD = 2 hours;
uint256 public constant RESOLUTION_BOND = 100 ether;

function proposeResolution(uint256 marketId, bool outcome) external payable {
    require(msg.value >= RESOLUTION_BOND);
    // ... implementation from audit ...
}

function challengeResolution(uint256 marketId) external payable {
    // ... implementation from audit ...
}

function finalizeResolution(uint256 marketId) external {
    // ... implementation from audit ...
}
```

4. **Safety Checks** - Add invariants and validations
```solidity
// Add to all pool operations
assert(market.totalYesPool + market.totalNoPool == market.totalPool);

// Add before payouts
require(winningPool > 0, "Zero winning pool");
require(payout <= contractBalance, "Insufficient balance");

// Add to oracle updates
require(updateCount < MAX_PLAYERS_PER_UPDATE, "Gas limit");
```

**Testing:**
- Unit tests for FPMM pricing (20+ cases)
- Integration tests for buy/sell flows
- Invariant tests for pool integrity
- Gas benchmarking vs LMSR

**Estimated Time:** 3 weeks

---

### Phase 2: Gas Optimization + Batch Operations (Weeks 4-5)

**Optimizations:**

1. **Storage Packing**
```solidity
struct PlayerMarket {
    uint128 yesReserve;  // Pack into one slot
    uint128 noReserve;   // Pack into one slot
    uint256 k;           // Separate slot
    bool isActive;       // Pack with other bools
}
```

2. **Batch Operations**
```solidity
function batchBuyShares(
    uint256[] calldata marketIds,
    bool[] calldata outcomes,
    uint256[] calldata amounts
) external nonReentrant {
    for (uint256 i = 0; i < marketIds.length; i++) {
        _buySharesInternal(marketIds[i], outcomes[i], amounts[i]);
    }
}

function batchClaimPayouts(
    uint256[] calldata marketIds,
    bool[] calldata outcomes
) external nonReentrant {
    for (uint256 i = 0; i < marketIds.length; i++) {
        _claimPayoutInternal(marketIds[i], outcomes[i]);
    }
}
```

3. **Unchecked Math**
```solidity
unchecked {
    for (uint256 i = 0; i < players.length; ++i) {
        // Safe: i can't overflow
    }
}
```

4. **Custom Errors**
```solidity
error InsufficientBalance();
error MarketNotResolved();
error AlreadyClaimed();

// Replace require strings with custom errors
if (balance < amount) revert InsufficientBalance();
```

**Testing:**
- Gas benchmarking before/after
- Verify no functionality regression
- Test batch operations with various sizes

**Estimated Time:** 2 weeks

---

### Phase 3: ERC-1155 Integration (Weeks 6-9)

**Implementation:**

1. **InfoFiConditionalTokens.sol** - ERC-1155 wrapper
```solidity
contract InfoFiConditionalTokens is ERC1155, AccessControl {
    // Gnosis CTF-compatible structure
    struct Condition {
        address oracle;
        bytes32 questionId;
        uint256 outcomeSlotCount;
    }
    
    function splitPosition(...) external {
        // Transfer collateral → mint outcome tokens
    }
    
    function mergePositions(...) external {
        // Burn outcome tokens → return collateral
    }
    
    function redeemPositions(...) external {
        // Burn winning tokens → payout
    }
}
```

2. **Migration Strategy**
- Deploy new CTF contract
- Keep existing markets on old system
- New markets use CTF
- Gradual migration over 1-2 months

**Benefits Unlocked:**
- ✅ Secondary market trading
- ✅ DeFi composability
- ✅ Flash loan arbitrage
- ✅ Liquidity provision

**Estimated Time:** 4 weeks

---

### Phase 4: Advanced Features (Weeks 10-16)

**Optional Enhancements:**

1. **Hybrid CLOB** (if volume > $10M/month)
   - Off-chain order matching
   - On-chain settlement
   - Requires infrastructure investment

2. **UMA Oracle Integration**
   - Replace two-step with UMA Optimistic Oracle
   - Full decentralization
   - Higher complexity

3. **Cross-Market Strategies**
   - Combinatorial markets
   - Multi-outcome betting
   - Advanced arbitrage

4. **Flash Loan Support**
   - Enable flash loan arbitrage
   - Requires ERC-1155 integration first

**Estimated Time:** 6-8 weeks

---

## Comparison: LMSR vs FPMM for Our Use Case

| Criterion | LMSR | FPMM | Winner |
|-----------|------|------|--------|
| **Gas Efficiency** | 150k per trade | 120k per trade | ✅ FPMM |
| **Implementation Complexity** | High (fixed-point math) | Low (elementary math) | ✅ FPMM |
| **Capital Efficiency** | Poor (locked funding) | Good (dynamic LP) | ✅ FPMM |
| **Sustainability** | Guaranteed losses | Fee-based profit | ✅ FPMM |
| **Industry Adoption** | Declining | Growing | ✅ FPMM |
| **Pseudo-Categorical Fit** | Complex | Natural | ✅ FPMM |
| **Bounded Loss** | Yes (0.693b) | Manageable with limits | 🟰 TIE |
| **Price Discovery** | Slightly better at extremes | Good overall | 🟰 TIE |

**FPMM wins 6/8 criteria**

---

## Addressing LMSR Analysis Recommendations

The LMSR analysis document recommends:

> "Start with Gnosis FPMM deployed on Polygon or Arbitrum"

✅ **We agree** - This aligns with our decision

> "Fund binary markets with $100-150 each"

✅ **We'll implement** - Per-season funding model

> "Implement 1-2% trading fees"

✅ **Planned** - Phase 2 fee structure

> "Integrate UMA Optimistic Oracle V3"

🟡 **Deferred** - Start with two-step, migrate to UMA in Phase 4

> "Build with clear upgrade paths"

✅ **Factory pattern** - Already planned

> "Focus optimization on highest-impact changes"

✅ **Phase 2 focus** - Storage packing, custom errors, batching

---

## Addressing Polymarket Audit Findings

### Critical Findings

1. **🔴 ERC-1155 Integration**
   - **Status:** Deferred to Phase 3
   - **Rationale:** MVP doesn't require composability
   - **Timeline:** Weeks 6-9

2. **🟡 Resolution Mechanism**
   - **Status:** Implementing in Phase 1
   - **Solution:** Two-step with challenge period
   - **Timeline:** Weeks 1-3

3. **🟡 Gas Optimization**
   - **Status:** Dedicated Phase 2
   - **Solution:** Batching, packing, custom errors
   - **Timeline:** Weeks 4-5

4. **🟡 Invariant Checks**
   - **Status:** Implementing in Phase 1
   - **Solution:** Assert statements in all pool ops
   - **Timeline:** Weeks 1-3

### Moderate Findings

All moderate findings addressed in phased approach.

---

## Pseudo-Categorical Architecture Preserved

**Our unique innovation remains intact:**

✅ **One FPMM per season** (not per market)  
✅ **Shared liquidity pool** across all player markets  
✅ **Cross-market price effects** through shared reserves  
✅ **Central source of truth** (raffle positions)  
✅ **Capital efficiency** (one pool serves all)

**FPMM makes this SIMPLER:**
- No complex LMSR categorical math
- Natural reserve management
- Easier to understand and audit
- Better gas efficiency

---

## Risk Mitigation

### Technical Risks

1. **FPMM price extremes**
   - **Mitigation:** Price bounds (0.1% - 99.9%)
   - **Mitigation:** Trade size limits (10% of reserves)

2. **Liquidity fragmentation**
   - **Mitigation:** Season-level pooling
   - **Mitigation:** Dynamic rebalancing

3. **Front-running**
   - **Mitigation:** Slippage protection (maxPrice)
   - **Mitigation:** Transaction deadlines

### Economic Risks

1. **Insufficient liquidity**
   - **Mitigation:** Treasury-funded initial pools
   - **Mitigation:** Fee reinvestment

2. **Market manipulation**
   - **Mitigation:** Trade size limits
   - **Mitigation:** Circuit breakers

3. **Loss scenarios**
   - **Mitigation:** Conservative initial funding
   - **Mitigation:** Fee buffer

---

## Success Metrics

### Phase 1 (Weeks 1-3)
- ✅ FPMM pricing accurate within 0.5%
- ✅ All safety checks passing
- ✅ Two-step resolution working
- ✅ Gas < 150k per trade on L2

### Phase 2 (Weeks 4-5)
- ✅ Gas reduced to < 120k per trade
- ✅ Batch operations 60%+ gas savings
- ✅ Storage optimized (40k+ gas saved)

### Phase 3 (Weeks 6-9)
- ✅ ERC-1155 integration complete
- ✅ Secondary markets functional
- ✅ Migration path validated

### Phase 4 (Weeks 10-16)
- ✅ Advanced features deployed
- ✅ Volume metrics justify enhancements
- ✅ User feedback incorporated

---

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| **Phase 1** | 3 weeks | FPMM core, security, two-step resolution |
| **Phase 2** | 2 weeks | Gas optimization, batch operations |
| **Phase 3** | 4 weeks | ERC-1155 integration, composability |
| **Phase 4** | 6-8 weeks | Advanced features (optional) |
| **Total** | **15-17 weeks** | Production-ready system |

---

## Clarifying Questions for User

Before proceeding with implementation, please confirm:

### 1. Initial Liquidity Strategy

**Question:** How should we fund initial season liquidity pools?

**Options:**
- A) Treasury funds each season with fixed amount (e.g., 10,000 SOF)
- B) Dynamic funding based on expected participation
- C) User LP provision from day one
- D) Hybrid (treasury seed + user LP)

**Recommendation:** Option A for MVP simplicity

### 2. Fee Structure

**Question:** What fee percentage for sustainability?

**Options:**
- A) 1% trading fee (conservative)
- B) 2% trading fee (moderate)
- C) Dynamic fees based on volume
- D) No fees initially (subsidized)

**Recommendation:** Option B (2% fee) - industry standard

### 3. ERC-1155 Priority

**Question:** Is composability critical for MVP?

**Options:**
- A) Yes - implement in Phase 1 (adds 2-3 weeks)
- B) No - defer to Phase 3 as planned
- C) Partial - basic ERC-1155 without full CTF

**Recommendation:** Option B (defer) - launch faster

### 4. Resolution Mechanism

**Question:** Two-step vs UMA Oracle for MVP?

**Options:**
- A) Two-step (simpler, faster to implement)
- B) UMA Oracle (more decentralized, complex)
- C) Start two-step, migrate to UMA later

**Recommendation:** Option C - pragmatic approach

### 5. Deployment Target

**Question:** Which L2 for initial deployment?

**Options:**
- A) Polygon (lowest gas, proven)
- B) Arbitrum (better security, higher gas)
- C) Base (Coinbase ecosystem, growing)
- D) Multi-chain from start

**Recommendation:** Option A (Polygon) - proven track record

---

## Next Steps

**Upon your approval:**

1. ✅ Finalize architecture decisions based on your answers
2. ✅ Create detailed technical specifications for Phase 1
3. ✅ Begin SeasonFPMM.sol implementation
4. ✅ Set up testing framework
5. ✅ Deploy to local Anvil for testing

**Estimated start:** Immediately upon confirmation

---

## Conclusion

**Final Architecture: Pseudo-Categorical FPMM**

This decision is based on:
- ✅ Industry trends (LMSR declining, FPMM growing)
- ✅ Gas efficiency (20k gas savings per trade)
- ✅ Implementation simplicity (elementary math)
- ✅ Sustainable economics (fee-based, no guaranteed losses)
- ✅ Perfect fit with our pseudo-categorical design
- ✅ Battle-tested Gnosis contracts available
- ✅ Clear upgrade path to advanced features

**We preserve our innovation** (pseudo-categorical season markets) while **choosing the superior mechanism** (FPMM over LMSR) based on comprehensive analysis.

**Ready to proceed with implementation upon your confirmation of the 5 clarifying questions above.** 🚀

---

**References:**
- [LMSR Implementation Analysis](../09-investigations/lmsr-implementation-analysis.md)
- [Polymarket Audit](../09-investigations/INFOFI_POLYMARKET_AUDIT.md)
- [Pseudo-Categorical Summary](./INFOFI_PSEUDO_CATEGORICAL_SUMMARY.md)
- [Gnosis FPMM Documentation](https://docs.gnosis.io/conditionaltokens/)
