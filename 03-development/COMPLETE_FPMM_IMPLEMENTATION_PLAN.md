# Complete FPMM + Conditional Tokens Implementation Plan

## Executive Summary

**Current Status**: Incomplete FPMM implementation that doesn't track user positions
**Goal**: Build a production-ready prediction market system using Gnosis Conditional Tokens Framework + FPMM
**Approach**: Follow Polymarket/Gnosis reference architecture with adaptations for SecondOrder.fun

---

## Research Findings

### How Gnosis CTF + FPMM Actually Works

Based on Gnosis official implementation and Polymarket's production system:

1. **Conditional Tokens (ERC1155)** - Users hold outcome tokens, not just reserves
2. **FPMM as Liquidity Pool** - Market maker holds conditional tokens and provides liquidity
3. **Position IDs** - Derived from `conditionId` + `collectionId` + `collateralToken`
4. **Buy Flow**: User pays collateral → FPMM splits into outcome tokens → Transfers outcome tokens to user
5. **Sell Flow**: User sends outcome tokens → FPMM merges back to collateral → Returns collateral to user

### Key Insight: Our Current Implementation is Wrong

Our `SimpleFPMM.buy()` only updates reserves but **never gives users any tokens**. This is fundamentally broken.

**Correct Flow** (from Gnosis reference):
```solidity
function buy(uint investmentAmount, uint outcomeIndex, uint minOutcomeTokensToBuy) external {
    // 1. Calculate how many outcome tokens to buy
    uint outcomeTokensToBuy = calcBuyAmount(investmentAmount, outcomeIndex);
    
    // 2. Take user's collateral (SOF)
    collateralToken.transferFrom(msg.sender, address(this), investmentAmount);
    
    // 3. Split collateral into ALL outcome tokens via CTF
    conditionalTokens.splitPosition(...);
    
    // 4. Transfer the specific outcome token to user
    conditionalTokens.safeTransferFrom(
        address(this), 
        msg.sender, 
        positionIds[outcomeIndex], 
        outcomeTokensToBuy, 
        ""
    );
}
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    SecondOrder.fun Layer                     │
├─────────────────────────────────────────────────────────────┤
│  InfoFiMarketFactory                                         │
│  - Monitors player positions (via RafflePositionTracker)    │
│  - Creates markets when player crosses 1% threshold          │
│  - Manages market lifecycle                                  │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ├──> RaffleOracleAdapter
                 │    - Prepares conditions for each player
                 │    - Resolves conditions when VRF determines winner
                 │
                 └──> InfoFiFPMMV2Manager
                      - Deploys individual FPMM contracts per player
                      - Provides initial liquidity (100 SOF)
                      │
┌─────────────────┴────────────────────────────────────────────┐
│              Gnosis Conditional Tokens Layer                  │
├──────────────────────────────────────────────────────────────┤
│  ConditionalTokens (ERC1155)                                 │
│  - Holds all outcome tokens                                  │
│  - Users query: balanceOf(user, positionId)                  │
│                                                               │
│  SimpleFPMM (per player/season)                              │
│  - Holds conditional tokens as reserves                      │
│  - Splits collateral → outcome tokens on buy                 │
│  - Merges outcome tokens → collateral on sell                │
│  - Uses x * y = k pricing                                    │
└───────────────────────────────────────────────────────────────┘
```

---

## Complete Implementation Plan

### Phase 1: Fix SimpleFPMM Contract ⚠️ CRITICAL

**Problem**: Current implementation doesn't interact with ConditionalTokens at all

**Solution**: Implement proper Gnosis FPMM pattern

#### 1.1 Update SimpleFPMM Constructor

```solidity
contract SimpleFPMM is ERC20, ReentrancyGuard, ERC1155TokenReceiver {
    IConditionalTokens public immutable conditionalTokens;
    IERC20 public immutable collateralToken;
    bytes32 public immutable conditionId;
    
    uint256[] public positionIds; // NEW: Store position IDs for YES/NO
    uint256 public yesReserve;
    uint256 public noReserve;
    
    constructor(
        address _conditionalTokens,
        address _collateralToken,
        bytes32 _conditionId,
        address _treasury,
        string memory _name,
        string memory _symbol
    ) ERC20(_name, _symbol) {
        conditionalTokens = IConditionalTokens(_conditionalTokens);
        collateralToken = IERC20(_collateralToken);
        conditionId = _conditionId;
        treasury = _treasury;
        
        // Calculate position IDs for YES (index 0) and NO (index 1)
        positionIds = new uint256[](2);
        positionIds[0] = _calculatePositionId(0); // YES
        positionIds[1] = _calculatePositionId(1); // NO
    }
    
    function _calculatePositionId(uint256 outcomeIndex) internal view returns (uint256) {
        bytes32 collectionId = conditionalTokens.getCollectionId(
            bytes32(0), // parentCollectionId
            conditionId,
            1 << outcomeIndex // indexSet: 0b01 for YES, 0b10 for NO
        );
        return conditionalTokens.getPositionId(collateralToken, collectionId);
    }
}
```

#### 1.2 Implement Proper buy() Function

```solidity
function buy(
    bool buyYes,
    uint256 amountIn,
    uint256 minAmountOut
) external nonReentrant returns (uint256 amountOut) {
    require(amountIn > 0, "Zero amount");
    
    uint256 outcomeIndex = buyYes ? 0 : 1;
    
    // Calculate output using x * y = k
    amountOut = calcBuyAmount(buyYes, amountIn);
    require(amountOut >= minAmountOut, "Slippage exceeded");
    
    // Take user's collateral
    require(
        collateralToken.transferFrom(msg.sender, address(this), amountIn),
        "Transfer failed"
    );
    
    // Calculate fee
    uint256 fee = (amountIn * FEE_BPS) / 10000;
    uint256 amountInAfterFee = amountIn - fee;
    feesCollected += fee;
    
    // Approve CTF to spend collateral
    require(
        collateralToken.approve(address(conditionalTokens), amountInAfterFee),
        "Approval failed"
    );
    
    // Split collateral into outcome tokens
    uint256[] memory partition = new uint256[](2);
    partition[0] = 1; // 0b01 (YES)
    partition[1] = 2; // 0b10 (NO)
    
    conditionalTokens.splitPosition(
        address(collateralToken),
        bytes32(0), // parentCollectionId
        conditionId,
        partition,
        amountInAfterFee
    );
    
    // Update reserves
    if (buyYes) {
        yesReserve -= amountOut;
        noReserve += amountInAfterFee;
    } else {
        noReserve -= amountOut;
        yesReserve += amountInAfterFee;
    }
    
    // Transfer outcome tokens to buyer
    conditionalTokens.safeTransferFrom(
        address(this),
        msg.sender,
        positionIds[outcomeIndex],
        amountOut,
        ""
    );
    
    emit Trade(msg.sender, buyYes, amountIn, amountOut);
}
```

#### 1.3 Implement sell() Function

```solidity
function sell(
    bool sellYes,
    uint256 amountOut,
    uint256 maxAmountIn
) external nonReentrant returns (uint256 amountIn) {
    require(amountOut > 0, "Zero amount");
    
    uint256 outcomeIndex = sellYes ? 0 : 1;
    
    // Calculate input needed
    amountIn = calcSellAmount(sellYes, amountOut);
    require(amountIn <= maxAmountIn, "Slippage exceeded");
    
    // Take user's outcome tokens
    conditionalTokens.safeTransferFrom(
        msg.sender,
        address(this),
        positionIds[outcomeIndex],
        amountIn,
        ""
    );
    
    // Calculate fee
    uint256 fee = (amountOut * FEE_BPS) / (10000 - FEE_BPS);
    uint256 amountOutPlusFee = amountOut + fee;
    feesCollected += fee;
    
    // Update reserves
    if (sellYes) {
        yesReserve += amountIn;
        noReserve -= amountOutPlusFee;
    } else {
        noReserve += amountIn;
        yesReserve -= amountOutPlusFee;
    }
    
    // Merge positions back to collateral
    uint256[] memory partition = new uint256[](2);
    partition[0] = 1;
    partition[1] = 2;
    
    conditionalTokens.mergePositions(
        address(collateralToken),
        bytes32(0),
        conditionId,
        partition,
        amountOutPlusFee
    );
    
    // Return collateral to seller
    require(
        collateralToken.transfer(msg.sender, amountOut),
        "Transfer failed"
    );
    
    emit Trade(msg.sender, sellYes, amountIn, amountOut);
}
```

#### 1.4 Add ERC1155 Receiver Implementation

```solidity
function onERC1155Received(
    address operator,
    address from,
    uint256 id,
    uint256 value,
    bytes calldata data
) external override returns (bytes4) {
    require(
        msg.sender == address(conditionalTokens),
        "Only CTF"
    );
    return this.onERC1155Received.selector;
}

function onERC1155BatchReceived(
    address operator,
    address from,
    uint256[] calldata ids,
    uint256[] calldata values,
    bytes calldata data
) external override returns (bytes4) {
    require(
        msg.sender == address(conditionalTokens),
        "Only CTF"
    );
    return this.onERC1155BatchReceived.selector;
}
```

---

### Phase 2: Update InfoFiFPMMV2Manager

#### 2.1 Pass ConditionalTokens Address

```solidity
function createMarket(
    uint256 seasonId,
    address player,
    bytes32 conditionId
) external onlyRole(FACTORY_ROLE) nonReentrant returns (address fpmm, address lpToken) {
    // ... existing checks ...
    
    // Deploy SimpleFPMM with CTF address
    SimpleFPMM fpmmContract = new SimpleFPMM(
        address(conditionalTokens), // NEW
        address(collateralToken),
        conditionId,
        treasury,
        string(abi.encodePacked("FPMM-S", _uint2str(seasonId))),
        "FPMM"
    );
    
    // ... rest of implementation ...
}
```

#### 2.2 Initial Liquidity Provision

```solidity
// After deploying FPMM, add initial liquidity
collateralToken.approve(fpmm, INITIAL_FUNDING);

// Split initial funding into outcome tokens
uint256[] memory partition = new uint256[](2);
partition[0] = 1; // YES
partition[1] = 2; // NO

conditionalTokens.splitPosition(
    address(collateralToken),
    bytes32(0),
    conditionId,
    partition,
    INITIAL_FUNDING
);

// Transfer outcome tokens to FPMM to initialize reserves
uint256 yesPositionId = fpmmContract.positionIds(0);
uint256 noPositionId = fpmmContract.positionIds(1);

conditionalTokens.safeTransferFrom(
    address(this),
    fpmm,
    yesPositionId,
    INITIAL_FUNDING / 2,
    ""
);

conditionalTokens.safeTransferFrom(
    address(this),
    fpmm,
    noPositionId,
    INITIAL_FUNDING / 2,
    ""
);
```

---

### Phase 3: Frontend Integration

#### 3.1 Read User Positions from ConditionalTokens

```javascript
// src/services/onchainInfoFi.js

export async function readFpmmPosition({ 
  seasonId, 
  player, 
  account, 
  prediction, 
  networkKey = 'LOCAL' 
}) {
  const publicClient = createPublicClient(...);
  const addrs = getContractAddresses(networkKey);
  
  // 1. Get FPMM address
  const fpmmAddress = await publicClient.readContract({
    address: addrs.INFOFI_FPMM,
    abi: fpmmManagerAbi,
    functionName: 'getMarket',
    args: [BigInt(seasonId), getAddress(player)],
  });
  
  if (!fpmmAddress || fpmmAddress === '0x0000000000000000000000000000000000000000') {
    return { amount: 0n };
  }
  
  // 2. Get position IDs from FPMM
  const positionIds = await publicClient.readContract({
    address: fpmmAddress,
    abi: fpmmAbi,
    functionName: 'positionIds',
    args: [prediction ? 0 : 1], // 0 for YES, 1 for NO
  });
  
  // 3. Query ConditionalTokens for user's balance
  const balance = await publicClient.readContract({
    address: addrs.CONDITIONAL_TOKENS,
    abi: conditionalTokensAbi,
    functionName: 'balanceOf',
    args: [getAddress(account), positionIds],
  });
  
  return { amount: balance };
}
```

#### 3.2 Update placeBetTx to Use New buy() Signature

```javascript
export async function placeBetTx({ 
  marketId, 
  prediction, 
  amount, 
  networkKey = 'LOCAL', 
  seasonId, 
  player 
}) {
  // ... get FPMM address ...
  
  // Calculate minimum amount out (2% slippage)
  const expectedOut = await publicClient.readContract({
    address: fpmmAddress,
    abi: fpmmAbi,
    functionName: 'calcBuyAmount',
    args: [Boolean(prediction), parsed],
  });
  const minAmountOut = (expectedOut * 98n) / 100n;
  
  // Approve SOF for FPMM
  // ... approval logic ...
  
  // Execute buy
  const sim = await publicClient.simulateContract({
    address: fpmmAddress,
    abi: fpmmAbi,
    functionName: 'buy',
    args: [Boolean(prediction), parsed, minAmountOut],
    account: from,
  });
  
  const txHash = await walletClient.writeContract(sim.request);
  await publicClient.waitForTransactionReceipt({ hash: txHash });
  return txHash;
}
```

---

### Phase 4: Settlement & Redemption

#### 4.1 Resolve Conditions via RaffleOracleAdapter

```solidity
// contracts/src/infofi/RaffleOracleAdapter.sol

function batchResolveSeasonMarkets(
    uint256 seasonId,
    address[] calldata players,
    address winner
) external onlyRole(RESOLVER_ROLE) {
    for (uint256 i = 0; i < players.length; i++) {
        bytes32 conditionId = playerConditions[seasonId][players[i]];
        
        // Prepare payouts: [1, 0] if winner, [0, 1] if loser
        uint256[] memory payouts = new uint256[](2);
        if (players[i] == winner) {
            payouts[0] = 1; // YES wins
            payouts[1] = 0;
        } else {
            payouts[0] = 0;
            payouts[1] = 1; // NO wins
        }
        
        // Report to ConditionalTokens
        conditionalTokens.reportPayouts(
            _getQuestionId(seasonId, players[i]),
            payouts
        );
        
        emit ConditionResolved(seasonId, players[i], conditionId, players[i] == winner);
    }
}
```

#### 4.2 User Claims Winnings

```javascript
// Frontend: User redeems winning positions

export async function redeemPosition({ seasonId, player, networkKey = 'LOCAL' }) {
  const publicClient = createPublicClient(...);
  const walletClient = createWalletClient(...);
  const [from] = await walletClient.getAddresses();
  
  const addrs = getContractAddresses(networkKey);
  
  // Get condition ID
  const conditionId = await publicClient.readContract({
    address: addrs.ORACLE_ADAPTER,
    abi: oracleAdapterAbi,
    functionName: 'playerConditions',
    args: [BigInt(seasonId), getAddress(player)],
  });
  
  // Redeem positions
  const txHash = await walletClient.writeContract({
    address: addrs.CONDITIONAL_TOKENS,
    abi: conditionalTokensAbi,
    functionName: 'redeemPositions',
    args: [
      addrs.SOF, // collateralToken
      bytes32(0), // parentCollectionId
      conditionId,
      [1, 2] // indexSets for YES and NO
    ],
    account: from,
  });
  
  await publicClient.waitForTransactionReceipt({ hash: txHash });
  return txHash;
}
```

---

## Implementation Checklist

### Contracts
- [ ] Update `SimpleFPMM.sol` with proper CTF integration
  - [ ] Add `positionIds` array
  - [ ] Implement `buy()` with `splitPosition` + `safeTransferFrom`
  - [ ] Implement `sell()` with `safeTransferFrom` + `mergePositions`
  - [ ] Add ERC1155 receiver functions
  - [ ] Add view functions for position IDs
  
- [ ] Update `InfoFiFPMMV2.sol` manager
  - [ ] Pass `conditionalTokens` address to SimpleFPMM constructor
  - [ ] Implement proper initial liquidity provision
  - [ ] Add ERC1155 receiver to manager
  
- [ ] Update `RaffleOracleAdapter.sol`
  - [ ] Ensure `prepareCondition` is called correctly
  - [ ] Implement `batchResolveSeasonMarkets` with `reportPayouts`
  
- [ ] Add `ConditionalTokens` address to deployment
  - [ ] Update `Deploy.s.sol`
  - [ ] Add to `.env.example`
  - [ ] Update `update-env-addresses.js`

### Frontend
- [ ] Update `src/services/onchainInfoFi.js`
  - [ ] Implement `readFpmmPosition()` using CTF `balanceOf`
  - [ ] Update `placeBetTx()` with new buy signature
  - [ ] Add `sellPositionTx()` function
  - [ ] Add `redeemPositionTx()` function
  
- [ ] Update `src/components/infofi/PositionsPanel.jsx`
  - [ ] Use `readFpmmPosition()` instead of `readBet()`
  - [ ] Display conditional token balances
  
- [ ] Update `src/components/infofi/ClaimCenter.jsx`
  - [ ] Add redemption functionality
  - [ ] Show redeemable positions after resolution
  
- [ ] Add `src/config/contracts.js`
  - [ ] Add `CONDITIONAL_TOKENS` address
  - [ ] Add `ORACLE_ADAPTER` address

### Testing
- [ ] Unit tests for SimpleFPMM
  - [ ] Test buy flow
  - [ ] Test sell flow
  - [ ] Test position ID calculation
  
- [ ] Integration tests
  - [ ] Create market → Buy position → Check balance
  - [ ] Resolve condition → Redeem position
  - [ ] Multiple users trading
  
- [ ] E2E tests
  - [ ] Full season lifecycle with InfoFi markets
  - [ ] UI displays positions correctly
  - [ ] Claim flow works end-to-end

---

## Questions for Clarification

### 1. ConditionalTokens Deployment
**Question**: Should we deploy our own ConditionalTokens contract or use an existing one?
- **Option A**: Deploy fresh instance (full control, easier testing)
- **Option B**: Use existing Gnosis deployment (if available on our network)
- **Recommendation**: Deploy our own for Anvil/testnet, consider existing for mainnet

### 2. Initial Liquidity Amount
**Question**: Current INITIAL_LIQUIDITY is 100 SOF. Is this appropriate?
- Creates 50/50 split (50 YES tokens, 50 NO tokens)
- Affects initial pricing and slippage
- **Recommendation**: Keep 100 SOF but make configurable

### 3. Fee Structure
**Question**: Current FEE_BPS is 200 (2%). Should we adjust?
- Polymarket uses 2% on CLOB
- Gnosis examples use 1-2%
- **Recommendation**: Keep 2% but make it adjustable per market

### 4. LP Token Distribution
**Question**: Who should receive LP tokens from initial liquidity?
- **Option A**: Treasury keeps them (can remove liquidity later)
- **Option B**: Burn them (permanent liquidity)
- **Option C**: Distribute to early traders
- **Recommendation**: Treasury keeps them for flexibility

### 5. Position Display in UI
**Question**: How should we display positions?
- Show raw conditional token amounts?
- Convert to estimated SOF value?
- Show both?
- **Recommendation**: Show both with clear labels

### 6. Sell Functionality
**Question**: Should users be able to sell positions back to FPMM?
- **Yes**: More liquid, better UX
- **No**: Simpler, forces holding until resolution
- **Recommendation**: Yes, implement sell for better UX

### 7. Migration Strategy
**Question**: What about existing trades (like the one in the transaction)?
- Those positions are lost (no tokens were minted)
- **Options**:
  - A) Ignore (small test amounts)
  - B) Airdrop equivalent positions
  - C) Refund SOF
- **Recommendation**: Option A for test environment, document for users

---

## Timeline Estimate

### Week 1: Core Contracts
- Days 1-2: Update SimpleFPMM with CTF integration
- Days 3-4: Update InfoFiFPMMV2Manager
- Day 5: Testing and debugging

### Week 2: Settlement & Frontend
- Days 1-2: RaffleOracleAdapter settlement logic
- Days 3-4: Frontend service layer updates
- Day 5: UI component updates

### Week 3: Testing & Polish
- Days 1-2: Integration tests
- Days 3-4: E2E tests
- Day 5: Documentation and deployment

**Total: ~3 weeks for complete implementation**

---

## Success Criteria

✅ **Contract Level**
- Users receive ERC1155 conditional tokens when buying
- Positions can be queried via `ConditionalTokens.balanceOf()`
- Sell functionality works correctly
- Settlement resolves conditions properly
- Winners can redeem positions for collateral

✅ **Frontend Level**
- PositionsPanel displays user's conditional token balances
- Trade widget executes buy/sell successfully
- Claim center shows redeemable positions
- All transactions reflect correctly in UI

✅ **Integration Level**
- Full season lifecycle works: create → trade → resolve → claim
- Multiple users can trade simultaneously
- Prices update correctly based on AMM formula
- No orphaned positions or lost funds

---

## Risk Mitigation

### Technical Risks
1. **CTF Complexity**: Use Gnosis reference implementation as guide
2. **Position ID Calculation**: Extensive unit tests
3. **Gas Costs**: Optimize by batching where possible

### Economic Risks
1. **Insufficient Liquidity**: Monitor and adjust INITIAL_LIQUIDITY
2. **Price Manipulation**: Implement position limits
3. **Impermanent Loss**: Treasury accepts this as cost of providing liquidity

### UX Risks
1. **Complexity**: Clear documentation and tooltips
2. **Transaction Failures**: Better error messages
3. **Confusion**: Educational content about conditional tokens

---

## Next Steps

**AWAITING YOUR APPROVAL TO PROCEED**

Once approved, I will:
1. Implement Phase 1 (SimpleFPMM fixes)
2. Test thoroughly
3. Move to Phase 2 (Manager updates)
4. Continue through all phases

Please review and provide feedback on:
- Architecture approach
- Questions for clarification
- Timeline expectations
- Any concerns or modifications needed
