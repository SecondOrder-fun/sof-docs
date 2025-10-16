# SecondOrder.fun InfoFi LMSR Market Implementation - FINAL PLAN v4.0

**Date:** October 16, 2025  
**Status:** Ready for Implementation  
**Based on:** Gnosis Categorical Markets + Polymarket Analysis  
**Estimated Time:** 3-4 weeks

---

## Executive Summary

This plan implements **Logarithmic Market Scoring Rule (LMSR)** prediction markets following Gnosis' **categorical market** architecture, where each season has one LMSR instance managing multiple player outcome markets. Key innovations:

1. **Pseudo-Categorical Markets** - One LMSR per season, multiple player positions as outcomes
2. **Shared Liquidity Pool** - All player markets in a season share the same `b` parameter
3. **Cross-Market Impact** - Trading in one market affects prices in others (by design)
4. **Central Source of Truth** - Raffle position data drives all market probabilities
5. **YES + NO = 10 $SOF Invariant** - Enables arbitrage and predictable pricing
6. **Polymarket-Inspired UI** - Clean, modern trading interface
7. **Canonical ABI Usage** - Single source of truth for contract interactions

---

## Pseudo-Categorical Market Architecture

### Concept: Season as Categorical Market

Instead of treating each player's market as independent, we treat **each season as one categorical market** where:

- **Outcomes** = All players in the season (Player A wins, Player B wins, Player C wins, etc.)
- **Each player market** = Binary sub-market (YES/NO) within the categorical structure
- **Shared liquidity** = One LMSR instance per season with constant `b` parameter
- **Cross-market effects** = Trading in one player's market affects prices in all others

### Why This Works

**1. Reflects Reality**
- All player markets derive from the same raffle
- Only ONE player can win the raffle
- Probabilities must sum to 100% across all players
- Trading activity reveals information about the entire season

**2. Gnosis Categorical Pattern**
```
Traditional Categorical: "Who will win the World Cup?"
- 32 outcomes (teams)
- Probabilities sum to 100%
- Shared liquidity pool

Our Pseudo-Categorical: "Who will win Season X?"
- N outcomes (players with markets)
- Each outcome has YES/NO binary market
- Probabilities sum to 100% across all YES positions
- Shared liquidity pool per season
```

**3. Mathematical Elegance**
```
LMSR Cost Function: C(q) = b * ln(Σ exp(q_i / b))

For categorical with N outcomes:
- q_i = net quantity of outcome i sold
- Sum over all N outcomes
- Single b parameter for entire season

For our pseudo-categorical:
- Each player has YES/NO positions
- YES positions compete (sum to 100%)
- NO positions are complements
- Single b parameter shared across all player markets
```

**4. Benefits**

✅ **Information aggregation** - Trading reveals season-wide sentiment  
✅ **Efficient liquidity** - One pool serves all markets  
✅ **Realistic pricing** - Prices reflect competitive dynamics  
✅ **Simpler architecture** - One LMSR per season vs. hundreds  
✅ **Gas efficiency** - Shared state reduces storage costs  
✅ **Cross-market arbitrage** - Sophisticated strategies emerge

### Implementation Strategy

**Per Season:**
- Deploy one `LMSRMarketMaker` instance
- Set `atomicOutcomeSlotCount` = number of players with markets
- Initialize funding based on expected season volume
- Track `lmsrStates` mapping: playerId → (netYesTokensSold, netNoTokensSold)

**Per Player Market:**
- Register as outcome in season's LMSR
- Track YES/NO positions separately
- Prices calculated from shared LMSR state
- Trading updates global season state

---

## Research Findings: Gnosis LMSR Architecture

### Key Insights from Gnosis Implementation

**1. Cost Function with Fixed-Point Math**
```
C(q) = b * ln(Σ exp(q_i / b))

Where:
- b = funding / ln(atomicOutcomeSlotCount)
- q_i = net quantity of outcome i sold
- funding = initial market maker funding
```

**2. Critical Implementation Details**

- **Fixed-point precision**: Uses 2^64 for internal calculations
- **Overflow protection**: `sumExpOffset` function prevents overflow by normalizing exponents
- **Net quantities**: Tracks `netOutcomeTokensSold` = minted - held by market
- **Marginal pricing**: Price = exp(q_i/b) / Σ exp(q_j/b)

**3. Gnosis Market Maker Pattern**

```solidity
// Gnosis uses a base MarketMaker contract with:
- ConditionalTokens integration (ERC-1155)
- Collateral token (IERC20)
- Funding amount (initial liquidity)
- Fee structure (optional)
- Atomic outcome slot count (number of outcomes)
```

**4. Complete Set Mechanics**

- **Minting**: collateral → full set of outcome tokens (1:1 ratio)
- **Redemption**: full set → collateral (maintains invariant)
- **Splitting**: Convert collateral to specific outcome positions
- **Merging**: Combine outcome positions back to collateral

---

## Phase 1: Smart Contract Architecture

### 1.1 Season-Level LMSR Market Maker

**File:** `contracts/src/infofi/SeasonLMSR.sol`

**Key Features:**
- One instance per season (pseudo-categorical market)
- Tracks multiple player markets as outcomes
- Shared liquidity pool with constant `b` parameter
- Fixed-point math with overflow protection
- Cross-market price impact by design

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

/**
 * @title SeasonLMSR
 * @notice Season-level LMSR for pseudo-categorical prediction markets
 * @dev One instance per season, manages all player markets as outcomes
 * 
 * Architecture:
 * - Pseudo-categorical: Each player is an outcome in the season
 * - Each outcome has YES/NO binary positions
 * - Shared liquidity pool (constant b parameter)
 * - Trading in one market affects all others
 * 
 * Key formulas:
 * - Cost function: C(q) = b * ln(Σ exp(q_i / b)) over all player outcomes
 * - Price: P(player_i YES) = exp(q_i_yes/b) / Σ exp(q_j_yes/b)
 * - Liquidity parameter: b = funding / ln(playerCount)
 */
contract SeasonLMSR is Ownable, Pausable {
    using Math for uint256;
    
    // Fixed-point precision (2^64 for compatibility with Gnosis)
    uint256 private constant ONE = 2**64;
    
    // Maximum exponent to prevent overflow (from Gnosis)
    int256 private constant EXP_LIMIT = 10000000000000000000; // 10 * 2^64
    
    // Complete set value (10 SOF in wei)
    uint256 public constant COMPLETE_SET_VALUE = 10e18;
    
    // Season parameters
    uint256 public immutable seasonId;
    uint256 public funding; // Initial market maker funding
    uint256 public playerCount; // Number of players with markets
    
    // Per-player state tracking (playerId => YES/NO net quantities)
    struct PlayerMarketState {
        int256 netYesTokensSold;
        int256 netNoTokensSold;
        bool isActive;
    }
    
    mapping(uint256 => PlayerMarketState) public playerStates;
    uint256[] public activePlayers; // Track all active player IDs
    
    IERC20 public immutable collateralToken; // SOF token
    
    event FundingUpdated(uint256 oldFunding, uint256 newFunding);
    event PlayerMarketCreated(uint256 indexed playerId);
    event Trade(
        uint256 indexed playerId,
        address indexed trader,
        bool isYes,
        int256 amount,
        uint256 cost
    );
    
    constructor(
        uint256 _seasonId,
        address _collateralToken,
        uint256 _initialFunding,
        address _initialOwner
    ) Ownable(_initialOwner) {
        require(_initialFunding > 0, "Funding must be positive");
        seasonId = _seasonId;
        collateralToken = IERC20(_collateralToken);
        funding = _initialFunding;
        playerCount = 0;
    }
    
    /**
     * @notice Register a new player market in this season
     * @dev Called by InfoFiMarketFactory when creating markets
     */
    function registerPlayerMarket(uint256 playerId) external onlyOwner {
        require(!playerStates[playerId].isActive, "Player already registered");
        
        playerStates[playerId] = PlayerMarketState({
            netYesTokensSold: 0,
            netNoTokensSold: 0,
            isActive: true
        });
        
        activePlayers.push(playerId);
        playerCount++;
        
        emit PlayerMarketCreated(playerId);
    }
    
    /**
     * @notice Calculate liquidity parameter b = funding / ln(2)
     */
    function getLiquidityParameter() public view returns (uint256) {
        return (funding * ONE) / LN_2;
    }
    
    /**
     * @notice Calculate cost level: b * ln(exp(q_yes/b) + exp(q_no/b))
     * @dev Uses sumExpOffset to prevent overflow
     */
    function calcCostLevel() public view returns (uint256) {
        uint256 b = getLiquidityParameter();
        
        // Calculate exp(q_yes/b) and exp(q_no/b) with offset
        (uint256 sum, int256 offset) = sumExpOffset(b);
        
        // Cost = b * (ln(sum) + offset)
        uint256 lnSum = ln(sum);
        int256 costLevel = int256((b * lnSum) / ONE) + (offset * int256(b)) / int256(ONE);
        
        return uint256(costLevel);
    }
    
    /**
     * @notice Calculate sum of exponentials with offset to prevent overflow
     * @dev Based on Gnosis implementation
     * @return sum The sum of exp((q_i - offset)/b)
     * @return offset The offset used for numerical stability
     */
    function sumExpOffset(uint256 b) internal view returns (uint256 sum, int256 offset) {
        // Find maximum q value for offset
        int256 maxQ = netYesTokensSold > netNoTokensSold ? netYesTokensSold : netNoTokensSold;
        offset = (maxQ * int256(ONE)) / int256(b);
        
        // Calculate exp((q_yes - offset*b)/b) + exp((q_no - offset*b)/b)
        int256 expYes = ((netYesTokensSold * int256(ONE)) / int256(b)) - offset;
        int256 expNo = ((netNoTokensSold * int256(ONE)) / int256(b)) - offset;
        
        require(expYes < EXP_LIMIT && expNo < EXP_LIMIT, "Exponent overflow");
        
        sum = exp(expYes) + exp(expNo);
    }
    
    /**
     * @notice Calculate marginal price of YES outcome
     * @return Price in basis points (0-10000 = 0%-100%)
     */
    function calcMarginalPrice(bool isYes) public view returns (uint256) {
        uint256 b = getLiquidityParameter();
        (uint256 sum, int256 offset) = sumExpOffset(b);
        
        int256 q = isYes ? netYesTokensSold : netNoTokensSold;
        int256 expQ = ((q * int256(ONE)) / int256(b)) - offset;
        
        uint256 numerator = exp(expQ);
        
        // Return as basis points (0-10000)
        return (numerator * 10000) / sum;
    }
    
    /**
     * @notice Calculate cost to buy outcome tokens
     * @param isYes True for YES, false for NO
     * @param amount Number of tokens to buy (can be negative for selling)
     * @return cost Cost in collateral tokens (negative means revenue from selling)
     */
    function calcNetCost(bool isYes, int256 amount) public view returns (int256 cost) {
        // Save current state
        int256 oldYes = netYesTokensSold;
        int256 oldNo = netNoTokensSold;
        uint256 oldCostLevel = calcCostLevel();
        
        // Calculate new state
        int256 newYes = isYes ? oldYes + amount : oldYes;
        int256 newNo = isYes ? oldNo : oldNo + amount;
        
        // Temporarily update state for calculation
        netYesTokensSold = newYes;
        netNoTokensSold = newNo;
        uint256 newCostLevel = calcCostLevel();
        
        // Restore state
        netYesTokensSold = oldYes;
        netNoTokensSold = oldNo;
        
        // Cost = new level - old level
        cost = int256(newCostLevel) - int256(oldCostLevel);
    }
    
    /**
     * @notice Fixed-point natural logarithm
     * @dev Simplified implementation - production should use library
     */
    function ln(uint256 x) internal pure returns (uint256) {
        require(x > 0, "ln(0) undefined");
        // Simplified ln calculation
        // Production: Use proper fixed-point math library
        uint256 result = 0;
        uint256 y = x;
        
        while (y >= 2 * ONE) {
            result += ONE;
            y = y / 2;
        }
        
        return result;
    }
    
    /**
     * @notice Fixed-point exponential
     * @dev Simplified implementation - production should use library
     */
    function exp(int256 x) internal pure returns (uint256) {
        if (x < 0) {
            return ONE / exp(-x);
        }
        
        // Simplified exp calculation
        // Production: Use proper fixed-point math library
        uint256 result = ONE;
        uint256 term = ONE;
        uint256 xAbs = uint256(x);
        
        for (uint256 i = 1; i <= 10; i++) {
            term = (term * xAbs) / (i * ONE);
            result += term;
            if (term < 100) break;
        }
        
        return result;
    }
    
    /**
     * @notice Update funding amount (owner only)
     */
    function updateFunding(uint256 newFunding) external onlyOwner {
        require(newFunding > 0, "Funding must be positive");
        uint256 oldFunding = funding;
        funding = newFunding;
        emit FundingUpdated(oldFunding, newFunding);
    }
    
    /**
     * @notice Pause trading (emergency)
     */
    function pause() external onlyOwner {
        _pause();
    }
    
    /**
     * @notice Unpause trading
     */
    function unpause() external onlyOwner {
        _unpause();
    }
}
```

### 1.2 Enhanced InfoFiMarket.sol Integration

**Updates to:** `contracts/src/infofi/InfoFiMarket.sol`

**Key Changes:**
1. Integrate LMSRMarketMaker for pricing
2. Add buy/sell functions using LMSR
3. Implement complete set mint/redeem
4. Maintain YES + NO = 10 SOF invariant
5. Keep existing batch claim and safety features

```solidity
// Add after existing imports
import "./LMSRMarketMaker.sol";

contract InfoFiMarket is AccessControl, ReentrancyGuard {
    // ... existing code ...
    
    // LMSR integration
    LMSRMarketMaker public lmsrMarketMaker;
    bool public lmsrEnabled = true;
    
    // Market state for LMSR
    struct LMSRState {
        int256 netYesTokensSold;
        int256 netNoTokensSold;
        uint256 totalLiquidity;
    }
    
    mapping(uint256 => LMSRState) public lmsrStates;
    
    constructor(...) {
        // ... existing constructor code ...
        
        // Initialize LMSR with 1000 SOF initial funding
        lmsrMarketMaker = new LMSRMarketMaker(
            address(sofToken),
            1000e18, // Initial funding
            msg.sender
        );
    }
    
    /**
     * @notice Buy outcome shares using LMSR pricing
     * @dev Replaces/enhances existing placeBet function
     */
    function buyShares(
        uint256 marketId,
        bool isYes,
        uint256 amount
    ) external nonReentrant whenNotPaused returns (uint256 cost) {
        Market storage market = markets[marketId];
        require(market.isActive, "Market not active");
        require(!market.isResolved, "Market resolved");
        require(lmsrEnabled, "LMSR disabled");
        
        // Calculate cost using LMSR
        int256 netCost = lmsrMarketMaker.calcNetCost(isYes, int256(amount));
        require(netCost > 0, "Invalid trade");
        cost = uint256(netCost);
        
        // Transfer SOF from user
        require(sofToken.transferFrom(msg.sender, address(this), cost), "Transfer failed");
        
        // Update LMSR state
        LMSRState storage state = lmsrStates[marketId];
        if (isYes) {
            state.netYesTokensSold += int256(amount);
        } else {
            state.netNoTokensSold += int256(amount);
        }
        state.totalLiquidity += cost;
        
        // Update user position
        Position storage pos = positions[marketId][msg.sender];
        if (isYes) {
            pos.yesAmount += amount;
        } else {
            pos.noAmount += amount;
        }
        
        // Update totals
        totalYesPool += isYes ? cost : 0;
        totalNoPool += isYes ? 0 : cost;
        totalPool += cost;
        
        emit SharesPurchased(msg.sender, marketId, isYes, amount, cost);
    }
    
    /**
     * @notice Sell outcome shares using LMSR pricing
     * @dev NEW - enables exit before resolution
     */
    function sellShares(
        uint256 marketId,
        bool isYes,
        uint256 amount
    ) external nonReentrant whenNotPaused returns (uint256 revenue) {
        Market storage market = markets[marketId];
        require(market.isActive, "Market not active");
        require(!market.isResolved, "Market resolved");
        require(lmsrEnabled, "LMSR disabled");
        
        Position storage pos = positions[marketId][msg.sender];
        require(
            isYes ? pos.yesAmount >= amount : pos.noAmount >= amount,
            "Insufficient shares"
        );
        
        // Calculate revenue using LMSR (negative cost)
        int256 netCost = lmsrMarketMaker.calcNetCost(isYes, -int256(amount));
        require(netCost < 0, "Invalid trade");
        revenue = uint256(-netCost);
        
        // Verify solvency
        require(revenue <= totalPool, "Insufficient liquidity");
        
        // Update LMSR state
        LMSRState storage state = lmsrStates[marketId];
        if (isYes) {
            state.netYesTokensSold -= int256(amount);
            pos.yesAmount -= amount;
        } else {
            state.netNoTokensSold -= int256(amount);
            pos.noAmount -= amount;
        }
        state.totalLiquidity -= revenue;
        
        // Update totals
        totalYesPool -= isYes ? revenue : 0;
        totalNoPool -= isYes ? 0 : revenue;
        totalPool -= revenue;
        
        // Transfer SOF to user
        require(sofToken.transfer(msg.sender, revenue), "Transfer failed");
        
        emit SharesSold(msg.sender, marketId, isYes, amount, revenue);
    }
    
    /**
     * @notice Mint complete set (10 SOF → 1 YES + 1 NO)
     * @dev Enables arbitrage when prices don't sum to 10 SOF
     */
    function mintCompleteSet(
        uint256 marketId,
        uint256 amount
    ) external nonReentrant whenNotPaused {
        Market storage market = markets[marketId];
        require(market.isActive, "Market not active");
        require(!market.isResolved, "Market resolved");
        
        uint256 cost = amount * COMPLETE_SET_VALUE;
        
        // Transfer SOF from user
        require(sofToken.transferFrom(msg.sender, address(this), cost), "Transfer failed");
        
        // Mint both YES and NO shares
        Position storage pos = positions[marketId][msg.sender];
        pos.yesAmount += amount;
        pos.noAmount += amount;
        
        // Update totals (both pools get equal amounts)
        totalYesPool += cost / 2;
        totalNoPool += cost / 2;
        totalPool += cost;
        
        emit CompleteSetMinted(msg.sender, marketId, amount, cost);
    }
    
    /**
     * @notice Redeem complete set (1 YES + 1 NO → 10 SOF)
     * @dev Enables arbitrage when prices don't sum to 10 SOF
     */
    function redeemCompleteSet(
        uint256 marketId,
        uint256 amount
    ) external nonReentrant whenNotPaused {
        Position storage pos = positions[marketId][msg.sender];
        require(pos.yesAmount >= amount, "Insufficient YES");
        require(pos.noAmount >= amount, "Insufficient NO");
        
        uint256 revenue = amount * COMPLETE_SET_VALUE;
        
        // Burn both YES and NO shares
        pos.yesAmount -= amount;
        pos.noAmount -= amount;
        
        // Update totals
        totalYesPool -= revenue / 2;
        totalNoPool -= revenue / 2;
        totalPool -= revenue;
        
        // Transfer SOF to user
        require(sofToken.transfer(msg.sender, revenue), "Transfer failed");
        
        emit CompleteSetRedeemed(msg.sender, marketId, amount, revenue);
    }
    
    // ... existing claimPayout, batchClaimPayouts, etc. remain unchanged ...
}
```

---

## Phase 2: Frontend Service Layer

### 2.1 LMSR Service Integration

**File:** `src/services/onchainInfoFi.js`

**Add canonical ABI imports:**
```javascript
import InfoFiMarketABI from '@/contracts/abis/InfoFiMarket.json';
import LMSRMarketMakerABI from '@/contracts/abis/LMSRMarketMaker.json';
import RaffleABI from '@/contracts/abis/Raffle.json';
```

**New functions for LMSR:**

```javascript
/**
 * Get current market prices from LMSR
 */
export async function getMarketPrices(marketAddress, marketId) {
  // Get LMSR state
  const lmsrState = await publicClient.readContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'lmsrStates',
    args: [marketId],
  });
  
  // Get LMSR market maker address
  const lmsrAddress = await publicClient.readContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'lmsrMarketMaker',
  });
  
  // Calculate prices
  const yesPrice = await publicClient.readContract({
    address: lmsrAddress,
    abi: LMSRMarketMakerABI,
    functionName: 'calcMarginalPrice',
    args: [true],
  });
  
  const noPrice = await publicClient.readContract({
    address: lmsrAddress,
    abi: LMSRMarketMakerABI,
    functionName: 'calcMarginalPrice',
    args: [false],
  });
  
  return {
    yesPrice: Number(yesPrice) / 100, // Convert basis points to percentage
    noPrice: Number(noPrice) / 100,
    netYesTokensSold: lmsrState.netYesTokensSold,
    netNoTokensSold: lmsrState.netNoTokensSold,
    totalLiquidity: lmsrState.totalLiquidity,
  };
}

/**
 * Calculate cost to buy shares
 */
export async function calculateBuyCost(marketAddress, isYes, amount) {
  const lmsrAddress = await publicClient.readContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'lmsrMarketMaker',
  });
  
  const cost = await publicClient.readContract({
    address: lmsrAddress,
    abi: LMSRMarketMakerABI,
    functionName: 'calcNetCost',
    args: [isYes, amount],
  });
  
  return cost;
}

/**
 * Calculate revenue from selling shares
 */
export async function calculateSellRevenue(marketAddress, isYes, amount) {
  const lmsrAddress = await publicClient.readContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'lmsrMarketMaker',
  });
  
  const cost = await publicClient.readContract({
    address: lmsrAddress,
    abi: LMSRMarketMakerABI,
    functionName: 'calcNetCost',
    args: [isYes, -amount], // Negative for selling
  });
  
  return -cost; // Return positive revenue
}

/**
 * Buy shares transaction
 */
export async function buySharesTx(marketAddress, marketId, isYes, amount) {
  const { request } = await publicClient.simulateContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'buyShares',
    args: [marketId, isYes, amount],
    account: walletClient.account,
  });
  
  return await walletClient.writeContract(request);
}

/**
 * Sell shares transaction
 */
export async function sellSharesTx(marketAddress, marketId, isYes, amount) {
  const { request } = await publicClient.simulateContract({
    address: marketAddress,
    abi: InfoFiMarketABI,
    functionName: 'sellShares',
    args: [marketId, isYes, amount],
    account: walletClient.account,
  });
  
  return await walletClient.writeContract(request);
}

// ... existing functions remain ...
```

---

## Phase 3: Polymarket-Inspired UI

### 3.1 Market Card Component

**Component:** `src/components/prediction/InfoFiMarketCard.jsx`

**Key Features:**
- Buy/Sell toggle
- YES/NO outcome selector with live prices
- Amount input with quick buttons
- Real-time cost/revenue calculation
- Position display with P&L
- Arbitrage opportunity alerts

**Structure:**
```jsx
<Card>
  <CardHeader>
    <PlayerInfo />
    <CurrentProbability />
  </CardHeader>
  
  <CardContent>
    <ProbabilityChart />
    
    <TradingInterface>
      <BuySellToggle />
      <OutcomeSelector />
      <AmountInput />
      <CostDisplay />
      <TradeButton />
    </TradingInterface>
    
    <PositionDisplay />
    <ArbitrageAlert />
    <MarketStats />
  </CardContent>
</Card>
```

---

## Implementation Checklist

### Phase 1: Smart Contracts (Week 1-2)

- [ ] Create `LMSRMarketMaker.sol` with Gnosis-based implementation
- [ ] Implement `calcCostLevel()` with fixed-point math
- [ ] Implement `sumExpOffset()` for overflow protection
- [ ] Implement `calcMarginalPrice()` for YES/NO
- [ ] Implement `calcNetCost()` for buy/sell
- [ ] Add proper `ln()` and `exp()` fixed-point functions
- [ ] Update `InfoFiMarket.sol` with LMSR integration
- [ ] Add `buyShares()` function
- [ ] Add `sellShares()` function
- [ ] Add `mintCompleteSet()` function
- [ ] Add `redeemCompleteSet()` function
- [ ] Add safety checks from INFOFI_IMPLEMENTATION_FINAL.md
- [ ] Add `batchClaimPayouts()` function
- [ ] Write comprehensive unit tests (20+ cases)
- [ ] Test LMSR pricing accuracy
- [ ] Test complete set arbitrage
- [ ] Deploy to local Anvil

### Phase 2: Frontend Services (Week 2-3)

- [ ] Add canonical ABI imports
- [ ] Implement `getMarketPrices()`
- [ ] Implement `calculateBuyCost()`
- [ ] Implement `calculateSellRevenue()`
- [ ] Implement `buySharesTx()`
- [ ] Implement `sellSharesTx()`
- [ ] Implement `mintCompleteSetTx()`
- [ ] Implement `redeemCompleteSetTx()`
- [ ] Add functions from INFOFI_IMPLEMENTATION_FINAL.md
- [ ] Verify no inline ABIs

### Phase 3: UI Components (Week 3-4)

- [ ] Redesign `InfoFiMarketCard.jsx`
- [ ] Add Buy/Sell toggle
- [ ] Add YES/NO selector with live prices
- [ ] Add amount input with quick buttons
- [ ] Add real-time cost calculation
- [ ] Add position display with P&L
- [ ] Create `ProbabilityChart.jsx`
- [ ] Create `ArbitrageAlert.jsx`
- [ ] Update `ClaimCenter.jsx`
- [ ] Remove old claim buttons

### Phase 4: Testing (Week 4)

- [ ] Unit tests for LMSR math
- [ ] Integration tests for buy/sell
- [ ] Test arbitrage scenarios
- [ ] Test batch claims
- [ ] E2E on Anvil
- [ ] Gas optimization
- [ ] UI/UX testing

### Phase 5: Documentation (Week 4)

- [ ] Document LMSR implementation
- [ ] Document Gnosis adaptations
- [ ] Document arbitrage mechanics
- [ ] Update API docs
- [ ] Create user guide

---

## Success Metrics

### Market Mechanics
- ✅ LMSR pricing matches Gnosis reference (±0.5%)
- ✅ YES + NO prices sum to 10 SOF (±0.1%)
- ✅ Complete set mint/redeem works
- ✅ Arbitrage opportunities exploitable
- ✅ Sell functionality pre-resolution

### Performance
- ✅ Buy transaction: < 200k gas
- ✅ Sell transaction: < 200k gas
- ✅ Price calculation: < 50k gas
- ✅ Batch claim: 60-70% savings

### Code Quality
- ✅ 100% canonical ABI usage
- ✅ Test coverage > 80%
- ✅ No critical security issues
- ✅ Gnosis patterns followed

---

## Risk Mitigation

1. **Fixed-Point Math Errors** → Use Gnosis reference, extensive testing
2. **Overflow Protection** → Implement `sumExpOffset` properly
3. **LMSR Complexity** → Start with binary markets only
4. **Gas Costs** → Optimize with unchecked math where safe
5. **Arbitrage Exploitation** → Monitor and adjust funding
6. **ABI Mismatches** → Automated `copy-abis.js` script

---

## Timeline

**Week 1:** LMSR contract implementation  
**Week 2:** InfoFiMarket integration + testing  
**Week 3:** Frontend services + UI components  
**Week 4:** Testing + documentation + deployment  

**Total:** 4 weeks to production-ready

---

## Key Differences from Original Plan

### What Changed:
1. **Gnosis LMSR** instead of custom implementation
2. **Fixed-point math** (2^64 precision) instead of floating point
3. **Overflow protection** via `sumExpOffset`
4. **Complete set mechanics** following Conditional Tokens pattern
5. **Proper cost function** with logarithmic calculations

### What Stayed:
1. YES + NO = 10 SOF invariant
2. Polymarket-inspired UI
3. Canonical ABI usage
4. Batch claim functionality
5. Safety checks and gas optimizations

---

## References

- [Gnosis Conditional Tokens Market Makers](https://github.com/gnosis/conditional-tokens-market-makers)
- [Gnosis LMSR Documentation](https://gnosis-pm-contracts.readthedocs.io/en/latest/LMSRMarketMaker.html)
- [Polymarket Deep Dive](./docs/09-investigations/polymarket-deep-dive.md)
- [LMSR Primer](https://gnosis-pm-js.readthedocs.io/en/v1.3.0/lmsr-primer.html)

---

**Ready to implement with Gnosis-proven architecture! 🚀**
