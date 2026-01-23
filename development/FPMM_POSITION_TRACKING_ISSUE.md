# FPMM Position Tracking Issue

## Problem Summary

**Transaction Hash**: `0x9579954b9cf8bf95cca8f4d44bb2eec1013fc8f5119816d59e3f6185d4689b84`

✅ **Transaction Status**: SUCCESS  
❌ **UI Display**: No positions shown  

## Root Cause

The `SimpleFPMM.buy()` function **does not mint or transfer any tokens to the user**. It only updates internal reserves:

```solidity
function buy(bool buyYes, uint256 amountIn, uint256 minAmountOut) 
    external nonReentrant returns (uint256 amountOut) {
    // ... fee calculation ...
    
    // Updates reserves only
    if (buyYes) {
        yesReserve = newYesReserve;
        noReserve = newNoReserve;
    }
    
    // Takes user's SOF
    collateralToken.transferFrom(msg.sender, address(this), amountIn);
    
    // Emits event
    emit Trade(msg.sender, buyYes, amountIn, amountOut);
    
    // ❌ MISSING: No token minted/transferred to user!
}
```

**What happens**:
1. User sends 10 SOF to FPMM
2. FPMM updates `yesReserve` and `noReserve`
3. User gets... nothing

**What should happen**:
1. User sends 10 SOF to FPMM
2. FPMM mints conditional tokens to user
3. User holds ERC1155 tokens representing their position

## Why PositionsPanel Shows Nothing

`PositionsPanel.jsx` tries to read positions using `readBet()`, which queries:
- Old system: `InfoFiMarket.getBet(marketId, user, prediction)`
- New system: Should query conditional token balances

But the FPMM doesn't give users any tokens, so there's nothing to query!

## The Two Paths Forward

### Option 1: Proper Gnosis CTF Integration (Recommended)

Update `SimpleFPMM.buy()` to mint conditional tokens:

```solidity
function buy(bool buyYes, uint256 amountIn, uint256 minAmountOut) 
    external nonReentrant returns (uint256 amountOut) {
    // ... existing logic ...
    
    // NEW: Mint conditional tokens to user
    uint256 outcomeIndex = buyYes ? 0 : 1;
    uint256 positionId = conditionalTokens.getPositionId(
        collateralToken,
        conditionalTokens.getCollectionId(bytes32(0), conditionId, outcomeIndex)
    );
    
    // Split collateral into conditional tokens
    conditionalTokens.splitPosition(
        collateralToken,
        bytes32(0),
        conditionId,
        [1 << outcomeIndex],
        amountOut
    );
    
    // Transfer conditional tokens to buyer
    conditionalTokens.safeTransferFrom(
        address(this),
        msg.sender,
        positionId,
        amountOut,
        ""
    );
}
```

**Frontend changes needed**:
- Query `ConditionalTokens.balanceOf(user, positionId)` instead of `InfoFiMarket.getBet()`
- Calculate `positionId` from `conditionId` and outcome index

### Option 2: Simple Position Tracking (Quick Fix)

Add a mapping to track positions:

```solidity
// Add to SimpleFPMM
mapping(address => mapping(bool => uint256)) public positions;

function buy(bool buyYes, uint256 amountIn, uint256 minAmountOut) 
    external nonReentrant returns (uint256 amountOut) {
    // ... existing logic ...
    
    // NEW: Track user's position
    positions[msg.sender][buyYes] += amountOut;
    
    emit Trade(msg.sender, buyYes, amountIn, amountOut);
}

// Add view function
function getPosition(address user, bool outcome) 
    external view returns (uint256) {
    return positions[user][outcome];
}
```

**Frontend changes needed**:
- Call `SimpleFPMM.getPosition(user, true/false)` to read positions
- Update `readFpmmPosition()` to use this function

## Recommended Solution

**Use Option 2 (Simple Position Tracking)** for now because:

1. ✅ **Quick to implement** - Just add a mapping
2. ✅ **No CTF complexity** - Avoid dealing with ERC1155 position IDs
3. ✅ **Easier frontend** - Simple function call
4. ✅ **Backward compatible** - Can migrate to CTF later

**Later migrate to Option 1** when:
- Need composability with other DeFi protocols
- Want standard ERC1155 token positions
- Need to support secondary markets

## Implementation Steps

### 1. Update SimpleFPMM Contract

```solidity
// contracts/src/infofi/InfoFiFPMMV2.sol

contract SimpleFPMM is ERC20, ReentrancyGuard {
    // ... existing code ...
    
    // ADD: Position tracking
    mapping(address => uint256) public yesPositions;
    mapping(address => uint256) public noPositions;
    
    function buy(bool buyYes, uint256 amountIn, uint256 minAmountOut) 
        external nonReentrant returns (uint256 amountOut) {
        // ... existing logic ...
        
        // ADD: Track position
        if (buyYes) {
            yesPositions[msg.sender] += amountOut;
        } else {
            noPositions[msg.sender] += amountOut;
        }
        
        emit Trade(msg.sender, buyYes, amountIn, amountOut);
    }
    
    // ADD: View functions
    function getYesPosition(address user) external view returns (uint256) {
        return yesPositions[user];
    }
    
    function getNoPosition(address user) external view returns (uint256) {
        return noPositions[user];
    }
}
```

### 2. Update Frontend Service

```javascript
// src/services/onchainInfoFi.js

export async function readFpmmPosition({ seasonId, player, account, prediction, networkKey = 'LOCAL' }) {
  // ... get FPMM address ...
  
  const fpmmAbi = [
    {
      "type": "function",
      "name": "getYesPosition",
      "inputs": [{"name": "user", "type": "address"}],
      "outputs": [{"name": "", "type": "uint256"}],
      "stateMutability": "view"
    },
    {
      "type": "function",
      "name": "getNoPosition",
      "inputs": [{"name": "user", "type": "address"}],
      "outputs": [{"name": "", "type": "uint256"}],
      "stateMutability": "view"
    }
  ];
  
  const functionName = prediction ? 'getYesPosition' : 'getNoPosition';
  
  const position = await publicClient.readContract({
    address: fpmmAddress,
    abi: fpmmAbi,
    functionName,
    args: [getAddress(account)],
  });
  
  return { amount: position };
}
```

### 3. Update PositionsPanel

```javascript
// src/components/infofi/PositionsPanel.jsx

// Replace readBet with readFpmmPosition
const yes = await readFpmmPosition({ 
  seasonId: m.seasonId, 
  player: m.player, 
  account: address, 
  prediction: true, 
  networkKey: netKey 
});
```

### 4. Redeploy Contracts

```bash
# From contracts directory
forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --private-key $PRIVATE_KEY --broadcast

# Update .env
node scripts/update-env-addresses.js

# Restart frontend
npm run dev
```

## Testing Checklist

- [ ] Deploy updated SimpleFPMM contract
- [ ] Place a test trade
- [ ] Verify `getYesPosition(user)` returns correct amount
- [ ] Check PositionsPanel displays the position
- [ ] Test selling/closing positions
- [ ] Verify position updates after multiple trades

## Migration Path

For existing trades (like the one in the transaction):
- Positions are lost (no way to recover them)
- Users need to trade again after contract update
- Consider airdropping equivalent positions if significant value involved

## Related Files

- `contracts/src/infofi/InfoFiFPMMV2.sol` - FPMM contract
- `src/services/onchainInfoFi.js` - Frontend service
- `src/components/infofi/PositionsPanel.jsx` - UI component
- `contracts/script/Deploy.s.sol` - Deployment script
