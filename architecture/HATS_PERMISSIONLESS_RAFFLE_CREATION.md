# Permissionless Raffle Creation via Hats Protocol

**Author:** Briareos  
**Date:** February 6, 2026  
**Status:** Proposal / Implementation Plan  
**SOF Stake Requirement:** 50,000 $SOF (adjustable)

---

## Executive Summary

This document outlines a comprehensive implementation plan for integrating Hats Protocol into SecondOrder.fun to enable **permissionless raffle creation** through stake-based eligibility. Users who stake 50K $SOF receive a "Raffle Creator" Hat, which grants them the authority to create new seasons/raffles without requiring manual admin approval.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Hats Protocol Overview](#2-hats-protocol-overview)
3. [Architecture Design](#3-architecture-design)
4. [Smart Contract Integration](#4-smart-contract-integration)
5. [Staking Eligibility Module](#5-staking-eligibility-module)
6. [Safe Multisig Integration](#6-safe-multisig-integration)
7. [Implementation Phases](#7-implementation-phases)
8. [Security Considerations](#8-security-considerations)
9. [Gas Estimates & Costs](#9-gas-estimates--costs)
10. [Testing Strategy](#10-testing-strategy)
11. [Deployment Checklist](#11-deployment-checklist)
12. [Appendix: Contract Addresses](#appendix-contract-addresses)

---

## 1. Problem Statement

### Current State
- Raffle creation is gated by `SEASON_CREATOR_ROLE` in the Raffle contract
- This role is manually granted by the `DEFAULT_ADMIN_ROLE` (currently a team EOA/multisig)
- Creates centralization bottleneck and friction for community-driven raffles

### Desired State
- Anyone can create a raffle by staking 50K $SOF
- Stake acts as skin-in-the-game and spam prevention
- Misbehaving creators can be slashed (stake forfeited)
- Fully on-chain, permissionless, and auditable

---

## 2. Hats Protocol Overview

### What is Hats Protocol?
Hats Protocol is a DAO-native roles and permissions system where:
- **Hats** = ERC-1155 tokens representing roles
- **Wearers** = Addresses holding a hat token (can be EOA, multisig, contract)
- **Admins** = Hats that control other hats (hierarchical tree structure)
- **Eligibility Modules** = Contracts that determine who can wear a hat

### Key Components for This Integration

| Component | Purpose |
|-----------|---------|
| **Hats.sol** | Core protocol contract (already deployed on Base Sepolia & Mainnet) |
| **StakingEligibility.sol** | Eligibility module requiring token stake to wear a hat |
| **HatsSignerGate.sol** | Links hat wearers to Safe multisig signing authority |
| **HatsModuleFactory.sol** | Factory for deploying eligibility modules |

### Hats Protocol Addresses (Base Sepolia)

```
Hats.sol:              0x3bc1A0Ad72417f2d411118085256fC53CBdDd137
HatsModuleFactory:     0x0a3f85fa597B6a967271286aA0724811acDF5CD9
StakingEligibility:    0x... (implementation - deploy instance via factory)
```

> Note: Verify addresses at https://docs.hatsprotocol.xyz before deployment

---

## 3. Architecture Design

### Hats Tree Structure

```
                    ┌──────────────────────┐
                    │     Top Hat (0x1)    │
                    │   SecondOrder DAO    │
                    │  (Safe Multisig)     │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
   ┌──────────▼────────┐ ┌─────▼─────┐ ┌───────▼───────┐
   │  Raffle Creator   │ │   Judge   │ │  Recipient    │
   │   Hat (0x1.1)     │ │   Hat     │ │    Hat        │
   │                   │ │ (0x1.2)   │ │  (0x1.3)      │
   │  Eligibility:     │ │           │ │               │
   │  StakingEligibility│ │ Can slash │ │ Receives      │
   │  minStake: 50K SOF│ │ bad actors│ │ slashed stakes│
   └───────────────────┘ └───────────┘ └───────────────┘
```

### Data Flow

```
User wants to create raffle
        │
        ▼
┌───────────────────────┐
│ 1. Stake 50K $SOF     │
│    via StakingElig.   │
│    stake() function   │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ 2. Claim Hat          │
│    (auto-minted when  │
│    stake >= minStake) │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ 3. Create Raffle      │
│    Raffle.createSeason│
│    checks balanceOf   │
│    (Raffle Creator Hat│
└───────────────────────┘
```

---

## 4. Smart Contract Integration

### Current Raffle.sol Access Control

```solidity
// Current implementation
bytes32 public constant SEASON_CREATOR_ROLE = keccak256("SEASON_CREATOR_ROLE");

function createSeason(...) external onlyRole(SEASON_CREATOR_ROLE) {
    // ... season creation logic
}
```

### Modified Raffle.sol with Hats Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "openzeppelin-contracts/contracts/access/AccessControl.sol";
import "hats-protocol/src/Interfaces/IHats.sol";

contract Raffle is RaffleStorage, AccessControl, ReentrancyGuard, VRFConsumerBaseV2Plus {
    
    // Hats Protocol integration
    IHats public immutable HATS;
    uint256 public raffleCreatorHatId;
    
    // Keep existing role for backwards compatibility / emergency admin
    bytes32 public constant SEASON_CREATOR_ROLE = keccak256("SEASON_CREATOR_ROLE");
    
    // New modifier that checks hat OR role
    modifier canCreateSeason() {
        bool hasHat = raffleCreatorHatId != 0 && HATS.isWearerOfHat(msg.sender, raffleCreatorHatId);
        bool hasRole = hasRole(SEASON_CREATOR_ROLE, msg.sender);
        
        if (!hasHat && !hasRole) revert UnauthorizedCaller();
        _;
    }
    
    constructor(
        address _sofToken,
        address _vrfCoordinator,
        uint256 _vrfSubscriptionId,
        bytes32 _vrfKeyHash,
        address _hatsProtocol  // NEW: Hats contract address
    ) VRFConsumerBaseV2Plus(_vrfCoordinator) {
        // ... existing constructor logic ...
        HATS = IHats(_hatsProtocol);
    }
    
    /**
     * @notice Set the Hat ID that grants raffle creation rights
     * @param _hatId The Hats Protocol hat ID for Raffle Creators
     */
    function setRaffleCreatorHat(uint256 _hatId) external onlyRole(DEFAULT_ADMIN_ROLE) {
        raffleCreatorHatId = _hatId;
        emit RaffleCreatorHatUpdated(_hatId);
    }
    
    /**
     * @notice Create a new season (modified to use canCreateSeason modifier)
     */
    function createSeason(
        RaffleTypes.SeasonConfig memory config,
        RaffleTypes.BondStep[] memory bondSteps,
        uint16 buyFeeBps,
        uint16 sellFeeBps
    ) external canCreateSeason nonReentrant returns (uint256 seasonId) {
        // ... existing createSeason logic unchanged ...
    }
    
    /**
     * @notice Check if an address can create seasons
     * @param account Address to check
     * @return bool True if account can create seasons
     */
    function canCreateSeason(address account) external view returns (bool) {
        bool hasHat = raffleCreatorHatId != 0 && HATS.isWearerOfHat(account, raffleCreatorHatId);
        bool hasRole = hasRole(SEASON_CREATOR_ROLE, account);
        return hasHat || hasRole;
    }
}
```

### IHats Interface (Required Import)

```solidity
// src/lib/IHats.sol
interface IHats {
    function isWearerOfHat(address account, uint256 hatId) external view returns (bool);
    function balanceOf(address account, uint256 hatId) external view returns (uint256);
    function hatSupply(uint256 hatId) external view returns (uint32);
    function isActive(uint256 hatId) external view returns (bool);
    function isInGoodStanding(address account, uint256 hatId) external view returns (bool);
}
```

---

## 5. Staking Eligibility Module

### Module Configuration

```solidity
// Parameters for StakingEligibility deployment
struct StakingEligibilityConfig {
    address hatId;           // The hat this module controls eligibility for
    address stakingToken;    // $SOF token address
    uint256 minStake;        // 50,000 * 10^18 (50K SOF)
    uint256 judgeHatId;      // Hat ID that can slash
    uint256 recipientHatId;  // Hat ID that receives slashed stakes
    uint256 cooldownPeriod;  // Time before unstake completes (e.g., 7 days)
}
```

### Deployment Script

```solidity
// script/DeployHatsIntegration.s.sol
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "hats-module/src/HatsModuleFactory.sol";
import "hats-protocol/src/Hats.sol";

contract DeployHatsIntegration is Script {
    // Base Sepolia addresses
    address constant HATS = 0x3bc1A0Ad72417f2d411118085256fC53CBdDd137;
    address constant HATS_MODULE_FACTORY = 0x0a3f85fa597B6a967271286aA0724811acDF5CD9;
    address constant STAKING_ELIGIBILITY_IMPL = 0x...; // Get from Hats docs
    
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address sofToken = vm.envAddress("SOF_TOKEN");
        
        vm.startBroadcast(deployerPrivateKey);
        
        IHats hats = IHats(HATS);
        HatsModuleFactory factory = HatsModuleFactory(HATS_MODULE_FACTORY);
        
        // 1. Create Top Hat for SecondOrder DAO
        uint256 topHatId = hats.mintTopHat(
            msg.sender, // Initial wearer (transfer to Safe later)
            "SecondOrder DAO",
            "ipfs://..." // Image URI
        );
        
        // 2. Create Raffle Creator Hat (child of Top Hat)
        uint256 raffleCreatorHatId = hats.createHat(
            topHatId,                    // Admin hat
            "Raffle Creator",            // Details
            100,                         // Max supply (100 concurrent creators)
            address(0),                  // Eligibility (set after module deploy)
            address(0),                  // Toggle (active by default)
            true,                        // Mutable
            "ipfs://..."                 // Image URI
        );
        
        // 3. Create Judge Hat
        uint256 judgeHatId = hats.createHat(
            topHatId,
            "Raffle Judge",
            5,                           // Max 5 judges
            address(0),
            address(0),
            true,
            "ipfs://..."
        );
        
        // 4. Create Recipient Hat (for slashed stakes)
        uint256 recipientHatId = hats.createHat(
            topHatId,
            "Treasury Recipient",
            1,                           // Single recipient (treasury)
            address(0),
            address(0),
            true,
            "ipfs://..."
        );
        
        // 5. Deploy StakingEligibility module
        bytes memory initData = abi.encode(
            50_000 * 10**18,  // minStake: 50K SOF
            judgeHatId,       // judgeHat
            recipientHatId,   // recipientHat
            7 days            // cooldownPeriod
        );
        
        bytes memory otherImmutableArgs = abi.encodePacked(sofToken);
        
        address stakingEligibility = factory.createHatsModule(
            STAKING_ELIGIBILITY_IMPL,
            raffleCreatorHatId,
            otherImmutableArgs,
            initData,
            uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) // salt
        );
        
        // 6. Set eligibility module on the hat
        hats.changeHatEligibility(raffleCreatorHatId, stakingEligibility);
        
        // 7. Mint Judge hat to DAO Safe (or designated judges)
        // hats.mintHat(judgeHatId, daoSafeAddress);
        
        // 8. Mint Recipient hat to Treasury
        // hats.mintHat(recipientHatId, treasuryAddress);
        
        vm.stopBroadcast();
        
        console.log("Top Hat ID:", topHatId);
        console.log("Raffle Creator Hat ID:", raffleCreatorHatId);
        console.log("Judge Hat ID:", judgeHatId);
        console.log("Recipient Hat ID:", recipientHatId);
        console.log("Staking Eligibility:", stakingEligibility);
    }
}
```

---

## 6. Safe Multisig Integration

### Overview

The Top Hat (DAO root) should be worn by a Safe multisig. This ensures:
1. Decentralized governance over the hat tree
2. Multi-sig approval for sensitive operations (slashing, parameter changes)
3. Integration with existing DAO tooling

### Hats Signer Gate Integration

If we want hat wearers to also have Safe signing authority (optional, for advanced use cases):

```
┌─────────────────────────────────────────────────────────────┐
│                     Safe Multisig                            │
│  (DAO Treasury / Governance)                                 │
│                                                              │
│  Signers determined by:                                      │
│  - Wearers of "DAO Signer" Hat                              │
│  - Via Hats Signer Gate (Zodiac Module)                     │
└─────────────────────────────────────────────────────────────┘
```

### Option A: Simple Integration (Recommended for MVP)

```
1. Deploy Hats tree with Top Hat worn by existing Safe
2. Safe is admin of all child hats
3. Raffle Creator eligibility is purely stake-based
4. No Hats Signer Gate needed
```

### Option B: Full Hats Signer Gate (Advanced)

If we want hat-gated signing on a dedicated "Raffle Approvals" Safe:

```solidity
// Deploy via Hats App UI or script:
// 1. Deploy HatsSignerGate with:
//    - ownerHatId: topHatId
//    - signerHatId: raffleCreatorHatId
//    - minThreshold: 1
//    - targetThreshold: 3
//    - maxSigners: 10

// 2. This creates a Safe where:
//    - Anyone wearing Raffle Creator hat can sign
//    - 3-of-N signatures required for execution
```

### Recommended Approach

For SecondOrder.fun's use case, **Option A is sufficient**:
- Users stake → get hat → can create raffles directly
- No multisig approval needed for individual raffle creation
- Multisig (wearing Top Hat) retains admin powers for:
  - Slashing misbehaving creators
  - Adjusting minStake threshold
  - Emergency hat revocation

---

## 7. Implementation Phases

### Phase 1: Foundation (Week 1)

| Task | Description | Owner |
|------|-------------|-------|
| 1.1 | Deploy Hats tree on Base Sepolia | Contract Dev |
| 1.2 | Deploy StakingEligibility module | Contract Dev |
| 1.3 | Create IHats interface in SOF contracts | Contract Dev |
| 1.4 | Write unit tests for Hats integration | Contract Dev |

### Phase 2: Contract Integration (Week 2)

| Task | Description | Owner |
|------|-------------|-------|
| 2.1 | Modify Raffle.sol with dual auth (Hat OR Role) | Contract Dev |
| 2.2 | Add setRaffleCreatorHat admin function | Contract Dev |
| 2.3 | Add canCreateSeason view function | Contract Dev |
| 2.4 | Deploy updated Raffle to Sepolia | Contract Dev |
| 2.5 | Integration tests with live Hats contracts | QA |

### Phase 3: Frontend (Week 3)

| Task | Description | Owner |
|------|-------------|-------|
| 3.1 | Staking UI component (stake/unstake) | Frontend Dev |
| 3.2 | Hat status display (wearing/not wearing) | Frontend Dev |
| 3.3 | Create Raffle button gated by hat ownership | Frontend Dev |
| 3.4 | Cooldown timer display for unstaking | Frontend Dev |

### Phase 4: Admin & Monitoring (Week 4)

| Task | Description | Owner |
|------|-------------|-------|
| 4.1 | Judge dashboard for slashing | Frontend Dev |
| 4.2 | Subgraph/indexer for hat events | Backend Dev |
| 4.3 | Alerting for slashing events | DevOps |
| 4.4 | Documentation for users | Docs |

### Phase 5: Mainnet Launch (Week 5)

| Task | Description | Owner |
|------|-------------|-------|
| 5.1 | Audit review of changes | Security |
| 5.2 | Deploy Hats tree on Base Mainnet | Contract Dev |
| 5.3 | Deploy updated Raffle on Mainnet | Contract Dev |
| 5.4 | Transfer Top Hat to DAO Safe | Ops |
| 5.5 | Announce feature launch | Marketing |

---

## 8. Security Considerations

### Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Stake too low → spam | Medium | Set 50K SOF (~$X USD), adjustable by DAO |
| Malicious raffle creation | Medium | Judge can slash, community monitoring |
| Cooldown too short | Low | 7-day cooldown gives judges time to act |
| Top Hat compromise | Critical | Multisig with 3/5 threshold, hardware keys |
| Eligibility module bug | High | Use audited Hats modules, test extensively |

### Slashing Criteria

Define clear criteria for when a Raffle Creator can be slashed:

1. **Creating fraudulent raffles** (fake prizes, rigged outcomes)
2. **Abandoning active raffles** (not finalizing, not distributing prizes)
3. **Violating platform terms** (prohibited content, scams)
4. **Sybil attacks** (creating multiple stakes to manipulate)

### Emergency Procedures

1. **Pause Hat**: Toggle module can deactivate the Raffle Creator hat
2. **Slash Stake**: Judge hat wearers can slash misbehaving creators
3. **Revoke Role**: Admin can always revoke SEASON_CREATOR_ROLE as fallback
4. **Upgrade Eligibility**: Admin can change eligibility module if compromised

---

## 9. Gas Estimates & Costs

### One-Time Deployment Costs

| Operation | Estimated Gas | Cost @ 0.1 gwei |
|-----------|---------------|-----------------|
| Create Top Hat | ~150,000 | ~0.000015 ETH |
| Create Child Hats (x3) | ~300,000 | ~0.00003 ETH |
| Deploy StakingEligibility | ~500,000 | ~0.00005 ETH |
| Set Eligibility on Hat | ~50,000 | ~0.000005 ETH |
| **Total** | ~1,000,000 | ~0.0001 ETH |

### Per-User Costs

| Operation | Estimated Gas | Cost @ 0.1 gwei |
|-----------|---------------|-----------------|
| Approve SOF | ~50,000 | ~0.000005 ETH |
| Stake (first time) | ~150,000 | ~0.000015 ETH |
| Begin Unstake | ~80,000 | ~0.000008 ETH |
| Complete Unstake | ~100,000 | ~0.00001 ETH |
| Create Raffle | ~800,000 | ~0.00008 ETH |

---

## 10. Testing Strategy

### Unit Tests

```solidity
// test/HatsIntegration.t.sol

contract HatsIntegrationTest is Test {
    function test_StakeAndWearHat() public {
        // Stake 50K SOF
        // Assert user is now wearer of Raffle Creator hat
    }
    
    function test_CreateSeasonWithHat() public {
        // Stake and claim hat
        // Create season successfully
    }
    
    function test_CreateSeasonWithoutHat_Fails() public {
        // Don't stake
        // Attempt create season → reverts
    }
    
    function test_UnstakeBelowMinimum_LosesHat() public {
        // Stake 50K, verify hat
        // Begin unstake 40K
        // Verify hat is revoked (balance below minimum)
    }
    
    function test_SlashRemovesHatAndStake() public {
        // Stake and claim hat
        // Judge slashes
        // Verify: stake gone, hat revoked, in bad standing
    }
    
    function test_ForgiveAllowsRestaking() public {
        // Slash user
        // Attempt stake → fails (bad standing)
        // Forgive user
        // Stake → succeeds
    }
}
```

### Integration Tests

```solidity
contract HatsIntegrationE2E is Test {
    function test_FullFlow_StakeCreateRaffleUnstake() public {
        // 1. User approves and stakes 50K SOF
        // 2. User creates raffle
        // 3. Raffle runs, finalizes, distributes prizes
        // 4. User begins unstake
        // 5. Wait cooldown
        // 6. User completes unstake
        // 7. Verify user no longer has hat, cannot create raffle
    }
}
```

### Testnet Validation

1. Deploy to Base Sepolia
2. Create test raffle with staked account
3. Test slashing flow
4. Test unstaking cooldown
5. Verify UI displays correct state

---

## 11. Deployment Checklist

### Pre-Deployment

- [ ] All unit tests passing
- [ ] Integration tests passing on fork
- [ ] Security review complete
- [ ] Gas estimates validated
- [ ] Frontend integration tested on Sepolia

### Sepolia Deployment

- [ ] Deploy Hats tree
- [ ] Deploy StakingEligibility module
- [ ] Configure eligibility on Raffle Creator hat
- [ ] Deploy updated Raffle contract
- [ ] Set raffleCreatorHatId in Raffle
- [ ] Mint Judge hat to test accounts
- [ ] Mint Recipient hat to test treasury
- [ ] Test full flow end-to-end

### Mainnet Deployment

- [ ] Verify all Sepolia tests pass
- [ ] Deploy Hats tree (use same structure)
- [ ] Deploy StakingEligibility with mainnet SOF
- [ ] Deploy updated Raffle contract
- [ ] Configure hat ID
- [ ] Transfer Top Hat to DAO Safe multisig
- [ ] Mint Judge hat to designated governance
- [ ] Mint Recipient hat to Treasury
- [ ] Verify all permissions correct
- [ ] Monitor for 48 hours

### Post-Deployment

- [ ] Update frontend to point to new contracts
- [ ] Update documentation
- [ ] Announce feature launch
- [ ] Monitor Subgraph for events
- [ ] Set up alerts for slashing

---

## Appendix A: Contract Addresses (VERIFIED)

All Hats Protocol contracts are deployed at identical addresses on both chains.

### Base Sepolia & Base Mainnet (Same Addresses)

| Contract | Address | Verified |
|----------|---------|----------|
| Hats.sol | `0x3bc1A0Ad72417f2d411118085256fC53CBdDd137` | ✅ Both chains |
| HatsModuleFactory | `0x0a3f85fa597B6a967271286aA0724811acDF5CD9` | ✅ Both chains |
| StakingEligibility v0.3.0 (impl) | `0x9E01030aF633Be5a439DF122F2eEf750b44B8aC7` | ✅ Both chains |

### SecondOrder.fun Contracts (To Be Updated)

| Contract | Sepolia | Mainnet | Notes |
|----------|---------|---------|-------|
| $SOF Token | `0x...` | TBD | Existing deployment |
| Raffle (current) | `0x...` | TBD | Existing deployment |
| Raffle (upgraded) | TBD | TBD | With Hats integration |
| StakingEligibility (instance) | TBD | TBD | Deployed via factory |

### Hat IDs (Generated After Deployment)

| Hat | ID | Description |
|-----|-----|-------------|
| Top Hat | TBD | DAO governance root (worn by Safe) |
| Raffle Creator | TBD | Stake-gated creator role (50K SOF) |
| Judge | TBD | Can slash bad actors |
| Recipient | TBD | Receives slashed stakes (Treasury) |

---

## Appendix B: Step-by-Step Implementation Guide

### Prerequisites

```bash
# Install Hats SDK dependencies
forge install Hats-Protocol/hats-protocol
forge install Hats-Protocol/hats-module

# Add remappings
echo "hats-protocol/=lib/hats-protocol/src/" >> remappings.txt
echo "hats-module/=lib/hats-module/src/" >> remappings.txt
```

### Step 1: Add IHats Interface

Create `contracts/src/lib/IHats.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IHats {
    function mintTopHat(
        address _target,
        string calldata _details,
        string calldata _imageURI
    ) external returns (uint256 topHatId);
    
    function createHat(
        uint256 _admin,
        string calldata _details,
        uint32 _maxSupply,
        address _eligibility,
        address _toggle,
        bool _mutable,
        string calldata _imageURI
    ) external returns (uint256 newHatId);
    
    function mintHat(uint256 _hatId, address _wearer) external returns (bool success);
    
    function changeHatEligibility(uint256 _hatId, address _newEligibility) external;
    
    function isWearerOfHat(address _user, uint256 _hatId) external view returns (bool);
    
    function balanceOf(address _wearer, uint256 _hatId) external view returns (uint256);
    
    function isInGoodStanding(address _wearer, uint256 _hatId) external view returns (bool);
    
    function isActive(uint256 _hatId) external view returns (bool);
    
    function transferHat(uint256 _hatId, address _from, address _to) external;
}
```

### Step 2: Add IHatsModuleFactory Interface

Create `contracts/src/lib/IHatsModuleFactory.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IHatsModuleFactory {
    function createHatsModule(
        address _implementation,
        uint256 _hatId,
        bytes calldata _otherImmutableArgs,
        bytes calldata _initData,
        uint256 _saltNonce
    ) external returns (address instance);
}
```

### Step 3: Modify Raffle.sol

Add to constructor and state:

```solidity
// Add import
import "../lib/IHats.sol";

// Add state variable
IHats public immutable HATS;
uint256 public raffleCreatorHatId;

// Add event
event RaffleCreatorHatUpdated(uint256 indexed hatId);

// Modify constructor
constructor(
    address _sofToken,
    address _vrfCoordinator,
    uint256 _vrfSubscriptionId,
    bytes32 _vrfKeyHash,
    address _hatsProtocol  // NEW PARAMETER
) VRFConsumerBaseV2Plus(_vrfCoordinator) {
    // ... existing logic ...
    HATS = IHats(_hatsProtocol);
}

// Add new modifier
modifier canCreateSeason() {
    bool hasHat = raffleCreatorHatId != 0 && 
                  HATS.isWearerOfHat(msg.sender, raffleCreatorHatId) &&
                  HATS.isInGoodStanding(msg.sender, raffleCreatorHatId);
    bool hasRole = hasRole(SEASON_CREATOR_ROLE, msg.sender);
    
    if (!hasHat && !hasRole) revert UnauthorizedCaller();
    _;
}

// Add admin function
function setRaffleCreatorHat(uint256 _hatId) external onlyRole(DEFAULT_ADMIN_ROLE) {
    raffleCreatorHatId = _hatId;
    emit RaffleCreatorHatUpdated(_hatId);
}

// Add view function
function canAccountCreateSeason(address account) external view returns (bool) {
    bool hasHat = raffleCreatorHatId != 0 && 
                  HATS.isWearerOfHat(account, raffleCreatorHatId) &&
                  HATS.isInGoodStanding(account, raffleCreatorHatId);
    bool hasRole = hasRole(SEASON_CREATOR_ROLE, account);
    return hasHat || hasRole;
}

// Modify createSeason to use new modifier
function createSeason(
    RaffleTypes.SeasonConfig memory config,
    RaffleTypes.BondStep[] memory bondSteps,
    uint16 buyFeeBps,
    uint16 sellFeeBps
) external canCreateSeason nonReentrant returns (uint256 seasonId) {
    // ... existing logic unchanged ...
}
```

### Step 4: Create Deployment Script

Create `contracts/script/DeployHatsTree.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/lib/IHats.sol";
import "../src/lib/IHatsModuleFactory.sol";

contract DeployHatsTree is Script {
    // Verified addresses (same on Sepolia & Mainnet)
    address constant HATS = 0x3bc1A0Ad72417f2d411118085256fC53CBdDd137;
    address constant HATS_MODULE_FACTORY = 0x0a3f85fa597B6a967271286aA0724811acDF5CD9;
    address constant STAKING_ELIGIBILITY_IMPL = 0x9E01030aF633Be5a439DF122F2eEf750b44B8aC7;
    
    // Configuration
    uint256 constant MIN_STAKE = 50_000 * 10**18; // 50K SOF
    uint256 constant COOLDOWN_PERIOD = 7 days;
    uint32 constant MAX_CREATORS = 100;
    uint32 constant MAX_JUDGES = 5;
    
    function run() external {
        address sofToken = vm.envAddress("SOF_TOKEN");
        address treasuryAddress = vm.envAddress("TREASURY_ADDRESS");
        address daoSafe = vm.envAddress("DAO_SAFE"); // Optional: transfer Top Hat to Safe
        
        uint256 deployerKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerKey);
        
        vm.startBroadcast(deployerKey);
        
        IHats hats = IHats(HATS);
        IHatsModuleFactory factory = IHatsModuleFactory(HATS_MODULE_FACTORY);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 1: Create Top Hat (DAO Root)
        // ═══════════════════════════════════════════════════════════════════
        uint256 topHatId = hats.mintTopHat(
            deployer, // Mint to deployer first, transfer to Safe later
            "SecondOrder.fun DAO",
            "ipfs://QmSecondOrderDAOImage" // Replace with actual IPFS hash
        );
        console.log("Top Hat ID:", topHatId);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 2: Create Raffle Creator Hat (stake-gated)
        // ═══════════════════════════════════════════════════════════════════
        uint256 raffleCreatorHatId = hats.createHat(
            topHatId,                       // Admin
            "Raffle Creator",               // Details
            MAX_CREATORS,                   // Max supply
            address(0),                     // Eligibility (set after module deploy)
            address(0),                     // Toggle (active by default)
            true,                           // Mutable
            "ipfs://QmRaffleCreatorImage"   // Replace with actual IPFS hash
        );
        console.log("Raffle Creator Hat ID:", raffleCreatorHatId);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 3: Create Judge Hat (can slash)
        // ═══════════════════════════════════════════════════════════════════
        uint256 judgeHatId = hats.createHat(
            topHatId,
            "Raffle Judge",
            MAX_JUDGES,
            address(0),
            address(0),
            true,
            "ipfs://QmJudgeImage"
        );
        console.log("Judge Hat ID:", judgeHatId);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 4: Create Recipient Hat (receives slashed stakes)
        // ═══════════════════════════════════════════════════════════════════
        uint256 recipientHatId = hats.createHat(
            topHatId,
            "Treasury Recipient",
            1,                              // Only 1 recipient (treasury)
            address(0),
            address(0),
            true,
            "ipfs://QmTreasuryImage"
        );
        console.log("Recipient Hat ID:", recipientHatId);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 5: Deploy StakingEligibility Module
        // ═══════════════════════════════════════════════════════════════════
        bytes memory otherImmutableArgs = abi.encodePacked(sofToken);
        bytes memory initData = abi.encode(
            MIN_STAKE,          // minStake: 50K SOF
            judgeHatId,         // judgeHat
            recipientHatId,     // recipientHat
            COOLDOWN_PERIOD     // cooldownPeriod: 7 days
        );
        
        address stakingEligibility = factory.createHatsModule(
            STAKING_ELIGIBILITY_IMPL,
            raffleCreatorHatId,
            otherImmutableArgs,
            initData,
            uint256(keccak256(abi.encodePacked(block.timestamp, deployer, "SOF_STAKING")))
        );
        console.log("StakingEligibility Instance:", stakingEligibility);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 6: Set Eligibility Module on Raffle Creator Hat
        // ═══════════════════════════════════════════════════════════════════
        hats.changeHatEligibility(raffleCreatorHatId, stakingEligibility);
        console.log("Eligibility set on Raffle Creator Hat");
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 7: Mint Recipient Hat to Treasury
        // ═══════════════════════════════════════════════════════════════════
        hats.mintHat(recipientHatId, treasuryAddress);
        console.log("Recipient Hat minted to Treasury:", treasuryAddress);
        
        // ═══════════════════════════════════════════════════════════════════
        // STEP 8: (Optional) Transfer Top Hat to DAO Safe
        // ═══════════════════════════════════════════════════════════════════
        if (daoSafe != address(0)) {
            hats.transferHat(topHatId, deployer, daoSafe);
            console.log("Top Hat transferred to DAO Safe:", daoSafe);
        }
        
        vm.stopBroadcast();
        
        // ═══════════════════════════════════════════════════════════════════
        // OUTPUT SUMMARY
        // ═══════════════════════════════════════════════════════════════════
        console.log("\n=== DEPLOYMENT SUMMARY ===");
        console.log("Top Hat ID:           ", topHatId);
        console.log("Raffle Creator Hat ID:", raffleCreatorHatId);
        console.log("Judge Hat ID:         ", judgeHatId);
        console.log("Recipient Hat ID:     ", recipientHatId);
        console.log("StakingEligibility:   ", stakingEligibility);
        console.log("Min Stake:             50,000 SOF");
        console.log("Cooldown:              7 days");
        console.log("===========================\n");
        
        console.log("NEXT STEPS:");
        console.log("1. Deploy upgraded Raffle contract with HATS parameter");
        console.log("2. Call raffle.setRaffleCreatorHat(", raffleCreatorHatId, ")");
        console.log("3. Mint Judge Hats to designated accounts");
        console.log("4. Test staking + raffle creation flow");
    }
}
```

### Step 5: Run Deployment

```bash
# Set environment variables
export PRIVATE_KEY=0x...
export SOF_TOKEN=0x...  # Your SOF token address
export TREASURY_ADDRESS=0x...  # Treasury for slashed stakes
export DAO_SAFE=0x...  # (Optional) Safe multisig for Top Hat

# Deploy to Base Sepolia
forge script script/DeployHatsTree.s.sol:DeployHatsTree \
  --rpc-url https://sepolia.base.org \
  --broadcast \
  --verify

# Record the output Hat IDs and StakingEligibility address!
```

### Step 6: Upgrade Raffle Contract

```bash
# Deploy new Raffle with HATS parameter
forge script script/DeployRaffleV2.s.sol:DeployRaffleV2 \
  --rpc-url https://sepolia.base.org \
  --broadcast \
  --verify

# Configure the Hat ID
cast send <NEW_RAFFLE_ADDRESS> \
  "setRaffleCreatorHat(uint256)" \
  <RAFFLE_CREATOR_HAT_ID> \
  --rpc-url https://sepolia.base.org \
  --private-key $PRIVATE_KEY
```

### Step 7: Test End-to-End Flow

```bash
# 1. Approve SOF for StakingEligibility
cast send $SOF_TOKEN \
  "approve(address,uint256)" \
  <STAKING_ELIGIBILITY_ADDRESS> \
  50000000000000000000000 \
  --rpc-url https://sepolia.base.org \
  --private-key $PRIVATE_KEY

# 2. Stake 50K SOF
cast send <STAKING_ELIGIBILITY_ADDRESS> \
  "stake(uint256)" \
  50000000000000000000000 \
  --rpc-url https://sepolia.base.org \
  --private-key $PRIVATE_KEY

# 3. Verify you now wear the hat
cast call $HATS \
  "isWearerOfHat(address,uint256)" \
  $YOUR_ADDRESS \
  <RAFFLE_CREATOR_HAT_ID> \
  --rpc-url https://sepolia.base.org
# Should return: true

# 4. Create a raffle!
cast send <RAFFLE_ADDRESS> \
  "createSeason((string,uint256,uint256,uint8,uint16,address,address),(uint256,uint256,uint256)[],uint16,uint16)" \
  "(Test Raffle,1707300000,1707900000,1,6500,0x...,0x...)" \
  "[(0,1000000000000000000000,1000000000000000000)]" \
  100 \
  100 \
  --rpc-url https://sepolia.base.org \
  --private-key $PRIVATE_KEY
```

---

## Appendix C: Frontend Integration

### Staking UI Component

```typescript
// src/hooks/useHatsStaking.ts
import { useContractRead, useContractWrite, usePrepareContractWrite } from 'wagmi';

const STAKING_ELIGIBILITY_ABI = [
  'function stake(uint256 amount) external',
  'function beginUnstake(uint256 amount) external',
  'function completeUnstake(address staker) external',
  'function stakes(address staker) external view returns (uint256)',
  'function cooldownPeriod() external view returns (uint256)',
  'function unstakingData(address) external view returns (uint256 amount, uint256 endsAt)',
];

export function useHatsStaking(stakingEligibilityAddress: string) {
  // Read current stake
  const { data: currentStake } = useContractRead({
    address: stakingEligibilityAddress,
    abi: STAKING_ELIGIBILITY_ABI,
    functionName: 'stakes',
    args: [address],
  });

  // Stake tokens
  const { config: stakeConfig } = usePrepareContractWrite({
    address: stakingEligibilityAddress,
    abi: STAKING_ELIGIBILITY_ABI,
    functionName: 'stake',
    args: [parseEther('50000')],
  });
  const { write: stake } = useContractWrite(stakeConfig);

  // Begin unstake
  const { config: unstakeConfig } = usePrepareContractWrite({
    address: stakingEligibilityAddress,
    abi: STAKING_ELIGIBILITY_ABI,
    functionName: 'beginUnstake',
    args: [parseEther('50000')],
  });
  const { write: beginUnstake } = useContractWrite(unstakeConfig);

  return { currentStake, stake, beginUnstake };
}
```

### Check Hat Ownership

```typescript
// src/hooks/useCanCreateRaffle.ts
import { useContractRead } from 'wagmi';

const HATS_ABI = [
  'function isWearerOfHat(address,uint256) view returns (bool)',
  'function isInGoodStanding(address,uint256) view returns (bool)',
];

export function useCanCreateRaffle(address: string) {
  const { data: isWearer } = useContractRead({
    address: HATS_ADDRESS,
    abi: HATS_ABI,
    functionName: 'isWearerOfHat',
    args: [address, RAFFLE_CREATOR_HAT_ID],
  });

  const { data: inGoodStanding } = useContractRead({
    address: HATS_ADDRESS,
    abi: HATS_ABI,
    functionName: 'isInGoodStanding',
    args: [address, RAFFLE_CREATOR_HAT_ID],
  });

  return isWearer && inGoodStanding;
}
```

---

## Appendix D: Fallback Strategy

If Hats Protocol integration encounters issues on Base Sepolia during testing:

1. **Defer to Mainnet**: Complete development/testing with mocked Hats calls, deploy real integration only on Base Mainnet at launch
2. **Feature Flag**: Add `HATS_ENABLED` env var to toggle between Hats check and legacy AccessControl
3. **Backwards Compatible**: The `canCreateSeason` modifier already supports BOTH hat ownership AND role-based access — existing `SEASON_CREATOR_ROLE` grants continue working

```solidity
// Fallback in Raffle.sol if Hats is disabled
modifier canCreateSeason() {
    if (address(HATS) == address(0) || raffleCreatorHatId == 0) {
        // Hats not configured — fall back to role-only
        if (!hasRole(SEASON_CREATOR_ROLE, msg.sender)) revert UnauthorizedCaller();
    } else {
        // Hats configured — check hat OR role
        bool hasHat = HATS.isWearerOfHat(msg.sender, raffleCreatorHatId) &&
                      HATS.isInGoodStanding(msg.sender, raffleCreatorHatId);
        bool hasRole = hasRole(SEASON_CREATOR_ROLE, msg.sender);
        if (!hasHat && !hasRole) revert UnauthorizedCaller();
    }
    _;
}
```

---

## References

- [Hats Protocol Documentation](https://docs.hatsprotocol.xyz/)
- [Staking Eligibility Module](https://github.com/Hats-Protocol/staking-eligibility)
- [Hats Module Factory](https://github.com/Hats-Protocol/hats-module)
- [Hats Modules Registry](https://github.com/Hats-Protocol/modules-registry)
- [Safe Multisig](https://safe.global/)
- [SecondOrder.fun Contracts](../contracts/src/core/)

---

*Last Updated: February 6, 2026*
