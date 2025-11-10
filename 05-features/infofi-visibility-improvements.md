# InfoFi Market Creation Visibility Improvements

## Problem

When InfoFi market creation was skipped due to insufficient gas, it failed silently with no indication of why the market wasn't created. This made debugging extremely difficult.

## Solution Implemented

Added explicit event emission when InfoFi market creation is skipped due to insufficient gas.

### Changes Made

#### 1. New Event Definition

**File**: `contracts/src/core/Raffle.sol` (lines 55-58)

```solidity
/// @dev Emitted when InfoFi market creation is skipped due to insufficient gas
event InfoFiSkippedInsufficientGas(
    uint256 indexed seasonId, 
    address indexed player, 
    uint256 gasAvailable, 
    uint256 gasRequired
);
```

#### 2. Event Emission in recordParticipant()

**File**: `contracts/src/core/Raffle.sol` (lines 237-254)

```solidity
if (infoFiFactory != address(0)) {
    uint256 gasAvailable = gasleft();
    if (gasAvailable >= 600000) {
        try IInfoFiMarketFactory(infoFiFactory).onPositionUpdate{gas: 500000}(
            seasonId, participant, oldTickets, newTicketsLocal, newTotalTickets
        ) {
            // Success - InfoFi market updated/created
        } catch {
            // InfoFi failure should not block raffle participation
        }
    } else {
        // Emit event when skipping due to insufficient gas
        emit InfoFiSkippedInsufficientGas(seasonId, participant, gasAvailable, 600000);
    }
}
```

#### 3. Event Emission in removeParticipant()

**File**: `contracts/src/core/Raffle.sol` (lines 290-306)

Same pattern as `recordParticipant()`.

## Benefits

### 1. Debugging Visibility

Transaction traces will now show:
```
emit InfoFiSkippedInsufficientGas(
    seasonId: 1,
    player: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8,
    gasAvailable: 189357,
    gasRequired: 600000
)
```

This immediately tells you:
- ✅ InfoFi factory is configured
- ✅ The check is running
- ❌ Insufficient gas (189K vs 600K required)
- 💡 Need to increase transaction gas limit

### 2. Backend Monitoring

The backend can listen for this event and:
- Log warnings when markets are skipped
- Alert admins if this happens frequently
- Track gas usage patterns
- Recommend optimal gas limits

### 3. Frontend User Experience

The frontend can:
- Detect when markets fail to create
- Show user-friendly error messages
- Suggest increasing gas limit
- Provide troubleshooting guidance

## Event Data Structure

```typescript
interface InfoFiSkippedInsufficientGas {
  seasonId: bigint;      // Which season
  player: string;        // Which player's market was skipped
  gasAvailable: bigint;  // How much gas was available
  gasRequired: bigint;   // How much gas was needed (600000)
}
```

## Backend Listener Implementation

```javascript
// Listen for InfoFiSkippedInsufficientGas events
publicClient.watchContractEvent({
  address: raffleAddress,
  abi: RaffleAbi,
  eventName: 'InfoFiSkippedInsufficientGas',
  onLogs: (logs) => {
    logs.forEach((log) => {
      const { seasonId, player, gasAvailable, gasRequired } = log.args;
      
      logger.warn({
        msg: '⚠️ InfoFi market creation skipped - insufficient gas',
        seasonId: seasonId.toString(),
        player,
        gasAvailable: gasAvailable.toString(),
        gasRequired: gasRequired.toString(),
        deficit: (gasRequired - gasAvailable).toString()
      });
      
      // Could trigger admin alert if this happens frequently
    });
  }
});
```

## Testing

After deploying the updated contract, you can verify the event is emitted by:

1. **Send transaction with low gas** (e.g., 500K)
2. **Check transaction receipt** for `InfoFiSkippedInsufficientGas` event
3. **Verify event data** shows correct gas values

Example verification:
```bash
# Get transaction receipt
cast receipt $TX_HASH --rpc-url http://127.0.0.1:8545

# Look for event with topic:
# keccak256("InfoFiSkippedInsufficientGas(uint256,address,uint256,uint256)")
```

## Next Steps

1. ✅ Contract changes implemented
2. ⏳ Compile contracts: `cd contracts && forge build`
3. ⏳ Deploy updated contracts
4. ⏳ Copy ABIs: `node scripts/copy-abis.js`
5. ⏳ Add backend listener for the new event
6. ⏳ Test with insufficient gas to verify event emission
7. ⏳ Test with sufficient gas (1.2M) to verify market creation

## Related Changes

This improvement works in conjunction with:
- **Frontend gas limit increase** to 1.2M (`src/hooks/useCurve.js`)
- **Contract gas check** at 600K threshold (`contracts/src/core/Raffle.sol`)
- **InfoFi factory call** with 500K gas stipend

## Status

✅ **Event defined**
✅ **Event emission added to both functions**
⏳ **Awaiting deployment and testing**
