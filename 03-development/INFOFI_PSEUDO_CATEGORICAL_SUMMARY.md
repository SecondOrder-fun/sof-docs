# InfoFi Pseudo-Categorical Market Architecture - Summary

**Date:** October 16, 2025  
**Status:** Architecture Decision  
**Impact:** Major simplification of LMSR implementation

---

## Key Decision: Pseudo-Categorical Markets

### Architecture Change

**From:** Independent LMSR per player market  
**To:** One LMSR per season (pseudo-categorical)

### Rationale

Each season is treated as a **categorical market** where:

- **Outcomes** = All players in the season
- **Each player** = Binary YES/NO sub-market
- **Shared liquidity** = One pool, constant `b` parameter
- **Cross-market impact** = Trading affects all markets (by design)

---

## Why This Works

### 1. Reflects Reality

- All player markets derive from **same raffle**
- **Only ONE player** can win
- Probabilities **must sum to 100%** across all players
- Trading reveals **season-wide information**

### 2. Gnosis Pattern

```
Traditional Categorical: "Who will win the World Cup?"
├─ 32 outcomes (teams)
├─ Probabilities sum to 100%
└─ Shared liquidity pool

Our Pseudo-Categorical: "Who will win Season X?"
├─ N outcomes (players with markets)
├─ Each outcome has YES/NO binary market
├─ YES probabilities sum to 100% across all players
└─ Shared liquidity pool per season
```

### 3. Mathematical Foundation

```
LMSR Cost Function: C(q) = b * ln(Σ exp(q_i / b))

For categorical with N outcomes:
- q_i = net quantity of outcome i sold
- Sum over all N outcomes
- Single b parameter

For our pseudo-categorical:
- Each player has YES/NO positions
- YES positions compete (sum to 100%)
- NO positions are complements
- Single b parameter shared across all player markets
```

---

## Benefits

✅ **Information aggregation** - Trading reveals season-wide sentiment  
✅ **Efficient liquidity** - One pool serves all markets  
✅ **Realistic pricing** - Prices reflect competitive dynamics  
✅ **Simpler architecture** - One LMSR per season vs. hundreds  
✅ **Gas efficiency** - Shared state reduces storage costs  
✅ **Cross-market arbitrage** - Sophisticated strategies emerge  
✅ **Central source of truth** - Raffle positions drive all prices

---

## Implementation Changes

### Smart Contracts

**New Contract:** `SeasonLMSR.sol`

```solidity
contract SeasonLMSR {
    uint256 public immutable seasonId;
    uint256 public funding; // Shared liquidity
    uint256 public playerCount; // Number of outcomes
    
    // Per-player state
    struct PlayerMarketState {
        int256 netYesTokensSold;
        int256 netNoTokensSold;
        bool isActive;
    }
    
    mapping(uint256 => PlayerMarketState) public playerStates;
    
    // Register new player market
    function registerPlayerMarket(uint256 playerId) external;
    
    // Calculate prices using shared LMSR
    function calcMarginalPrice(uint256 playerId, bool isYes) 
        public view returns (uint256);
    
    // Calculate cost for trade
    function calcNetCost(uint256 playerId, bool isYes, int256 amount) 
        public view returns (int256);
}
```

**Updated:** `InfoFiMarket.sol`

```solidity
contract InfoFiMarket {
    // One LMSR per season
    mapping(uint256 => SeasonLMSR) public seasonLMSRs;
    
    // Buy/sell uses season's LMSR
    function buyShares(uint256 marketId, bool isYes, uint256 amount) external {
        uint256 seasonId = markets[marketId].seasonId;
        uint256 playerId = markets[marketId].playerId;
        
        SeasonLMSR lmsr = seasonLMSRs[seasonId];
        int256 cost = lmsr.calcNetCost(playerId, isYes, int256(amount));
        
        // ... execute trade ...
    }
}
```

### Frontend Services

**Updated:** `onchainInfoFi.js`

```javascript
/**
 * Get prices for a player market from season's LMSR
 */
export async function getMarketPrices(marketAddress, marketId) {
  // Get season ID and player ID
  const market = await publicClient.readContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'markets',
    args: [marketId],
  });
  
  // Get season's LMSR
  const seasonLMSR = await publicClient.readContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'seasonLMSRs',
    args: [market.seasonId],
  });
  
  // Calculate prices from shared LMSR
  const yesPrice = await publicClient.readContract({
    address: seasonLMSR,
    abi: SeasonLMSRABI,
    functionName: 'calcMarginalPrice',
    args: [market.playerId, true],
  });
  
  return {
    yesPrice: Number(yesPrice) / 100,
    noPrice: 100 - (Number(yesPrice) / 100),
  };
}
```

---

## Cross-Market Effects (Feature, Not Bug)

### Example Scenario

**Season 1 with 3 players:**

- Player A: 40% win probability
- Player B: 35% win probability  
- Player C: 25% win probability

**User buys YES on Player A:**

- Player A probability increases to 45%
- Player B probability decreases to 33%
- Player C probability decreases to 22%
- **All prices adjust automatically**

### Why This Is Good

1. **Reflects reality** - If A's chances increase, others must decrease
2. **Information propagation** - Trading reveals insights across all markets
3. **Arbitrage opportunities** - Sophisticated traders can profit from imbalances
4. **Market efficiency** - Prices converge to true probabilities faster

---

## Comparison: Independent vs. Pseudo-Categorical

### Independent Markets (Original Plan)

```
Season 1:
├─ Player A Market (LMSR instance 1)
│  ├─ Funding: 1000 SOF
│  ├─ b parameter: 1443 SOF
│  └─ Isolated pricing
├─ Player B Market (LMSR instance 2)
│  ├─ Funding: 1000 SOF
│  ├─ b parameter: 1443 SOF
│  └─ Isolated pricing
└─ Player C Market (LMSR instance 3)
   ├─ Funding: 1000 SOF
   ├─ b parameter: 1443 SOF
   └─ Isolated pricing

Total: 3 LMSR instances, 3000 SOF locked
Problem: Probabilities can sum to > 100%
```

### Pseudo-Categorical (New Plan)

```
Season 1:
└─ Season LMSR (single instance)
   ├─ Funding: 3000 SOF (shared)
   ├─ b parameter: 2730 SOF
   ├─ Player A state (YES/NO)
   ├─ Player B state (YES/NO)
   └─ Player C state (YES/NO)

Total: 1 LMSR instance, 3000 SOF shared
Guarantee: YES probabilities sum to 100%
```

---

## Gas Efficiency Gains

### Per-Trade Gas Costs

**Independent Markets:**
- Read/write to isolated LMSR: ~150k gas
- No cross-market updates needed
- **Total: ~150k gas per trade**

**Pseudo-Categorical:**
- Read/write to shared LMSR: ~180k gas
- Prices for all markets updated automatically
- **Total: ~180k gas per trade**
- **But:** Eliminates need for separate oracle updates

### Deployment Costs

**Independent Markets:**
- Deploy LMSR per player: ~2M gas each
- 100 players = 200M gas
- **Total: ~200M gas for season setup**

**Pseudo-Categorical:**
- Deploy one LMSR per season: ~2.5M gas
- Register 100 players: ~50k gas each = 5M gas
- **Total: ~7.5M gas for season setup**
- **Savings: 96% reduction**

---

## Migration Path

### Phase 1: Deploy Season LMSR (Week 1)
- Create `SeasonLMSR.sol` contract
- Implement fixed-point math functions
- Add player registration
- Write unit tests

### Phase 2: Update InfoFiMarket (Week 2)
- Integrate `SeasonLMSR` mapping
- Update buy/sell functions
- Add complete set mechanics
- Test cross-market effects

### Phase 3: Frontend Integration (Week 3)
- Update service layer
- Add season LMSR queries
- Update UI components
- Test real-time pricing

### Phase 4: Testing & Deployment (Week 4)
- Comprehensive testing
- Gas optimization
- Documentation
- Deploy to testnet

---

## Success Criteria

✅ **One LMSR per season** deployed successfully  
✅ **YES probabilities sum to 100%** across all players  
✅ **Cross-market price updates** work correctly  
✅ **Gas costs** reduced by 90%+ for deployment  
✅ **Trading costs** remain under 200k gas  
✅ **Arbitrage opportunities** exploitable  
✅ **UI shows real-time** cross-market effects

---

## References

- [Gnosis LMSR Primer](https://gnosis-pm-js.readthedocs.io/en/v1.3.0/lmsr-primer.html)
- [Gnosis Categorical Markets](https://forum.gnosis.io/t/global-automated-marketmaker-gamm/609)
- [Polymarket Deep Dive](../09-investigations/polymarket-deep-dive.md)
- [Full Implementation Plan](./INFOFI_LMSR_IMPLEMENTATION_PLAN.md)

---

**Status:** Ready for implementation with pseudo-categorical architecture! 🚀
