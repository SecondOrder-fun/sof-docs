# Market Type Implementation Guide

## Overview

Market types in SecondOrder.fun are identified using **keccak256 hashes** for gas efficiency and type safety. This document explains the implementation pattern and how to add new market types.

## Current Implementation

### Contract Side (Solidity)

Market types are defined as `bytes32` constants using `keccak256`:

```solidity
// contracts/src/infofi/InfoFiMarketFactory.sol
bytes32 public constant WINNER_PREDICTION = keccak256("WINNER_PREDICTION");
// Hash: 0x9af7ac054212f2f6f51aadd6392aae69c37a65182710ccc31fc2ce8679842eab
```

Events emit the hash, not the string:

```solidity
event MarketCreated(
    uint256 indexed seasonId,
    address indexed player,
    bytes32 marketType,  // Emits the HASH (0x9af7ac...)
    bytes32 conditionId,
    address fpmmAddress
);

emit MarketCreated(seasonId, player, WINNER_PREDICTION, conditionId, fpmm);
```

### Backend Side (JavaScript)

The backend maintains a mapping to decode hashes back to strings:

```javascript
// backend/src/listeners/marketCreatedListener.js
const MARKET_TYPE_HASHES = {
  '0x9af7ac054212f2f6f51aadd6392aae69c37a65182710ccc31fc2ce8679842eab': 'WINNER_PREDICTION',
  // Add more market types here as they're added to the contract
};

// Usage in event listener
const marketTypeStr = MARKET_TYPE_HASHES[marketType] || 'UNKNOWN';
```

## Why Use Hashes Instead of Strings?

### Advantages of Hash-Based Approach

1. **Gas Efficiency**: Saves ~200-400 gas per event emission
   - Hash is pre-computed at compile time (zero runtime cost)
   - String conversion requires runtime processing

2. **Type Safety**: Cannot emit malformed or invalid strings
   - Compiler enforces valid constant references
   - No risk of typos in event emissions

3. **Industry Standard**: Used by major protocols
   - OpenZeppelin's AccessControl uses role hashes
   - Uniswap, Aave, Compound use similar patterns

4. **Consistent with Internal Logic**: Contract already uses hashes for comparisons
   - No conversion overhead between event emission and internal checks

### Disadvantages

1. **Requires Mapping**: Backend needs hash → string mapping
2. **Not Human-Readable**: Raw event logs show hash, not string
3. **Synchronization**: Backend must stay in sync with contract constants

## Adding New Market Types

### Step 1: Add Contract Constant

```solidity
// contracts/src/infofi/InfoFiMarketFactory.sol

// Existing
bytes32 public constant WINNER_PREDICTION = keccak256("WINNER_PREDICTION");

// NEW: Add your new market type
bytes32 public constant POSITION_SIZE = keccak256("POSITION_SIZE");
bytes32 public constant BEHAVIORAL = keccak256("BEHAVIORAL");
```

### Step 2: Calculate the Hash

Use `cast` to calculate the keccak256 hash:

```bash
cast keccak "POSITION_SIZE"
# Output: 0x1234567890abcdef... (example)

cast keccak "BEHAVIORAL"
# Output: 0xabcdef1234567890... (example)
```

### Step 3: Update Backend Mapping

```javascript
// backend/src/listeners/marketCreatedListener.js
const MARKET_TYPE_HASHES = {
  '0x9af7ac054212f2f6f51aadd6392aae69c37a65182710ccc31fc2ce8679842eab': 'WINNER_PREDICTION',
  '0x1234567890abcdef...': 'POSITION_SIZE',  // NEW
  '0xabcdef1234567890...': 'BEHAVIORAL',     // NEW
};
```

### Step 4: Update Contract Logic

Modify `onPositionUpdate()` or other functions to emit the new market type:

```solidity
function onPositionUpdate(
    uint256 seasonId,
    address player,
    uint256 oldTickets,
    uint256 newTickets,
    uint256 totalTickets
) external onlyRole(RAFFLE_ROLE) nonReentrant {
    // ... existing logic ...
    
    // Choose market type based on criteria
    bytes32 marketType;
    if (/* some condition */) {
        marketType = POSITION_SIZE;
    } else if (/* another condition */) {
        marketType = BEHAVIORAL;
    } else {
        marketType = WINNER_PREDICTION;
    }
    
    emit MarketCreated(seasonId, player, marketType, conditionId, fpmm);
}
```

### Step 5: Deploy and Test

1. **Redeploy contract** with new constants
2. **Copy updated ABIs**: `npm run copy-abis`
3. **Restart backend**: `npm run dev:backend`
4. **Test**: Trigger market creation and verify correct decoding

## Database Schema

Market types are stored as strings in the database:

```sql
-- infofi_markets table
market_type VARCHAR(50) NOT NULL  -- Stores "WINNER_PREDICTION", not the hash
```

The backend converts hash → string before database insertion.

## Testing New Market Types

### Contract Tests

```solidity
// contracts/test/InfoFiMarketFactory.t.sol
function testNewMarketType() public {
    bytes32 expectedHash = keccak256("POSITION_SIZE");
    assertEq(factory.POSITION_SIZE(), expectedHash);
}
```

### Backend Tests

```javascript
// tests/backend/marketCreatedListener.test.js
describe('Market Type Decoding', () => {
  it('should decode POSITION_SIZE hash', () => {
    const hash = '0x1234567890abcdef...';
    const decoded = MARKET_TYPE_HASHES[hash];
    expect(decoded).toBe('POSITION_SIZE');
  });
});
```

## Troubleshooting

### Unknown Market Type Warning

If you see this warning:

```
⚠️  Unknown marketType hash: 0x...
⚠️  Add this hash to MARKET_TYPE_HASHES mapping
```

**Solution**: Calculate the hash and add it to the backend mapping:

```bash
cast keccak "YOUR_MARKET_TYPE"
```

### Hash Mismatch

If the backend shows the wrong market type:

1. Verify contract constant: `cast call $CONTRACT "WINNER_PREDICTION()"`
2. Verify backend mapping matches
3. Ensure ABIs are up to date: `npm run copy-abis`

## Future Considerations

### Dynamic Market Types

If you need dynamic market types (not known at compile time), consider:

1. **Emit string directly**: Higher gas cost but more flexible
2. **Registry pattern**: Maintain on-chain hash → string mapping
3. **Hybrid approach**: Common types as hashes, rare types as strings

### Gas Cost Comparison

| Approach | Gas Cost | Pros | Cons |
|----------|----------|------|------|
| Hash (current) | ~21,000 | Cheapest, type-safe | Requires mapping |
| String bytes32 | ~21,200 | Human-readable | Slightly more gas |
| String dynamic | ~22,000+ | Flexible length | Most expensive |

## References

- OpenZeppelin AccessControl: Uses `bytes32` role identifiers
- EIP-712: Uses `keccak256` for type hashes
- Solidity Gas Optimization Guide: Recommends `bytes32` over `string`
