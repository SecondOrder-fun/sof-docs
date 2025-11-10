# Market Type Refactor Proposal

## Current Problem

Adding new market types requires:
1. ✅ Updating `InfoFiMarketFactory.sol` with new constants
2. ✅ Redeploying the factory contract
3. ✅ Updating backend hash mapping
4. ✅ Restarting backend

This is expensive and disruptive for production systems.

## Proposed Solutions

### Option 1: Registry Pattern (Recommended) ⭐

Create a separate `MarketTypeRegistry` contract that can be updated independently.

#### Architecture

```
┌─────────────────────────┐
│ MarketTypeRegistry      │
│ (Upgradeable/Mutable)   │
│                         │
│ - registerMarketType()  │
│ - getMarketType()       │
│ - isValidMarketType()   │
└─────────────────────────┘
            ▲
            │ reads from
            │
┌─────────────────────────┐
│ InfoFiMarketFactory     │
│ (Immutable Logic)       │
│                         │
│ - createMarket()        │
│ - onPositionUpdate()    │
└─────────────────────────┘
```

#### Implementation

```solidity
// contracts/src/infofi/MarketTypeRegistry.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "openzeppelin-contracts/contracts/access/AccessControl.sol";

/**
 * @title MarketTypeRegistry
 * @notice Manages market type definitions without requiring factory redeployment
 * @dev Allows adding new market types dynamically
 */
contract MarketTypeRegistry is AccessControl {
    bytes32 public constant ADMIN_ROLE = DEFAULT_ADMIN_ROLE;
    bytes32 public constant REGISTRY_MANAGER_ROLE = keccak256("REGISTRY_MANAGER_ROLE");

    struct MarketTypeInfo {
        bytes32 typeHash;        // keccak256 of the type name
        string typeName;         // Human-readable name
        bool isActive;           // Whether this type is currently active
        uint256 createdAt;       // When this type was registered
        string description;      // Optional description
    }

    // typeHash => MarketTypeInfo
    mapping(bytes32 => MarketTypeInfo) public marketTypes;
    
    // Array of all registered type hashes (for enumeration)
    bytes32[] public registeredTypes;
    
    // Reverse lookup: typeName => typeHash
    mapping(string => bytes32) public typeNameToHash;

    event MarketTypeRegistered(
        bytes32 indexed typeHash,
        string typeName,
        string description
    );

    event MarketTypeDeactivated(
        bytes32 indexed typeHash,
        string typeName
    );

    event MarketTypeReactivated(
        bytes32 indexed typeHash,
        string typeName
    );

    constructor(address admin) {
        _grantRole(ADMIN_ROLE, admin);
        _grantRole(REGISTRY_MANAGER_ROLE, admin);

        // Register default market types
        _registerMarketType("WINNER_PREDICTION", "Predict the raffle winner");
    }

    /**
     * @notice Register a new market type
     * @param typeName Human-readable name (e.g., "WINNER_PREDICTION")
     * @param description Optional description of the market type
     */
    function registerMarketType(
        string calldata typeName,
        string calldata description
    ) external onlyRole(REGISTRY_MANAGER_ROLE) {
        _registerMarketType(typeName, description);
    }

    /**
     * @notice Internal function to register a market type
     */
    function _registerMarketType(
        string memory typeName,
        string memory description
    ) internal {
        bytes32 typeHash = keccak256(abi.encodePacked(typeName));
        
        require(marketTypes[typeHash].typeHash == bytes32(0), "Market type already exists");
        require(bytes(typeName).length > 0, "Type name cannot be empty");

        marketTypes[typeHash] = MarketTypeInfo({
            typeHash: typeHash,
            typeName: typeName,
            isActive: true,
            createdAt: block.timestamp,
            description: description
        });

        registeredTypes.push(typeHash);
        typeNameToHash[typeName] = typeHash;

        emit MarketTypeRegistered(typeHash, typeName, description);
    }

    /**
     * @notice Deactivate a market type (doesn't delete, just marks inactive)
     */
    function deactivateMarketType(bytes32 typeHash) 
        external 
        onlyRole(REGISTRY_MANAGER_ROLE) 
    {
        require(marketTypes[typeHash].isActive, "Market type not active");
        marketTypes[typeHash].isActive = false;
        emit MarketTypeDeactivated(typeHash, marketTypes[typeHash].typeName);
    }

    /**
     * @notice Reactivate a previously deactivated market type
     */
    function reactivateMarketType(bytes32 typeHash) 
        external 
        onlyRole(REGISTRY_MANAGER_ROLE) 
    {
        require(!marketTypes[typeHash].isActive, "Market type already active");
        require(marketTypes[typeHash].typeHash != bytes32(0), "Market type not registered");
        marketTypes[typeHash].isActive = true;
        emit MarketTypeReactivated(typeHash, marketTypes[typeHash].typeName);
    }

    /**
     * @notice Check if a market type is valid and active
     */
    function isValidMarketType(bytes32 typeHash) external view returns (bool) {
        return marketTypes[typeHash].isActive;
    }

    /**
     * @notice Get market type info by hash
     */
    function getMarketType(bytes32 typeHash) 
        external 
        view 
        returns (MarketTypeInfo memory) 
    {
        return marketTypes[typeHash];
    }

    /**
     * @notice Get market type hash by name
     */
    function getMarketTypeHash(string calldata typeName) 
        external 
        view 
        returns (bytes32) 
    {
        return typeNameToHash[typeName];
    }

    /**
     * @notice Get all registered market types
     */
    function getAllMarketTypes() 
        external 
        view 
        returns (MarketTypeInfo[] memory) 
    {
        MarketTypeInfo[] memory types = new MarketTypeInfo[](registeredTypes.length);
        for (uint256 i = 0; i < registeredTypes.length; i++) {
            types[i] = marketTypes[registeredTypes[i]];
        }
        return types;
    }

    /**
     * @notice Get count of registered market types
     */
    function getMarketTypeCount() external view returns (uint256) {
        return registeredTypes.length;
    }
}
```

#### Updated InfoFiMarketFactory.sol

```solidity
// contracts/src/infofi/InfoFiMarketFactory.sol

contract InfoFiMarketFactory is AccessControl, ReentrancyGuard {
    // ... existing code ...

    MarketTypeRegistry public immutable marketTypeRegistry;

    constructor(
        // ... existing parameters ...
        address _marketTypeRegistry
    ) {
        // ... existing initialization ...
        marketTypeRegistry = MarketTypeRegistry(_marketTypeRegistry);
    }

    function _createMarketInternal(
        uint256 seasonId,
        address player,
        bytes32 marketType  // Now passed as parameter, not hardcoded
    ) internal {
        // Validate market type
        require(
            marketTypeRegistry.isValidMarketType(marketType),
            "Invalid market type"
        );

        // ... rest of existing logic ...
        
        emit MarketCreated(seasonId, player, marketType, conditionId, fpmm);
    }

    function onPositionUpdate(
        uint256 seasonId,
        address player,
        uint256 oldTickets,
        uint256 newTickets,
        uint256 totalTickets
    ) external onlyRole(RAFFLE_ROLE) nonReentrant {
        // ... existing threshold check ...

        // Determine market type based on criteria
        bytes32 marketType = _determineMarketType(seasonId, player, newTickets);

        try this._createMarketInternal(seasonId, player, marketType) {
            // Success
        } catch Error(string memory reason) {
            emit MarketCreationFailed(seasonId, player, marketType, reason);
        }
    }

    /**
     * @notice Determine which market type to create based on criteria
     * @dev Can be overridden in future versions for more complex logic
     */
    function _determineMarketType(
        uint256 seasonId,
        address player,
        uint256 newTickets
    ) internal view returns (bytes32) {
        // For now, always return WINNER_PREDICTION
        // In future, could have logic like:
        // - if (newTickets > 10000) return POSITION_SIZE
        // - if (player has history) return BEHAVIORAL
        return marketTypeRegistry.getMarketTypeHash("WINNER_PREDICTION");
    }
}
```

### Benefits of Registry Pattern

✅ **No Factory Redeployment**: Add new market types without touching factory
✅ **Dynamic Updates**: Registry can be updated on-chain
✅ **Enumeration**: Can query all available market types
✅ **Validation**: Factory validates market types against registry
✅ **Metadata**: Store descriptions and other info about market types
✅ **Deactivation**: Can disable market types without deleting them
✅ **Upgradeable**: Registry can be made upgradeable (proxy pattern)

### Drawbacks

❌ **Extra Gas Cost**: ~2,000-5,000 gas per market creation for registry lookup
❌ **Complexity**: More contracts to manage and deploy
❌ **Dependency**: Factory depends on registry being available

---

### Option 2: Pass Market Type as Parameter

Simplest approach - let the caller specify the market type:

```solidity
function onPositionUpdate(
    uint256 seasonId,
    address player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets,
    bytes32 marketType  // NEW: Caller specifies market type
) external onlyRole(RAFFLE_ROLE) nonReentrant {
    // Validate market type exists (could use registry or simple mapping)
    require(isValidMarketType(marketType), "Invalid market type");
    
    // ... rest of logic ...
}
```

#### Benefits
✅ **Maximum Flexibility**: Caller controls market type
✅ **No Registry Needed**: Simpler architecture
✅ **Lower Gas**: No external calls

#### Drawbacks
❌ **Caller Responsibility**: Raffle contract must know about market types
❌ **No Central Management**: Market types scattered across callers
❌ **Validation Needed**: Still need some way to validate market types

---

### Option 3: Event-Based Discovery (Backend-Only)

Keep contract simple, make backend smart:

```solidity
// Contract emits generic event
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 marketType,  // Any bytes32 value
    bytes32 conditionId,
    address fpmmAddress
);

// Backend listens and handles unknown types
const marketTypeStr = MARKET_TYPE_HASHES[marketType] || `UNKNOWN_${marketType}`;
```

#### Benefits
✅ **No Contract Changes**: Works with current implementation
✅ **Backend Flexibility**: Backend can handle new types gracefully
✅ **Lowest Gas**: No validation overhead

#### Drawbacks
❌ **No On-Chain Validation**: Can emit invalid market types
❌ **Backend Dependency**: Backend must stay in sync
❌ **No Enumeration**: Can't query available types on-chain

---

## Recommendation: Registry Pattern (Option 1)

### Why?

1. **Production-Ready**: No factory redeployment for new market types
2. **On-Chain Validation**: Ensures only valid market types are created
3. **Enumeration**: Frontend can query available market types
4. **Metadata**: Can store descriptions, thresholds, etc.
5. **Upgradeable**: Registry can be made upgradeable if needed

### Implementation Plan

1. **Phase 1**: Create `MarketTypeRegistry.sol`
2. **Phase 2**: Update `InfoFiMarketFactory.sol` to use registry
3. **Phase 3**: Update backend to query registry for available types
4. **Phase 4**: Create admin UI for managing market types

### Gas Cost Analysis

| Operation | Current | With Registry | Difference |
|-----------|---------|---------------|------------|
| Market Creation | ~350,000 | ~355,000 | +5,000 (+1.4%) |
| Registry Lookup | 0 | ~2,000 | +2,000 |
| Validation | 0 | ~3,000 | +3,000 |

**Verdict**: ~5,000 gas overhead is acceptable for the flexibility gained.

---

## Alternative: Hybrid Approach

For maximum flexibility with minimal gas:

1. **Keep common types hardcoded** (WINNER_PREDICTION) - 0 gas overhead
2. **Use registry for new types** - Only pay gas when using new types
3. **Backend queries both** - Checks hardcoded constants first, then registry

```solidity
function isValidMarketType(bytes32 marketType) public view returns (bool) {
    // Check hardcoded types first (no gas cost)
    if (marketType == WINNER_PREDICTION) return true;
    
    // Fall back to registry for new types
    return marketTypeRegistry.isValidMarketType(marketType);
}
```

This gives you the best of both worlds! 🎯
