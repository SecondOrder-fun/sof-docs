# Chainlink VRF v2 → v2.5 Migration Plan

**Status**: Planning Phase  
**Priority**: High (v2 deprecated Nov 29, 2024)  
**Scope**: Raffle.sol contract + deployment configuration  
**Estimated Effort**: 2-3 days (implementation + testing)

---

## Executive Summary

Your raffle settlement depends on Chainlink VRF v2, which **reached end-of-life on November 29, 2024**. This plan outlines a complete migration to VRF v2.5, which offers:

- ✅ Simplified request format (labeled objects vs positional args)
- ✅ Native token payment support (ETH + LINK)
- ✅ Better gas efficiency
- ✅ Continued Chainlink support
- ✅ Available on Base Mainnet & Base Sepolia

---

## Current Implementation Analysis

### What You Have (VRF v2)

**File**: `/contracts/src/core/Raffle.sol`

```solidity
// Current imports
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/VRFConsumerBaseV2.sol";
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/interfaces/VRFCoordinatorV2Interface.sol";

// Current contract inheritance
contract Raffle is RaffleStorage, AccessControl, ReentrancyGuard, VRFConsumerBaseV2 {
    
    // Current state variables
    VRFCoordinatorV2Interface private COORDINATOR;
    bytes32 public vrfKeyHash;
    uint64 public vrfSubscriptionId;
    uint32 public vrfCallbackGasLimit = 500000;
    uint16 public constant VRF_REQUEST_CONFIRMATIONS = 3;

    // Current request (lines 153-155, 180-182)
    uint256 requestId = COORDINATOR.requestRandomWords(
        vrfKeyHash, 
        vrfSubscriptionId, 
        VRF_REQUEST_CONFIRMATIONS, 
        vrfCallbackGasLimit, 
        seasons[seasonId].winnerCount
    );

    // Current fulfillment (line 189)
    function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
        // ... winner selection logic
    }
}
```

### Issues with Current Implementation

1. **Deprecated Base Class**: `VRFConsumerBaseV2` no longer receives updates
2. **Positional Arguments**: Request uses positional args (error-prone)
3. **No Native Token Support**: Only LINK payments
4. **Limited Gas Optimization**: Older implementation

---

## Phase 1: Smart Contract Migration

### Step 1.1: Update Imports

**File**: `/contracts/src/core/Raffle.sol`

**Current** (lines 8-9):

```solidity
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/VRFConsumerBaseV2.sol";
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/interfaces/VRFCoordinatorV2Interface.sol";
```

**New**:

```solidity
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol";
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/interfaces/IVRFCoordinatorV2Plus.sol";
import "chainlink-brownie-contracts/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";
```

**Why**:

- `VRFConsumerBaseV2Plus` is the v2.5 base class (actively maintained)
- `IVRFCoordinatorV2Plus` is the v2.5 coordinator interface
- `VRFV2PlusClient` provides the new request struct

### Step 1.2: Update Contract Inheritance

**Current** (line 24):

```solidity
contract Raffle is RaffleStorage, AccessControl, ReentrancyGuard, VRFConsumerBaseV2 {
```

**New**:

```solidity
contract Raffle is RaffleStorage, AccessControl, ReentrancyGuard, VRFConsumerBaseV2Plus {
```

### Step 1.3: Update State Variables

**Current** (lines 27-31):

```solidity
// VRF v2
VRFCoordinatorV2Interface private COORDINATOR;
bytes32 public vrfKeyHash;
uint64 public vrfSubscriptionId;
uint32 public vrfCallbackGasLimit = 500000;
```

**New**:

```solidity
// VRF v2.5
IVRFCoordinatorV2Plus private COORDINATOR;
bytes32 public vrfKeyHash;
uint256 public vrfSubscriptionId;  // Changed from uint64 to uint256 (v2.5 uses larger IDs)
uint32 public vrfCallbackGasLimit = 500000;  // Keep same value
```

**Why**:

- `IVRFCoordinatorV2Plus` is the v2.5 interface
- v2.5 subscription IDs are larger (uint256 vs uint64)
- Callback gas limit stays the same

### Step 1.4: Update Constructor

**Current** (lines 54-66):

```solidity
constructor(address _sofToken, address _vrfCoordinator, uint64 _vrfSubscriptionId, bytes32 _vrfKeyHash)
    VRFConsumerBaseV2(_vrfCoordinator)
{
    require(_sofToken != address(0), "Raffle: SOF zero");
    sofToken = IERC20(_sofToken);
    COORDINATOR = VRFCoordinatorV2Interface(_vrfCoordinator);
    vrfSubscriptionId = _vrfSubscriptionId;
    vrfKeyHash = _vrfKeyHash;

    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(SEASON_CREATOR_ROLE, msg.sender);
    _grantRole(EMERGENCY_ROLE, msg.sender);
}
```

**New**:

```solidity
constructor(address _sofToken, address _vrfCoordinator, uint256 _vrfSubscriptionId, bytes32 _vrfKeyHash)
    VRFConsumerBaseV2Plus(_vrfCoordinator)
{
    require(_sofToken != address(0), "Raffle: SOF zero");
    sofToken = IERC20(_sofToken);
    COORDINATOR = IVRFCoordinatorV2Plus(_vrfCoordinator);
    vrfSubscriptionId = _vrfSubscriptionId;
    vrfKeyHash = _vrfKeyHash;

    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(SEASON_CREATOR_ROLE, msg.sender);
    _grantRole(EMERGENCY_ROLE, msg.sender);
}
```

**Changes**:

- `uint64 _vrfSubscriptionId` → `uint256 _vrfSubscriptionId`
- `VRFConsumerBaseV2(_vrfCoordinator)` → `VRFConsumerBaseV2Plus(_vrfCoordinator)`
- `VRFCoordinatorV2Interface` → `IVRFCoordinatorV2Plus`

### Step 1.5: Update VRF Request Call (requestSeasonEnd)

**Current** (lines 152-155):

```solidity
uint256 requestId = COORDINATOR.requestRandomWords(
    vrfKeyHash, 
    vrfSubscriptionId, 
    VRF_REQUEST_CONFIRMATIONS, 
    vrfCallbackGasLimit, 
    seasons[seasonId].winnerCount
);
```

**New**:

```solidity
uint256 requestId = COORDINATOR.requestRandomWords(
    VRFV2PlusClient.RandomWordsRequest({
        keyHash: vrfKeyHash,
        subId: vrfSubscriptionId,
        requestConfirmations: VRF_REQUEST_CONFIRMATIONS,
        callbackGasLimit: vrfCallbackGasLimit,
        numWords: seasons[seasonId].winnerCount,
        extraArgs: VRFV2PlusClient._argsToBytes(
            VRFV2PlusClient.ExtraArgsV1({nativePayment: false})
        )
    })
);
```

**Why**:

- v2.5 uses labeled struct instead of positional args
- `extraArgs` allows specifying payment method (LINK or native)
- `nativePayment: false` = pay with LINK (set to `true` for ETH)

**Apply to both locations**:

- Line 152-155 in `requestSeasonEnd()`
- Line 180-182 in `requestSeasonEndEarly()`

### Step 1.6: Update fulfillRandomWords Signature

**Current** (line 189):

```solidity
function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
```

**New**:

```solidity
function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) internal override {
```

**Why**:

- v2.5 uses `calldata` instead of `memory` for gas efficiency
- Callback data is read-only, so `calldata` is more efficient

### Step 1.7: Update NatSpec Comment

**Current** (line 22):

```solidity
* @notice Manages seasons, deploys per-season RaffleToken and SOFBondingCurve, integrates VRF v2.
```

**New**:
```solidity
* @notice Manages seasons, deploys per-season RaffleToken and SOFBondingCurve, integrates VRF v2.5.
```

---

## Phase 2: Deployment Configuration

### Step 2.1: Update Environment Variables

**File**: `.env` (update for each network)

**Local Anvil** (no changes needed - use mock coordinator):

```bash
# Keep existing mock coordinator setup
```

**Base Sepolia** (testnet):

```bash
# VRF v2.5 Configuration for Base Sepolia
VRF_COORDINATOR_ADDRESS_TESTNET=0x5C210eC390E6c357B2Cc00181bD34dc2B3797e73
VRF_KEY_HASH_TESTNET=0x9e1b49a3bfe90b5b6853921322447c05f8068b0ab5cd4d00eda07e9fada085f1
VRF_SUBSCRIPTION_ID_TESTNET=<YOUR_SUBSCRIPTION_ID>  # Create via vrf.chain.link
```

**Base Mainnet** (production):

```bash
# VRF v2.5 Configuration for Base Mainnet
VRF_COORDINATOR_ADDRESS_MAINNET=0x5C210eC390E6c357B2Cc00181bD34dc2B3797e73
VRF_KEY_HASH_MAINNET=0x9e1b49a3bfe90b5b6853921322447c05f8068b0ab5cd4d00eda07e9fada085f1
VRF_SUBSCRIPTION_ID_MAINNET=<YOUR_SUBSCRIPTION_ID>  # Create via vrf.chain.link
```

**Reference**: [Chainlink VRF v2.5 Supported Networks](https://docs.chain.link/vrf/v2-5/supported-networks)

### Step 2.2: Update Deployment Script

**File**: `/contracts/script/deploy/00_DeployToSepolia.s.sol`

**Current** (example):

```solidity
address vrfCoordinator = 0x2Ca8E0C643bDe4C2E08ab7355Fb9B9B5b3F7124f;  // v2
uint64 subscriptionId = 123;
bytes32 keyHash = 0x...;
```

**New**:
```solidity
address vrfCoordinator = 0x5C210eC390E6c357B2Cc00181bD34dc2B3797e73;  // v2.5
uint256 subscriptionId = 123;  // Changed from uint64
bytes32 keyHash = 0x9e1b49a3bfe90b5b6853921322447c05f8068b0ab5cd4d00eda07e9fada085f1;
```

### Step 2.3: Create Subscription on Base Sepolia

**Steps**:

1. Go to [vrf.chain.link](https://vrf.chain.link)
2. Connect wallet with Base Sepolia selected
3. Click "Create Subscription"
4. Fund with LINK tokens (minimum 2 LINK recommended)
5. Note the subscription ID
6. Add Raffle contract as consumer
7. Update `.env` with subscription ID

---

## Phase 3: Testing Strategy

### Step 3.1: Unit Tests

**File**: `/contracts/test/RaffleVRF.t.sol` (update existing tests)

**Changes needed**:

- Update mock coordinator to v2.5 interface
- Update request struct in test calls
- Update subscription ID type (uint64 → uint256)
- Verify `calldata` vs `memory` in fulfillment

**Test cases to verify**:

```solidity
// 1. Request format is correct
function testVRFv25RequestFormat() public {
    // Verify VRFV2PlusClient.RandomWordsRequest struct is used
    // Verify extraArgs includes nativePayment flag
}

// 2. Fulfillment with calldata works
function testFulfillRandomWordsCalldata() public {
    // Verify fulfillRandomWords accepts calldata
    // Verify winner selection still works
}

// 3. Subscription ID type change
function testLargeSubscriptionId() public {
    // Test with uint256 subscription ID
    // Verify no overflow issues
}

// 4. Payment method configuration
function testNativePaymentFlag() public {
    // Test with nativePayment: false (LINK)
    // Test with nativePayment: true (ETH)
}
```

### Step 3.2: Integration Tests

**Local Anvil**:

1. Deploy Raffle with mock v2.5 coordinator
2. Create season
3. Buy tickets
4. Request season end
5. Simulate VRF callback
6. Verify winners selected correctly

**Base Sepolia**:

1. Deploy Raffle with real v2.5 coordinator
2. Create subscription and fund with LINK
3. Add Raffle as consumer
4. Create season
5. Buy tickets
6. Request season end
7. Wait for VRF callback (5-10 minutes)
8. Verify winners selected on-chain

### Step 3.3: Backwards Compatibility Check

**Verify**:

- ✅ Existing season data structures unchanged
- ✅ Winner selection logic unchanged
- ✅ Prize distribution unchanged
- ✅ Event emissions unchanged

---

## Phase 4: Deployment Checklist

### Pre-Deployment

- [ ] All imports updated to v2.5
- [ ] Contract inheritance updated
- [ ] State variables updated (uint64 → uint256)
- [ ] Constructor updated
- [ ] VRF request calls updated (both locations)
- [ ] fulfillRandomWords signature updated
- [ ] NatSpec comments updated
- [ ] Unit tests pass
- [ ] Integration tests pass on Anvil
- [ ] Code review completed

### Deployment to Base Sepolia

- [ ] Create VRF v2.5 subscription at vrf.chain.link
- [ ] Fund subscription with 5+ LINK
- [ ] Update `.env` with subscription ID
- [ ] Update deployment script with v2.5 addresses
- [ ] Deploy Raffle contract
- [ ] Add Raffle as consumer to subscription
- [ ] Verify contract addresses in `.env`
- [ ] Test end-to-end on Sepolia (create season, buy tickets, request end)
- [ ] Wait for VRF callback and verify winners

### Deployment to Base Mainnet

- [ ] Create VRF v2.5 subscription at vrf.chain.link
- [ ] Fund subscription with 10+ LINK
- [ ] Update `.env` with production subscription ID
- [ ] Update deployment script with mainnet addresses
- [ ] Deploy Raffle contract
- [ ] Add Raffle as consumer to subscription
- [ ] Verify contract addresses in `.env`
- [ ] Monitor first season end for VRF callback success

---

## Phase 5: Rollback Plan

If issues occur after deployment:

### Option A: Emergency Pause

```solidity
// Add emergency pause function
function pauseVRF() external onlyRole(EMERGENCY_ROLE) {
    // Prevent new VRF requests
    // Allow manual winner selection if needed
}
```

### Option B: Revert to v2

- Deploy new Raffle contract with v2 code
- Migrate active seasons to new contract
- Update frontend to use new address

### Option C: Manual Resolution

- If VRF callback fails, admin can manually select winners
- Use emergency role to bypass VRF requirement

---

## Timeline & Effort Estimate

| Phase | Task | Duration | Owner |
|-------|------|----------|-------|
| 1 | Smart contract updates | 2-3 hours | Dev |
| 2 | Configuration updates | 1 hour | Dev |
| 3 | Testing (unit + integration) | 4-6 hours | Dev + QA |
| 4 | Sepolia deployment | 2-3 hours | Dev |
| 4 | Mainnet deployment | 1-2 hours | Dev |
| **Total** | | **10-15 hours** | |

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| VRF callback fails | Low | High | Emergency pause + manual resolution |
| Subscription runs out of LINK | Medium | High | Monitor balance, auto-refill alerts |
| Contract size exceeds limit | Low | High | Already using storage contract pattern |
| Existing seasons break | Low | High | Thorough backwards compatibility testing |
| Deployment address mismatch | Medium | Medium | Double-check all addresses before deploy |

---

## Success Criteria

✅ All unit tests pass
✅ Integration tests pass on Anvil
✅ Integration tests pass on Base Sepolia
✅ VRF callback succeeds on Sepolia
✅ Winners selected correctly
✅ Prize distribution works
✅ No breaking changes to existing seasons
✅ Contract compiles without warnings

---

## References

- [Chainlink VRF v2.5 Getting Started](https://docs.chain.link/vrf/v2-5/getting-started)
- [VRF v2 to v2.5 Migration Guide](https://docs.chain.link/vrf/v2-5/migration-from-v2)
- [VRF v2.5 Supported Networks](https://docs.chain.link/vrf/v2-5/supported-networks)
- [VRFV2PlusClient Library](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol)

---

## Next Steps

1. **Review this plan** with the team
2. **Create feature branch**: `feature/vrf-v2-to-v25-migration`
3. **Implement Phase 1** (smart contract changes)
4. **Run Phase 3** (testing)
5. **Deploy to Sepolia** (Phase 4)
6. **Monitor for 1-2 seasons** before mainnet
7. **Deploy to Mainnet** (Phase 4)

