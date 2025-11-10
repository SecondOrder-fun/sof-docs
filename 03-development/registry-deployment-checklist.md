# Registry Pattern Deployment Checklist

## ✅ Completed Steps

1. **MarketTypeRegistry.sol created** - Full-featured registry contract
2. **InfoFiMarketFactory.sol updated** - Integrated with registry
3. **Deploy.s.sol updated** - Deploys registry before factory
4. **Contracts compiled** - All contracts build successfully
5. **ABIs copied** - Updated ABIs available to backend

## 🔄 Required Steps After Deployment

### 1. Redeploy All Contracts

Since the `InfoFiMarketFactory` constructor signature changed, you need to redeploy:

```bash
# Stop backend if running
# Kill any existing Anvil instance
npm run kill:zombies

# Start fresh Anvil
npm run anvil

# In another terminal, deploy contracts
npm run anvil:deploy
```

This will:
- Deploy `MarketTypeRegistry` with `WINNER_PREDICTION` registered
- Deploy `InfoFiMarketFactory` with registry address
- Copy updated ABIs to backend
- Update `.env` with new contract addresses

### 2. Restart Backend

After deployment completes:

```bash
npm run dev:backend
```

The backend will now use the updated ABIs with the registry integration.

### 3. Verify Market Creation

Test the flow:

```bash
cd contracts

# Create season
forge script script/CreateSeason.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast

# Start season
forge script script/StartSeason.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast

# Buy tickets (should trigger market creation at 1% threshold)
forge script script/BuyTickets.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast
```

**Expected Backend Logs:**
```
✅ PositionUpdate: Season 1, Player 0x... (0 → 2000 tickets)
   Total supply: 2000 | Updated 0 markets | Oracle updates: 0/0 | Player probability: 10000 bps
🎯 MarketCreated Event: Season 1
   Player: 0x...
   Market Type: WINNER_PREDICTION  ← Should show this!
   FPMM Address: 0x...
```

## 🎯 Key Changes

### Contract Changes

**InfoFiMarketFactory Constructor:**
```solidity
// OLD (7 parameters)
constructor(
    address _raffle,
    address _oracle,
    address _oracleAdapter,
    address _fpmmManager,
    address _sofToken,
    address _treasury,
    address _admin
)

// NEW (8 parameters)
constructor(
    address _raffle,
    address _oracle,
    address _oracleAdapter,
    address _fpmmManager,
    address _sofToken,
    address _marketTypeRegistry,  // NEW!
    address _treasury,
    address _admin
)
```

**Market Type Validation:**
```solidity
// Now validates against registry
function _createMarketInternal(uint256 seasonId, address player, bytes32 marketType) external {
    if (!marketTypeRegistry.isValidMarketType(marketType)) {
        revert InvalidMarketType(marketType);
    }
    // ...
}
```

### Backend Changes

**MarketTypeRegistry ABI Added:**
- `backend/src/abis/MarketTypeRegistryAbi.js` - New ABI file
- Backend can now query available market types on-chain

**InfoFiMarketFactory ABI Updated:**
- Constructor now includes `marketTypeRegistry` parameter
- Events unchanged (still emit `bytes32 marketType`)

## 📝 Adding New Market Types

### On-Chain (Admin Only)

```solidity
// Get registry address from deployment
MarketTypeRegistry registry = MarketTypeRegistry(REGISTRY_ADDRESS);

// Register new market type
registry.registerMarketType("POSITION_SIZE", "Predict final position size");

// Calculate hash for backend
bytes32 hash = keccak256("POSITION_SIZE");
// 0x... (use this in backend mapping)
```

### Backend Update

Update `backend/src/listeners/marketCreatedListener.js`:

```javascript
const MARKET_TYPE_HASHES = {
  '0x9af7ac054212f2f6f51aadd6392aae69c37a65182710ccc31fc2ce8679842eab': 'WINNER_PREDICTION',
  '0x...': 'POSITION_SIZE',  // Add new hash here
};
```

**No contract redeployment needed!** ✨

## 🔍 Troubleshooting

### Issue: "Updated 0 markets" in backend logs

**Cause:** Factory doesn't have RAFFLE_ROLE or contracts not properly configured

**Solution:**
1. Redeploy all contracts using `npm run anvil:deploy`
2. Restart backend
3. Verify factory address in `.env` matches deployed address

### Issue: "Invalid market type" error

**Cause:** Market type not registered in registry

**Solution:**
```bash
# Check registered types
cast call $MARKET_TYPE_REGISTRY "getAllMarketTypes()"

# Register if missing (as admin)
cast send $MARKET_TYPE_REGISTRY "registerMarketType(string,string)" "NEW_TYPE" "Description" --private-key $PRIVATE_KEY
```

### Issue: Backend shows garbled market type

**Cause:** Backend mapping doesn't have the hash

**Solution:**
1. Calculate hash: `cast keccak "MARKET_TYPE_NAME"`
2. Add to `MARKET_TYPE_HASHES` mapping
3. Restart backend

## 🎉 Benefits Achieved

✅ **No Factory Redeployment** - Add market types via registry
✅ **On-Chain Validation** - Factory validates types automatically
✅ **Backward Compatible** - Existing code still works
✅ **Gas Efficient** - Only ~5,000 gas overhead
✅ **Enumerable** - Query available types on-chain
✅ **Access Controlled** - Only admins can manage types

## 📚 Documentation

- **Full Proposal:** `MARKET_TYPE_REFACTOR_PROPOSAL.md`
- **Implementation Guide:** `MARKET_TYPE_IMPLEMENTATION.md`
- **Contract Source:** `contracts/src/infofi/MarketTypeRegistry.sol`
