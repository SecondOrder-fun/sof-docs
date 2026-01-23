# FPMM Implementation Progress

## ✅ Phase 1: SimpleFPMM Contract - COMPLETE

### Changes Made

#### 1. Added Position ID Tracking
- Added `uint256[2] public positionIds` to store YES/NO position IDs
- Implemented `_calculatePositionId()` to derive position IDs from condition and collection IDs
- Position IDs calculated in constructor using Gnosis CTF formulas

#### 2. Rewrote buy() Function
**Old behavior**: Only updated reserves, never gave users tokens
**New behavior**: 
- Takes user's SOF collateral
- Splits collateral into outcome tokens via `conditionalTokens.splitPosition()`
- Transfers outcome tokens to user via `conditionalTokens.safeTransferFrom()`
- Updates reserves correctly
- Users now receive ERC1155 conditional tokens they can hold/trade

#### 3. Added sell() Function
**New functionality**:
- Users can sell outcome tokens back to pool
- Takes user's conditional tokens via `safeTransferFrom()`
- Merges tokens back to collateral via `conditionalTokens.mergePositions()`
- Returns SOF to user
- Includes slippage protection

#### 4. Added Helper Functions
- `calcBuyAmount()`: Calculate expected outcome tokens for SOF input
- `calcSellAmount()`: Calculate outcome tokens needed for desired SOF output
- Both use constant product formula (x * y = k)

#### 5. Added ERC1155 Receiver
- `onERC1155Received()`: Required to receive conditional tokens
- `onERC1155BatchReceived()`: For batch transfers
- SimpleFPMM can now receive tokens from Conditional Tokens contract

#### 6. Updated IConditionalTokens Interface
- Added `safeTransferFrom()` for single token transfers
- Added `safeBatchTransferFrom()` for batch transfers
- Interface now complete for FPMM operations

### Files Modified
- `contracts/src/infofi/InfoFiFPMMV2.sol` - Complete rewrite of SimpleFPMM
- `contracts/src/infofi/interfaces/IConditionalTokens.sol` - Added transfer functions

### Key Improvements
✅ Users now receive actual tokens (ERC1155) when buying
✅ Positions can be queried via `ConditionalTokens.balanceOf(user, positionId)`
✅ Users can sell positions back to pool
✅ Proper integration with Gnosis Conditional Tokens Framework
✅ Follows Polymarket/Gnosis reference architecture

---

## 🔄 Phase 2: InfoFiFPMMV2Manager - IN PROGRESS

### What Needs to be Done

#### 1. Update createMarket() Function
Current issue: Manager doesn't provide initial liquidity correctly

**Required changes**:
```solidity
function createMarket(...) {
    // Deploy SimpleFPMM (already done)
    
    // NEW: Split initial funding into outcome tokens
    collateralToken.approve(address(conditionalTokens), INITIAL_FUNDING);
    
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
    
    // NEW: Transfer outcome tokens to FPMM to initialize reserves
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
    
    // FPMM now has 50 YES tokens and 50 NO tokens as reserves
}
```

#### 2. Add ERC1155 Receiver to Manager
Manager needs to receive conditional tokens during split operation:
```solidity
function onERC1155Received(...) external pure returns (bytes4) {
    return this.onERC1155Received.selector;
}

function onERC1155BatchReceived(...) external pure returns (bytes4) {
    return this.onERC1155BatchReceived.selector;
}
```

#### 3. Update getMarket() Function
Add view function to get FPMM address and position IDs:
```solidity
function getMarketDetails(uint256 seasonId, address player) 
    external view returns (
        address fpmm,
        uint256 yesPositionId,
        uint256 noPositionId
    ) {
    fpmm = playerMarkets[seasonId][player];
    if (fpmm != address(0)) {
        SimpleFPMM market = SimpleFPMM(fpmm);
        yesPositionId = market.positionIds(0);
        noPositionId = market.positionIds(1);
    }
}
```

---

## 📋 Phase 3: Frontend Integration - PENDING

### Required Changes

#### 1. Update onchainInfoFi.js Service

**Add readFpmmPosition()**:
```javascript
export async function readFpmmPosition({ seasonId, player, account, prediction, networkKey }) {
  // 1. Get FPMM address from manager
  const fpmmAddress = await getMarket(seasonId, player);
  
  // 2. Get position ID from FPMM
  const positionId = await fpmmContract.positionIds(prediction ? 0 : 1);
  
  // 3. Query ConditionalTokens for user's balance
  const balance = await conditionalTokens.balanceOf(account, positionId);
  
  return { amount: balance };
}
```

**Update placeBetTx()**:
```javascript
export async function placeBetTx({ seasonId, player, prediction, amount, networkKey }) {
  // Get FPMM address
  const fpmmAddress = await fpmmManager.getMarket(seasonId, player);
  
  // Calculate expected output with slippage
  const expectedOut = await fpmm.calcBuyAmount(prediction, parseUnits(amount, 18));
  const minAmountOut = (expectedOut * 98n) / 100n; // 2% slippage
  
  // Approve SOF
  await sofToken.approve(fpmmAddress, parseUnits(amount, 18));
  
  // Execute buy
  await fpmm.buy(prediction, parseUnits(amount, 18), minAmountOut);
}
```

**Add sellPositionTx()**:
```javascript
export async function sellPositionTx({ seasonId, player, prediction, amount, networkKey }) {
  const fpmmAddress = await fpmmManager.getMarket(seasonId, player);
  
  // Calculate expected SOF output
  const expectedOut = await fpmm.calcSellAmount(prediction, parseUnits(amount, 18));
  const minAmountOut = (expectedOut * 98n) / 100n;
  
  // Approve conditional tokens
  const positionId = await fpmm.positionIds(prediction ? 0 : 1);
  await conditionalTokens.setApprovalForAll(fpmmAddress, true);
  
  // Execute sell
  await fpmm.sell(prediction, parseUnits(amount, 18), maxAmountIn);
}
```

#### 2. Update PositionsPanel.jsx
Replace `readBet()` with `readFpmmPosition()`:
```javascript
const yes = await readFpmmPosition({ 
  seasonId: m.seasonId, 
  player: m.player, 
  account: address, 
  prediction: true, 
  networkKey: netKey 
});
```

#### 3. Add Redemption Flow
After market resolution, users need to redeem winning positions:
```javascript
export async function redeemPosition({ seasonId, player, networkKey }) {
  const conditionId = await oracleAdapter.playerConditions(seasonId, player);
  
  await conditionalTokens.redeemPositions(
    sofAddress,
    bytes32(0),
    conditionId,
    [1, 2] // Both YES and NO index sets
  );
}
```

---

## 📋 Phase 4: Settlement & Redemption - PENDING

### RaffleOracleAdapter Updates

**Add batchResolveSeasonMarkets()**:
```solidity
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
    }
}
```

### ClaimCenter Component
Add redemption UI for settled markets

---

## 📋 Phase 5: Testing & Deployment - PENDING

### Test Checklist
- [ ] Unit tests for SimpleFPMM buy/sell
- [ ] Integration test: Create market → Buy → Check balance
- [ ] Integration test: Sell position back to pool
- [ ] Integration test: Resolve condition → Redeem position
- [ ] E2E test: Full season lifecycle with trades
- [ ] Gas optimization review
- [ ] Security audit preparation

### Deployment Steps
1. Deploy ConditionalTokens contract
2. Update Deploy.s.sol with CTF address
3. Deploy all InfoFi contracts
4. Update .env files
5. Copy ABIs to frontend
6. Test on local Anvil
7. Deploy to testnet
8. Frontend testing
9. Production deployment

---

## Current Status Summary

✅ **Phase 1 Complete**: SimpleFPMM properly integrates with Conditional Tokens
- Users receive ERC1155 tokens when buying
- Can sell tokens back to pool
- Proper CTF integration following Gnosis/Polymarket patterns

🔄 **Phase 2 In Progress**: Need to update InfoFiFPMMV2Manager
- Initial liquidity provision needs CTF integration
- Manager needs ERC1155 receiver functions

⏳ **Phases 3-5 Pending**: Frontend, settlement, and testing

---

## Next Immediate Steps

1. Update `InfoFiFPMMV2Manager.createMarket()` to split initial funding
2. Add ERC1155 receiver to manager
3. Test contract compilation
4. Move to frontend integration

## Estimated Time Remaining

- Phase 2: 2-3 hours
- Phase 3: 4-6 hours  
- Phase 4: 3-4 hours
- Phase 5: 6-8 hours

**Total: ~15-21 hours remaining**
