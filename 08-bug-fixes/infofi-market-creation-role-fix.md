# InfoFi Market Creation Fix - Missing RAFFLE_ROLE

**Date**: November 2, 2025  
**Status**: ⚠️ Requires Admin Action

---

## Problem Summary

New player (account[1] - `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`) bought 3000 tickets, crossing the 1% threshold with 42.85% probability, but no InfoFi market was created.

**Transaction**: `0x62af15e53ac91fbe9b43ef4756beb64efb9f2e5f926cbda5b60b46f1098cc03b`

---

## Root Cause

The Raffle contract does NOT have the `RAFFLE_ROLE` on the InfoFiMarketFactory contract.

### Evidence

```bash
# Check if Raffle has RAFFLE_ROLE on factory
cast call 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
  "hasRole(bytes32,address)(bool)" \
  $(cast keccak "RAFFLE_ROLE()") \
  0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 \
  --rpc-url http://localhost:8545

# Result: false ❌
```

### Why Markets Aren't Created

1. Player buys tickets → Raffle emits `PositionUpdate` event
2. Raffle calls `infoFiFactory.onPositionUpdate(...)` in try-catch block
3. Factory's `onPositionUpdate` has `onlyRole(RAFFLE_ROLE)` modifier
4. Raffle doesn't have the role → Function reverts
5. Raffle's try-catch silently swallows the error
6. No market created, no error logged

**Code Reference** (`contracts/src/core/Raffle.sol:243-252`):
```solidity
if (infoFiFactory != address(0)) {
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(
        seasonId, participant, oldTickets, newTicketsLocal, state.totalTickets
    ) {
        // Success - InfoFi market updated/created
    } catch {
        // InfoFi failure should not block raffle participation ❌ SILENT!
        // Event will be emitted by factory if market creation failed
    }
}
```

---

## The Fix

Grant `RAFFLE_ROLE` to the Raffle contract on the InfoFiMarketFactory.

### Option 1: Using Cast (Recommended)

```bash
# Get the RAFFLE_ROLE hash
RAFFLE_ROLE=$(cast keccak "RAFFLE_ROLE()")

# Grant role to Raffle contract
cast send 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
  "grantRole(bytes32,address)" \
  $RAFFLE_ROLE \
  0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 \
  --private-key $PRIVATE_KEY \
  --rpc-url http://localhost:8545
```

### Option 2: Using Forge Script

Create `contracts/script/GrantRaffleRole.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/infofi/InfoFiMarketFactory.sol";

contract GrantRaffleRole is Script {
    function run() external {
        address factoryAddress = vm.envAddress("INFOFI_FACTORY_ADDRESS_LOCAL");
        address raffleAddress = vm.envAddress("RAFFLE_ADDRESS");
        
        vm.startBroadcast();
        
        InfoFiMarketFactory factory = InfoFiMarketFactory(factoryAddress);
        bytes32 raffleRole = keccak256("RAFFLE_ROLE");
        
        factory.grantRole(raffleRole, raffleAddress);
        
        console.log("Granted RAFFLE_ROLE to Raffle contract");
        console.log("Factory:", factoryAddress);
        console.log("Raffle:", raffleAddress);
        
        vm.stopBroadcast();
    }
}
```

Run with:
```bash
forge script script/GrantRaffleRole.s.sol --rpc-url http://localhost:8545 --broadcast
```

---

## Verification

After granting the role, verify it was successful:

```bash
# Check role was granted
cast call 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
  "hasRole(bytes32,address)(bool)" \
  $(cast keccak "RAFFLE_ROLE()") \
  0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 \
  --rpc-url http://localhost:8545

# Should return: true ✅
```

---

## Testing After Fix

1. **Buy tickets with a new player** (or existing player who hasn't crossed 1% yet)
2. **Check for MarketCreated event**:
   ```bash
   cast logs --from-block latest "MarketCreated(uint256,address,bytes32,bytes32,address)" \
     --address 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
     --rpc-url http://localhost:8545
   ```
3. **Verify market in database**:
   ```bash
   curl -s http://localhost:3000/api/infofi/markets?seasonId=1 | jq '.markets["1"]'
   ```
4. **Check frontend** at `/markets` - should show new market

---

## Why This Happened

The role was likely not granted during initial deployment. The deployment script should include:

```solidity
// In deployment script
factory.grantRole(factory.RAFFLE_ROLE(), raffleAddress);
```

---

## Prevention

### Update Deployment Scripts

Ensure all deployment scripts grant the necessary roles:

**File**: `contracts/script/DeployInfoFi.s.sol`

```solidity
// After deploying factory
factory.grantRole(factory.RAFFLE_ROLE(), raffleAddress);
factory.grantRole(factory.TREASURY_ROLE(), treasuryAddress);

// Verify roles were granted
require(factory.hasRole(factory.RAFFLE_ROLE(), raffleAddress), "RAFFLE_ROLE not granted");
require(factory.hasRole(factory.TREASURY_ROLE(), treasuryAddress), "TREASURY_ROLE not granted");
```

### Add Deployment Checklist

Create `DEPLOYMENT_CHECKLIST.md`:

- [ ] Deploy InfoFiMarketFactory
- [ ] Grant RAFFLE_ROLE to Raffle contract
- [ ] Grant TREASURY_ROLE to Treasury address
- [ ] Set InfoFi factory address on Raffle contract
- [ ] Verify all roles with `hasRole()` calls
- [ ] Test market creation with small purchase
- [ ] Verify backend listeners are running
- [ ] Check database for market records

---

## Related Issues

### Silent Failure Pattern

The Raffle contract's try-catch silently swallows errors. Consider adding logging:

```solidity
if (infoFiFactory != address(0)) {
    try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate(...) {
        // Success
    } catch Error(string memory reason) {
        emit InfoFiCallFailed(seasonId, participant, reason);
    } catch (bytes memory) {
        emit InfoFiCallFailed(seasonId, participant, "Unknown error");
    }
}
```

This would help diagnose issues faster.

---

## Summary

**Problem**: Raffle contract missing `RAFFLE_ROLE` on InfoFiMarketFactory  
**Impact**: No markets created when players cross 1% threshold  
**Fix**: Grant role using cast or forge script  
**Time**: < 1 minute  
**Risk**: Low (only grants permission, doesn't change logic)

---

## Commands Quick Reference

```bash
# 1. Grant role
RAFFLE_ROLE=$(cast keccak "RAFFLE_ROLE()")
cast send 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
  "grantRole(bytes32,address)" \
  $RAFFLE_ROLE \
  0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 \
  --private-key $PRIVATE_KEY \
  --rpc-url http://localhost:8545

# 2. Verify
cast call 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
  "hasRole(bytes32,address)(bool)" \
  $RAFFLE_ROLE \
  0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 \
  --rpc-url http://localhost:8545

# 3. Test with new purchase
# (Buy tickets with any account)

# 4. Check for MarketCreated event
cast logs --from-block latest "MarketCreated(uint256,address,bytes32,bytes32,address)" \
  --address 0x959922bE3CAee4b8Cd9a407cc3ac1C251C2007B1 \
  --rpc-url http://localhost:8545
```

---

**Status**: Ready to fix - requires admin transaction to grant role.
