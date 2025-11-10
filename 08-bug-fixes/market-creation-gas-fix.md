# InfoFi Market Creation Gas Fix

## Problem Identified

InfoFi markets were not being created for new players despite the bonding curve correctly calling `InfoFiMarketFactory.onPositionUpdate()`.

### Root Cause

The Raffle contract's `try-catch` block was not forwarding enough gas to the InfoFiMarketFactory due to **gas consumption before the external call**.

**Evidence from transaction traces:**

Transaction `0x77cab1e1c9994b96b5523e88371a82c9c37564345bd80a9d28918874dc491c26`:

- Factory was called with function selector `ae52ab5c` (onPositionUpdate)
- Only **843 gas** was available to the factory
- Transaction reverted with `[OutOfGas] EvmError: OutOfGas`
- Total gas used: 428,568

### Why It Happened

1. **EIP-150 (63/64 Rule)**: External calls can only forward **at most 63/64** of remaining gas
2. **Gas Consumption Timing**: State updates before the InfoFi call consumed ~427K gas
3. **Insufficient Remaining Gas**: By the time the InfoFi call was made, only ~1.3K gas remained
4. **63/64 Rule Applied**: 1.3K × 63/64 = **843 gas** forwarded to InfoFi factory

### Why `{gas: 500000}` Didn't Work

The `{gas: 500000}` stipend is a **maximum limit**, not a guarantee. The actual gas forwarded is:

```text
min(stipend, remaining_gas × 63/64)
```

If only 1.3K gas remains, you can only forward ~843 gas regardless of the stipend.

## Solution Implemented

**Move the InfoFi call to the BEGINNING of the function** before state updates consume gas:

```solidity
function recordParticipant(...) {
    require(seasons[seasonId].isActive, "Raffle: season inactive");
    
    // Calculate new values FIRST
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
                // Success
            } catch {
                // Failure - backend will sync from events
            }
        }
    }
    
    // NOW update state after InfoFi call
    if (!pos.isActive) {
        state.participants.push(participant);
        // ... more state updates
    }
    // ...
}
```

### Why This Works

1. **Early Execution**: InfoFi call happens before expensive state updates
2. **Gas Check**: `gasleft() >= 600000` ensures we have enough to forward 500K
3. **Calculation**: 600K × 63/64 = ~590K available, enough for 500K stipend
4. **Fallback**: If not enough gas, skip InfoFi call - backend syncs from events

## Changes Made

**File**: `contracts/src/core/Raffle.sol`

**Functions Modified**:

1. `recordParticipant()` (lines 217-264):
   - Moved InfoFi call to beginning (line 232-246)
   - Added `gasleft() >= 600000` check
   - Calculate values before state updates
   - State updates happen AFTER InfoFi call

2. `removeParticipant()` (lines 266-317):
   - Same pattern as recordParticipant
   - InfoFi call at beginning (line 282-294)
   - State updates after InfoFi call

## Verification from Research

### Solidity Documentation

- **Official docs confirm**: "The caller always retains 63/64th of the gas in a call"
- **Try-catch behavior**: Automatically limits gas to prevent parent transaction failure

### OpenZeppelin Best Practices

- Explicit gas forwarding is the recommended pattern
- Used in OpenZeppelin's proxy contracts for delegate calls
- Provides predictability and testability

### EIP-150 (Gas Cost Changes)

- Introduced the 63/64 rule to prevent gas griefing attacks
- External calls can never consume more than 63/64 of available gas
- Ensures parent contract always has gas to handle failures

## Testing Required

1. ✅ Compile contracts: `forge build`
2. ⏳ Deploy updated contracts
3. ⏳ Buy tickets with multiple accounts
4. ⏳ Verify markets created for each player crossing 1% threshold
5. ⏳ Confirm no out-of-gas errors in transaction traces

## References

- [Solidity Docs - Try/Catch](https://docs.soliditylang.org/en/v0.8.20/control-structures.html#try-catch)
- [EIP-150: Gas Cost Changes](https://eips.ethereum.org/EIPS/eip-150)
- [RareSkills: EIP-150 and 63/64 Rule](https://rareskills.io/post/eip-150-and-the-63-64-rule-for-gas)
- [Cyfrin: 63/64 Gas Rule](https://www.cyfrin.io/glossary/63-64-gas-rule-solidity-code-example)

## Status

✅ **Fix Implemented**
⏳ **Awaiting Deployment & Testing**
