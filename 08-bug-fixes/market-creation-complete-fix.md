# InfoFi Market Creation - Complete Fix

## Problem Summary

InfoFi markets were not being created when players crossed the 1% threshold, despite all the infrastructure being in place.

## Root Cause Analysis

### Issue 1: Gas Consumption Timing (Contract-Side)

**Problem**: The Raffle contract was calling InfoFiMarketFactory.onPositionUpdate() **after** expensive state updates, leaving insufficient gas to forward.

**Evidence from Transaction `0x77cab1e1c9994b96b5523e88371a82c9c37564345bd80a9d28918874dc491c26`**:

- Total gas used: 428,568
- Gas available for InfoFi call: **843** (only 1/64th of remaining gas)
- InfoFi call failed with `[OutOfGas] EvmError: OutOfGas`

**Why `{gas: 500000}` didn't work**:

The `{gas: 500000}` stipend is a **maximum limit**, not a guarantee. The actual gas forwarded is:

```text
min(stipend, remaining_gas × 63/64)
```

If only 1.3K gas remains when the call is made, you can only forward ~843 gas regardless of the stipend.

### Issue 2: Insufficient Gas Limit (Frontend-Side)

**Problem**: Even after moving the InfoFi call to the beginning of the function, the transaction still had insufficient total gas.

**Evidence from Transaction `0x15db6bf493197aa08f1f0ec7d7b8c7de087f0adec1fc733463dd3fb0036537e8`**:

- Gas limit: **428,837**
- Gas used: 423,322
- InfoFi call was **skipped** because `gasleft() < 600000` check failed

**Why this happened**:

1. Wagmi/Viem estimated gas based on historical transactions
2. Historical transactions didn't include successful InfoFi market creation
3. Estimation was too low for the new code path

## Complete Solution

### Part 1: Contract Changes (Raffle.sol)

**Move InfoFi call to the BEGINNING of the function** before state updates:

```solidity
function recordParticipant(uint256 seasonId, address participant, uint256 ticketAmount)
    external
    onlyRole(BONDING_CURVE_ROLE)
{
    require(seasons[seasonId].isActive, "Raffle: season inactive");
    
    // Calculate new values FIRST (minimal gas consumption)
    SeasonState storage state = seasonStates[seasonId];
    ParticipantPosition storage pos = state.participantPositions[participant];
    uint256 oldTickets = pos.ticketCount;
    uint256 newTicketsLocal = oldTickets + ticketAmount;
    uint256 newTotalTickets = state.totalTickets + ticketAmount;
    
    // Call InfoFi factory EARLY before consuming gas on state updates
    if (infoFiFactory != address(0)) {
        // Ensure we have enough gas remaining
        if (gasleft() >= 600000) {
            try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate{gas: 500000}(
                seasonId, participant, oldTickets, newTicketsLocal, newTotalTickets
            ) {
                // Success - InfoFi market updated/created
            } catch {
                // InfoFi failure should not block raffle participation
            }
        }
        // If not enough gas, skip InfoFi call - backend will sync from events
    }
    
    // NOW update state after InfoFi call
    if (!pos.isActive) {
        state.participants.push(participant);
        // ... more state updates
    }
    // ...
}
```

**Key improvements**:

1. ✅ InfoFi call happens **first** before expensive state updates
2. ✅ `gasleft() >= 600000` check ensures sufficient gas (600K × 63/64 = ~590K available)
3. ✅ Values calculated upfront with minimal gas consumption
4. ✅ Graceful fallback: if not enough gas, skip InfoFi (backend syncs from events)

### Part 2: Frontend Changes (useCurve.js)

**Add explicit gas limit** to ensure enough gas for InfoFi market creation:

```javascript
const buyTokensMutation = useMutation({
  mutationFn: async ({ tokenAmount, maxSofAmount }) => {
    return await writeContractAsync({
      ...curveContractConfig,
      functionName: 'buyTokens',
      args: [tokenAmount, maxSofAmount],
      gas: 800000n, // Explicit gas limit: 800K ensures enough for InfoFi
    });
  },
});
```

**Why 800K gas**:

- Bonding curve operations: ~200K gas
- Raffle recordParticipant: ~200K gas
- InfoFi market creation: ~300K gas
- Buffer for safety: ~100K gas
- **Total: ~800K gas**

## Gas Flow Breakdown

### With 800K Gas Limit

```text
Transaction starts: 800,000 gas
  ↓
BondingCurve.buyTokens(): -200,000 gas
  ↓ (600,000 gas remaining)
Raffle.recordParticipant() entry: 600,000 gas
  ↓
gasleft() >= 600000? ✅ YES
  ↓
Calculate values: -5,000 gas
  ↓ (595,000 gas remaining)
InfoFi call with {gas: 500000}: forwards ~585K gas (595K × 63/64)
  ↓
InfoFiMarketFactory.onPositionUpdate(): uses ~300K gas ✅ SUCCESS
  ↓
State updates: -200,000 gas
  ↓
Transaction complete: ~423K gas used total
```

### With Old 428K Gas Limit (Failed)

```text
Transaction starts: 428,000 gas
  ↓
BondingCurve.buyTokens(): -200,000 gas
  ↓ (228,000 gas remaining)
Raffle.recordParticipant() entry: 228,000 gas
  ↓
gasleft() >= 600000? ❌ NO - InfoFi call skipped
  ↓
State updates: -200,000 gas
  ↓
Transaction complete: ~423K gas used, NO MARKET CREATED
```

## Files Modified

### Smart Contracts

1. **`contracts/src/core/Raffle.sol`**
   - `recordParticipant()` (lines 217-264): Moved InfoFi call to beginning
   - `removeParticipant()` (lines 266-317): Same pattern

### Frontend

2. **`src/hooks/useCurve.js`**
   - `buyTokensMutation` (line 52): Added `gas: 800000n`

### Documentation

3. **`MARKET_CREATION_GAS_FIX.md`**: Detailed analysis
4. **`MARKET_CREATION_COMPLETE_FIX.md`**: This document

## Testing Checklist

- [ ] Compile contracts: `cd contracts && forge build`
- [ ] Deploy updated contracts
- [ ] Restart frontend dev server
- [ ] Buy tickets with account that crosses 1% threshold
- [ ] Verify InfoFi market created on-chain
- [ ] Verify market appears in database
- [ ] Verify market visible on frontend
- [ ] Check transaction trace shows successful InfoFi call

## Verification Commands

```bash
# Check if market was created on-chain
cast call $INFOFI_FACTORY_ADDRESS "marketCreated(uint256,address)(bool)" $SEASON_ID $PLAYER_ADDRESS --rpc-url http://127.0.0.1:8545

# Get market details
cast call $INFOFI_FACTORY_ADDRESS "getMarket(uint256,address)" $SEASON_ID $PLAYER_ADDRESS --rpc-url http://127.0.0.1:8545

# Trace transaction to verify InfoFi call succeeded
cast run $TX_HASH --rpc-url http://127.0.0.1:8545
```

## Expected Behavior After Fix

1. ✅ User buys tickets via frontend
2. ✅ Transaction uses 800K gas limit
3. ✅ Raffle.recordParticipant() called with 600K+ gas remaining
4. ✅ InfoFi call succeeds with 500K gas forwarded
5. ✅ Market created on-chain
6. ✅ Backend listener syncs market to database
7. ✅ Market appears on frontend

## Key Learnings

1. **Gas stipends are maximums, not guarantees** - `{gas: 500000}` doesn't guarantee 500K gas will be forwarded
2. **EIP-150 (63/64 rule) always applies** - External calls can only forward 63/64 of remaining gas
3. **Order matters** - Expensive operations before external calls reduce available gas
4. **Frontend gas estimation can be wrong** - Explicit gas limits may be necessary
5. **Always check gasleft()** - Verify sufficient gas before making external calls

## Status

✅ **Contract Fix**: Implemented and compiled
✅ **Frontend Fix**: Implemented
⏳ **Deployment**: Awaiting deployment and testing
⏳ **Verification**: Awaiting test transaction

## Next Steps

1. Deploy updated contracts
2. Restart frontend
3. Test with new ticket purchase
4. Verify market creation succeeds
5. Monitor gas usage and adjust if needed
