# Position Tracker Listener Improvements

## Overview

Enhanced the `positionTrackerListener.js` to properly watch `RafflePositionTracker` contract events using Viem best practices, including historical event scanning and improved error handling.

## Key Improvements

### 1. Historical Event Scanning

**New Function: `scanHistoricalPositionSnapshots()`**

- Scans past `PositionSnapshot` events using Viem's `getContractEvents()` API
- Automatically runs on listener startup to catch any missed events
- Configurable block range (defaults to last 1000 blocks)
- Processes historical events before starting live listener

```javascript
const historicalCount = await scanHistoricalPositionSnapshots(
  networkKey, 
  fromBlock, 
  'latest', 
  logger
);
```

### 2. Modular Event Processing

**New Function: `processPositionSnapshotEvent()`**

- Extracted event processing logic into reusable function
- Used by both historical scanner and live listener
- Consistent error handling and logging
- Properly re-throws errors for caller handling

### 3. Improved Live Listener Configuration

**Enhanced `watchContractEvent()` call:**

```javascript
client.watchContractEvent({
  address: chain.positionTracker,
  abi: RafflePositionTrackerAbi,
  eventName: 'PositionSnapshot',
  pollingInterval: 3000,
  batch: false, // Process events immediately
  onLogs: async (logs) => { /* ... */ },
  onError: (error) => { /* ... */ }
})
```

**Key settings:**

- `batch: false` - Process events immediately instead of batching
- `pollingInterval: 3000` - Poll every 3 seconds for new events
- Proper error handling with `onError` callback


### 4. Async/Await Pattern

**Made `startPositionTrackerListener()` async:**

- Allows historical scanning before starting live listener
- Proper error handling with try/catch
- Updated server.js to await the function call

```javascript
// In server.js
const stopTracker = await startPositionTrackerListener(c.key, app.log);
```

## Event Flow

### Startup Sequence

1. **Initialize** - Load chain config and create Viem client
2. **Historical Scan** - Fetch and process past events (last 1000 blocks)
3. **Start Live Listener** - Begin watching for new events
4. **Process Events** - Handle each event as it arrives

### Event Processing

For each `PositionSnapshot` event:

1. Extract event data (player, ticketCount, totalTickets, winProbabilityBps, seasonId)
2. Get or create player record in database
3. Check if InfoFi market exists for this player
4. If market exists: Update probability in database
5. If no market: Log info (market will be created by InfoFiMarketFactory)

## Viem Best Practices Applied

### 1. Use `getContractEvents()` for Historical Data

```javascript
const logs = await client.getContractEvents({
  address: chain.positionTracker,
  abi: RafflePositionTrackerAbi,
  eventName: 'PositionSnapshot',
  fromBlock,
  toBlock: toBlock === 'latest' ? undefined : toBlock
});
```

### 2. Use `watchContractEvent()` for Live Updates

```javascript
const unwatch = client.watchContractEvent({
  address: contractAddress,
  abi: contractAbi,
  eventName: 'EventName',
  onLogs: (logs) => { /* process */ },
  onError: (error) => { /* handle */ }
});
```

### 3. Proper Error Handling

- Separate try/catch blocks for historical vs live listener
- Error logging with full context
- Graceful degradation (continue if historical scan fails)
- Re-throw errors from event processing for visibility

### 4. Event Filtering

- Use `eventName` parameter to filter specific events
- Use `address` parameter to watch specific contract
- Use `fromBlock`/`toBlock` for historical range queries

## Testing

### Manual Testing

1. **Start backend** - Watch logs for historical scan
2. **Buy tickets** - Trigger PositionSnapshot event on-chain
3. **Check logs** - Verify event received and processed
4. **Check database** - Verify probability updated in infofi_markets

### Expected Log Output

```text
[positionTrackerListener] 🚀 INITIALIZING POSITION TRACKER LISTENER
[positionTrackerListener] 🔍 Scanning historical events...
[positionTrackerListener] 📊 Found 5 historical PositionSnapshot events
[positionTrackerListener] 🔄 Processing PositionSnapshot:
[positionTrackerListener]   Block: 12345
[positionTrackerListener]   Season: 1
[positionTrackerListener]   Player: 0x1234...5678
[positionTrackerListener]   Tickets: 2000 / 2000
[positionTrackerListener]   Win Probability: 10000 bps (100.00%)
[positionTrackerListener] ✅ Successfully updated probability
[positionTrackerListener] 🎯 Starting live event listener
[positionTrackerListener] ✅ LISTENER SUCCESSFULLY STARTED
```

## Troubleshooting

### No Events Received

**Check:**

1. Position tracker address configured in `.env`
2. Contract deployed and has emitted events
3. RPC endpoint accessible and responding
4. ABI matches deployed contract

**Debug:**

```bash
# Check if contract exists
cast code $RAFFLE_TRACKER_ADDRESS_LOCAL --rpc-url http://127.0.0.1:8545

# Check recent events
cast logs --address $RAFFLE_TRACKER_ADDRESS_LOCAL --rpc-url http://127.0.0.1:8545
```

### Historical Scan Fails

**Common causes:**

- Block range too large (reduce to smaller range)
- RPC rate limiting (add delay between requests)
- Contract not deployed at fromBlock (adjust starting block)

**Solution:**

- Listener continues with live events even if historical scan fails
- Check warning logs for specific error message


### Events Not Updating Database

**Check:**

1. Database connection working (check Supabase logs)
2. Player record exists or can be created
3. Market exists in database (created by InfoFiMarketFactory)
4. Correct column names used (season_id vs raffle_id)


## Related Files

- `/backend/src/services/positionTrackerListener.js` - Main listener implementation
- `/backend/fastify/server.js` - Listener initialization
- `/backend/src/abis/RafflePositionTrackerAbi.js` - Contract ABI
- `/contracts/src/core/RafflePositionTracker.sol` - Smart contract source

## References

- [Viem watchContractEvent Documentation](https://viem.sh/docs/contract/watchContractEvent)
- [Viem getContractEvents Documentation](https://viem.sh/docs/contract/getContractEvents)
- [RafflePositionTracker Contract](../../contracts/src/core/RafflePositionTracker.sol)
