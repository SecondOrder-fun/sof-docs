# SeasonStarted Event Listener - Design Plan

## Overview

Build a backend listener that subscribes to the `SeasonStarted` event emitted by the Raffle contract and logs when a season starts.

**Event Definition** (from Raffle.sol line 181):

```solidity
event SeasonStarted(uint256 indexed seasonId);
```

---

## Architecture Decision

### Why Viem's `watchContractEvent`?

1. **Type Safety**: Full TypeScript support with ABI-inferred types
2. **Polling Support**: Works with HTTP RPC (no WebSocket required)
3. **Event Filtering**: Built-in filtering by event name and indexed parameters
4. **Error Handling**: Dedicated error callback for robust error management
5. **Unwatch Capability**: Clean subscription management with unwatch function
6. **Production Ready**: Used across major Ethereum projects

### Why NOT Ethers.js?

- Viem is lighter-weight and more modern
- Better TypeScript support
- More composable API
- Aligns with existing Wagmi/Viem stack in frontend

---

## Step-by-Step Implementation Plan

### Phase 1: Project Structure Setup

**Location**: `backend/src/listeners/seasonStartedListener.js`

**Directory Structure**:

```text
backend/
├── src/
│   ├── listeners/
│   │   └── seasonStartedListener.js    (NEW - this file)
│   ├── lib/
│   │   └── viemClient.js               (EXISTING - public client)
│   └── config/
│       └── env.js                      (EXISTING - env vars)
├── fastify/
│   └── server.js                       (MODIFY - initialize listener)
```

**Responsibilities**:

- `seasonStartedListener.js`: Core listener logic
- `server.js`: Initialize listener on startup, handle graceful shutdown

---

### Phase 2: Viem Client Setup

**Requirement**: Ensure `backend/src/lib/viemClient.js` exports a configured public client

**Configuration Needed**:

```javascript
import { createPublicClient, http } from 'viem';
import { sepolia } from 'viem/chains'; // or mainnet for production

const publicClient = createPublicClient({
  chain: sepolia,
  transport: http(process.env.RPC_URL),
  pollingInterval: 3000, // 3 second polling for HTTP
});

export { publicClient };
```

**Key Parameters**:

- `chain`: Network (Sepolia for testnet, mainnet for production)
- `transport`: HTTP RPC endpoint from `process.env.RPC_URL`
- `pollingInterval`: 3000ms (3 seconds) for efficient polling

---

### Phase 3: ABI Import

**Requirement**: Import the full Raffle ABI from the auto-generated file

**What We Use**:

```javascript
import raffleAbi from '../abis/RaffleAbi.js';

// raffleAbi contains the complete ABI including:
// - Constructor
// - All public/external functions
// - All events (including SeasonStarted)
// - All state variables
```

**Source**:

- Full Raffle ABI auto-generated from Foundry compilation
- Located at: `backend/src/abis/RaffleAbi.js` (auto-generated, do not edit manually)
- Original compiled output: `contracts/out/Raffle.sol/Raffle.json`
- Event definition: Raffle.sol line 181 - `event SeasonStarted(uint256 indexed seasonId);`

**Why Full ABI?**

- Viem's `watchContractEvent` works with full ABIs
- Viem uses the full ABI to infer types and validate event names
- No need to extract or create mini ABIs
- Single source of truth for contract interface

---

### Phase 4: Listener Implementation

**Core Listener Function**:

```javascript
/**
 * Starts listening for SeasonStarted events from the Raffle contract
 * @param {string} raffleAddress - Raffle contract address
 * @param {object} raffleAbi - Raffle contract ABI
 * @param {object} logger - Fastify logger instance (app.log)
 * @returns {function} Unwatch function to stop listening
  */
async function startSeasonStartedListener(raffleAddress, raffleAbi, logger) {
  // Validate inputs
  if (!raffleAddress || !raffleAbi) {
    throw new Error('raffleAddress and raffleAbi are required');
  }

  // Start watching for SeasonStarted events
  const unwatch = publicClient.watchContractEvent({
    address: raffleAddress,
    abi: raffleAbi,
    eventName: 'SeasonStarted',
    onLogs: (logs) => {
      logs.forEach((log) => {
        const { seasonId } = log.args;
        logger.info(`✅ SeasonStarted Event: Season ${seasonId} has started`);
        // Future: Trigger backend actions (database updates, etc.)
      });
    },
    onError: (error) => {
      logger.error('❌ SeasonStarted Listener Error:', error.message);
      // Future: Implement retry logic or alerting
    },
    poll: true, // Use polling for HTTP transport
    pollingInterval: 3000, // Check every 3 seconds
  });

  logger.info(`🎧 Listening for SeasonStarted events on ${raffleAddress}`);
  return unwatch;
}
```

**Key Design Decisions**:

1. **`eventName: 'SeasonStarted'`**: Filter to only this event
2. **`poll: true`**: Use polling instead of WebSocket (works with HTTP)
3. **`pollingInterval: 3000`**: 3-second polling interval (balance between latency and RPC load)
4. **`onLogs` callback**: Processes each event as it arrives
5. **`onError` callback**: Handles connection/parsing errors gracefully
6. **Return `unwatch`**: Allows caller to stop listening when needed

---

### Phase 5: Server Integration

**Modify `backend/fastify/server.js`**:

```javascript
import { startSeasonStartedListener } from '../src/listeners/seasonStartedListener.js';
import raffleAbi from '../src/abis/RaffleAbi.js';

// In server startup:
let unwatchSeasonStarted;

async function startListeners() {
  try {
    const raffleAddress = process.env.RAFFLE_ADDRESS;
    
    if (!raffleAddress) {
      throw new Error('RAFFLE_ADDRESS not set in environment');
    }
    
    unwatchSeasonStarted = await startSeasonStartedListener(raffleAddress, raffleAbi, app.log);
  } catch (error) {
    app.log.error('Failed to start listeners:', error);
    // Don't crash server, but log the error
  }
}

// On graceful shutdown:
async function shutdown() {
  if (unwatchSeasonStarted) {
    unwatchSeasonStarted();
    app.log.info('🛑 Stopped SeasonStarted listener');
  }
  // ... other cleanup
}
```

---

### Phase 6: Error Handling & Resilience

**Error Scenarios**:

1. **Invalid Contract Address**: Caught by Viem validation
2. **Invalid ABI**: Caught by Viem type checking
3. **RPC Connection Failure**: Handled by `onError` callback
4. **Event Parsing Error**: Handled by `onError` callback
5. **Network Interruption**: Viem automatically retries polling

**Logging Strategy**:

```javascript
// Success case
app.log.info(`✅ SeasonStarted Event: Season ${seasonId} has started`);

// Error case
app.log.error('❌ SeasonStarted Listener Error:', error.message);

// Startup
app.log.info(`🎧 Listening for SeasonStarted events on ${raffleAddress}`);

// Shutdown
app.log.info('🛑 Stopped SeasonStarted listener');
```

**Note**: Use `app.log` (Fastify logger) instead of `console.log` for consistency with server logging and proper log level management.

---

### Phase 7: Environment Variables

**Required in `.env`**:

```bash
# RPC Configuration
RPC_URL=http://127.0.0.1:8545          # Local Anvil or Sepolia RPC
RAFFLE_ADDRESS=0x...                    # Deployed Raffle contract address

# Optional
LISTENER_POLL_INTERVAL=3000             # Polling interval in ms (default: 3000)
```

---

## Testing Strategy

### Unit Test: Listener Initialization

```javascript
// Test that listener starts without errors
test('startSeasonStartedListener initializes successfully', async () => {
  const unwatch = await startSeasonStartedListener(raffleAddress, raffleAbi);
  expect(unwatch).toBeDefined();
  expect(typeof unwatch).toBe('function');
  unwatch(); // Stop listening
});
```

### Integration Test: Event Reception

```javascript
// Test on local Anvil:
// 1. Deploy Raffle contract
// 2. Create a season
// 3. Call startSeason()
// 4. Verify listener logs the SeasonStarted event
```

### Manual Testing Checklist

- [ ] Backend starts successfully with listener initialized
- [ ] Listener logs appear in console
- [ ] Create season on Anvil
- [ ] Start season on Anvil
- [ ] Verify `✅ SeasonStarted Event: Season 1 has started` appears in logs
- [ ] Stop backend gracefully (Ctrl+C)
- [ ] Verify `🛑 Stopped SeasonStarted listener` appears in logs

---

## Viem Best Practices Applied

| Practice | Implementation |
|----------|-----------------|
| **Type Safety** | Use TypeScript with ABI-inferred types |
| **Error Handling** | Dedicated `onError` callback |
| **Resource Cleanup** | Return `unwatch()` function |
| **Polling Configuration** | Set `poll: true` and `pollingInterval` |
| **Event Filtering** | Use `eventName` parameter |
| **Logging** | Structured console logs with emoji prefixes |

---

## OpenZeppelin Best Practices Applied

| Practice | Implementation |
|----------|-----------------|
| **Event Indexing** | Listen to indexed `seasonId` parameter |
| **Access Control** | Listener doesn't modify state (read-only) |
| **Error Resilience** | Graceful error handling without state corruption |
| **Separation of Concerns** | Listener isolated in `backend/src/listeners/` |

---

## Future Enhancements (Out of Scope)

1. **Database Persistence**: Store events in Supabase
2. **Event Replay**: Scan historical events on startup
3. **Multiple Event Types**: Extend to listen for other Raffle events
4. **Metrics & Monitoring**: Track listener health
5. **Retry Logic**: Exponential backoff for failed connections
6. **Event Aggregation**: Batch process multiple events

---

## File Checklist

- [ ] Create `backend/src/listeners/seasonStartedListener.js`
- [ ] Verify `backend/src/lib/viemClient.js` exports configured client
- [ ] Modify `backend/fastify/server.js` to initialize listener
- [ ] Verify `.env` contains `RAFFLE_ADDRESS` and `RPC_URL`
- [ ] Extract Raffle ABI from compiled contracts
- [ ] Test listener on local Anvil

---

## Success Criteria

✅ **Listener starts without errors on server startup**
✅ **Logs appear when season is started on-chain**
✅ **Listener gracefully stops on server shutdown**
✅ **Error handling prevents server crashes**
✅ **Code follows Viem best practices**
✅ **Code follows OpenZeppelin patterns**
