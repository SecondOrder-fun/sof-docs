# SeasonStarted Event Listener - Data Retrieval Enhancement Plan

**Date**: Oct 25, 2025  
**Status**: Planning Phase - Awaiting Approval

---

## Executive Summary

Enhance the existing `SeasonStarted` event listener to automatically retrieve on-chain season data (BondingCurve address and RaffleToken address) when a season starts, enabling downstream systems to immediately interact with the new season's contracts.

---

## Problem Statement

Currently, the listener only logs that a season has started. It doesn't retrieve the associated contract addresses needed by other backend systems:
- **BondingCurve address**: Required for ticket purchasing and position tracking
- **RaffleToken address**: Required for token operations and balance queries

This forces manual lookup or requires other systems to poll for this data.

---

## Proposed Solution

### Architecture

```
SeasonStarted Event (seasonId)
    ↓
seasonStartedListener.js (onLogs callback)
    ↓
readContract(getSeasonDetails) via Viem
    ↓
Extract: bondingCurve, raffleToken addresses
    ↓
Store in database + emit event for other listeners
    ↓
Log success with contract addresses
```

### Implementation Details

#### Phase 1: Event Data Handler Enhancement

**File**: `backend/src/listeners/seasonStartedListener.js`

**Changes**:
1. Extract `seasonId` from event args
2. Call `publicClient.readContract()` to fetch season details
3. Parse response to get `bondingCurve` and `raffleToken` addresses
4. Log retrieved addresses with formatted output
5. Handle errors gracefully

**Code Pattern**:
```javascript
onLogs: async (logs) => {
  for (const log of logs) {
    const { seasonId } = log.args;
    
    // Retrieve season details from contract
    const seasonDetails = await publicClient.readContract({
      address: raffleAddress,
      abi: raffleAbi,
      functionName: 'getSeasonDetails',
      args: [seasonId],
    });
    
    // Extract addresses
    const { bondingCurve, raffleToken } = seasonDetails.config;
    
    // Log and store
    logger.info(`✅ Season ${seasonId} started`);
    logger.info(`   BondingCurve: ${bondingCurve}`);
    logger.info(`   RaffleToken: ${raffleToken}`);
  }
}
```

#### Phase 2: Data Persistence (Future)

**Database Storage**:
- Store retrieved addresses in `seasons` table
- Enable quick lookup without re-querying contract
- Track retrieval timestamp for auditing

**Event Emission** (Future):
- Emit `SeasonDataRetrieved` event for other listeners
- Allow downstream systems to react to data availability

---

## Technical Specifications

### Viem readContract Usage

**Function**: `publicClient.readContract()`  
**Purpose**: Call read-only contract functions  
**Async**: Yes (returns Promise)

**Parameters**:
- `address`: Raffle contract address (from env)
- `abi`: Raffle contract ABI (already imported)
- `functionName`: `'getSeasonDetails'`
- `args`: `[seasonId]` (from event)

**Return Type**:
```typescript
{
  config: {
    name: string,
    startTime: bigint,
    endTime: bigint,
    winnerCount: number,
    grandPrizeBps: number,
    raffleToken: address,      // ← We need this
    bondingCurve: address,     // ← We need this
    isActive: boolean,
    isCompleted: boolean
  },
  status: SeasonStatus,
  totalParticipants: bigint,
  totalTickets: bigint,
  totalPrizePool: bigint
}
```

### Error Handling

**Potential Errors**:
1. **Invalid seasonId**: Contract reverts if season doesn't exist
2. **RPC timeout**: Network latency or RPC provider issues
3. **Contract not deployed**: Address is invalid or contract not found

**Strategy**:
- Wrap `readContract` in try-catch
- Log detailed error information
- Don't crash listener on individual failures
- Continue listening for next events

---

## Implementation Checklist

- [ ] Modify `seasonStartedListener.js` to make `onLogs` async
- [ ] Add `readContract` call inside `onLogs` callback
- [ ] Extract `bondingCurve` and `raffleToken` from response
- [ ] Add formatted logging for retrieved addresses
- [ ] Add error handling with detailed logging
- [ ] Test with deployed contracts on Anvil
- [ ] Verify addresses match deployment output
- [ ] Document the data flow

---

## Testing Strategy

### Unit Test

```javascript
// Mock publicClient.readContract to return season details
// Verify onLogs calls readContract with correct parameters
// Verify logging output contains addresses
```

### Integration Test (Manual)

1. **Deploy contracts**: `npm run anvil:deploy`
2. **Start backend**: `npm run dev:backend`
3. **Create season**: `cd contracts && export $(cat ../.env | xargs) && forge script script/CreateSeason.s.sol --broadcast`
4. **Wait 61 seconds** for startTime
5. **Start season**: `cast send $RAFFLE_ADDRESS "startSeason(uint256)" 1`
6. **Verify logs show**:

```text
✅ Season 1 started
   BondingCurve: 0x...
   RaffleToken: 0x...
```

---

## Benefits

✅ **Automatic Discovery**: No manual address lookup needed  
✅ **Real-time**: Data available immediately when season starts  
✅ **Downstream Integration**: Other listeners can use retrieved addresses  
✅ **Audit Trail**: Logged addresses for verification  
✅ **Error Resilience**: Graceful handling of contract read failures  

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| RPC timeout on readContract | Implement timeout with fallback logging |
| Invalid seasonId in event | Contract validation; log and continue |
| Performance impact | Async/await prevents blocking; monitor latency |
| Data inconsistency | Verify addresses match deployment records |

---

## Future Enhancements

1. **Database Storage**: Persist retrieved addresses for quick lookup
2. **Event Emission**: Trigger downstream listeners when data retrieved
3. **Caching**: Cache season details to reduce RPC calls
4. **Batch Retrieval**: Handle multiple seasons efficiently
5. **Metrics**: Track retrieval success rate and latency

---

## Approval Checklist

- [ ] Approach aligns with project architecture
- [ ] Viem usage follows best practices
- [ ] Error handling is comprehensive
- [ ] Testing strategy is feasible
- [ ] No blocking concerns identified

**Ready to proceed with implementation?**
