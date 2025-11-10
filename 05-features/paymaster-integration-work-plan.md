# Base Paymaster Integration - Work Plan

**Project:** SecondOrder.fun Gasless Market Creation  
**Date:** November 6, 2025  
**Status:** Ready for Implementation

---

## 🎯 Project Objective

Replace direct bonding curve → market factory contract calls with a gas-sponsored flow using Base's Paymaster service. The system will:
1. Listen for events from the bonding curve contract
2. Use Base Paymaster to sponsor gas for market creation
3. Bundle transactions using ERC-4337 (Account Abstraction)
4. Provide real-time updates to frontend via SSE

---

## 🏗️ Architecture Overview

### System Flow
```
Bonding Curve Contract (on-chain)
    ↓ [emits MarketCreationRequested event]
Event Listener Service (backend)
    ↓ [processes event]
Paymaster Service (backend)
    ↓ [creates sponsored UserOperation]
Base Paymaster Network
    ↓ [executes gasless transaction]
Market Factory Contract (on-chain)
    ↓ [creates market]
Frontend (via SSE)
    ↓ [displays real-time updates]
```

### Key Components

1. **Event Listener Service** (Fastify + Viem)
   - Monitors bonding curve contract events via WebSocket
   - Extracts market creation parameters
   - Triggers paymaster service

2. **Paymaster Service** (Viem Account Abstraction)
   - Creates Smart Account with backend as owner
   - Builds sponsored UserOperations
   - Submits to Base Paymaster network
   - Tracks transaction status

3. **SSE Service** (Fastify SSE)
   - Streams real-time updates to frontend
   - Reports transaction status
   - Handles connection management

4. **Frontend Integration** (React + Viem)
   - Listens to SSE updates
   - Displays transaction progress
   - Handles market creation UI

---

## 📦 Technology Stack

### Backend
- **Fastify** - Web framework for API and SSE
- **Viem 2.21+** - Ethereum client with native ERC-4337 support
- **Viem Account Abstraction** - Built-in Paymaster/Bundler clients
- **fastify-sse-v2** - Server-Sent Events plugin

### Frontend
- **React** - UI framework (Vite)
- **Viem** - Blockchain interactions
- **EventSource** - SSE client (native browser API)

### Blockchain
- **Base Network** - L2 chain
- **Base Paymaster** - Gas sponsorship service
- **ERC-4337 v0.6** - Account Abstraction standard
- **Solady Smart Account** - Smart account implementation

---

## 🔧 Implementation Steps

### Phase 1: Environment Setup

**Tasks:**
- [ ] Install dependencies (`viem@^2.21.0`, `fastify`, `fastify-sse-v2`)
- [ ] Configure Base RPC endpoints (HTTP + WebSocket)
- [ ] Set up Base Paymaster RPC URL
- [ ] Configure EntryPoint v0.6 address
- [ ] Generate/configure backend Smart Account private key
- [ ] Set up environment variables

**Environment Variables Required:**
```bash
BASE_RPC_URL=https://mainnet.base.org
BASE_WEBSOCKET_RPC=wss://base-mainnet.ws.alchemy.com/v2/YOUR_KEY
PAYMASTER_RPC_URL=https://api.developer.coinbase.com/rpc/v1/base/YOUR_KEY
ENTRY_POINT_ADDRESS=0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789
BONDING_CURVE_ADDRESS=0x...
MARKET_FACTORY_ADDRESS=0x...
BACKEND_PRIVATE_KEY=0x...
```

---

### Phase 2: Backend - Paymaster Service

**Tasks:**
- [ ] Create `PaymasterService` class
- [ ] Initialize Public Client (Viem)
- [ ] Create Smart Account using `toSoladySmartAccount`
- [ ] Initialize Paymaster Client
- [ ] Initialize Bundler Client with paymaster configuration
- [ ] Implement `createMarket()` method
  - Encode market creation call data
  - Create UserOperation with paymaster
  - Submit to bundler
  - Return transaction hash
- [ ] Add error handling and retry logic
- [ ] Implement transaction status polling

**Key Code Pattern:**
```javascript
import { createPublicClient, http } from 'viem';
import { 
  createBundlerClient,
  createPaymasterClient,
  toSoladySmartAccount 
} from 'viem/account-abstraction';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

// Setup clients
const publicClient = createPublicClient({ chain: base, transport: http() });
const paymasterClient = createPaymasterClient({ transport: http(PAYMASTER_URL) });
const owner = privateKeyToAccount(BACKEND_PRIVATE_KEY);

// Create Smart Account
const smartAccount = await toSoladySmartAccount({
  client: publicClient,
  owner,
  entryPoint: { address: ENTRY_POINT_ADDRESS, version: '0.6' }
});

// Create Bundler with Paymaster
const bundlerClient = createBundlerClient({
  account: smartAccount,
  client: publicClient,
  paymaster: paymasterClient,
  transport: http(PAYMASTER_URL)
});

// Send sponsored transaction
const hash = await bundlerClient.sendUserOperation({
  calls: [{ to: MARKET_FACTORY, data: callData }]
});
```

---

### Phase 3: Backend - Event Listener Service

**Tasks:**
- [ ] Create `EventListenerService` class
- [ ] Initialize WebSocket transport for Viem Public Client
- [ ] Set up contract event watcher using `publicClient.watchContractEvent`
- [ ] Define event handler for `MarketCreationRequested`
- [ ] Extract event parameters (token address, initial price, etc.)
- [ ] Call PaymasterService to create market
- [ ] Emit SSE updates during processing
- [ ] Handle reconnection and error recovery
- [ ] Log events to database (Supabase)

**Event Watching Pattern:**
```javascript
const unwatch = publicClient.watchContractEvent({
  address: BONDING_CURVE_ADDRESS,
  abi: bondingCurveAbi,
  eventName: 'MarketCreationRequested',
  onLogs: async (logs) => {
    for (const log of logs) {
      const { tokenAddress, initialPrice, creator } = log.args;
      await handleMarketCreation({ tokenAddress, initialPrice, creator });
    }
  },
  onError: (error) => {
    console.error('Event watching error:', error);
  }
});
```

---

### Phase 4: Backend - SSE Service

**Tasks:**
- [ ] Add `fastify-sse-v2` plugin to Fastify app
- [ ] Create SSE endpoint `/api/market-events`
- [ ] Implement connection management (store active connections)
- [ ] Create broadcast function for updates
- [ ] Define message types:
  - `event-detected` - Event received from blockchain
  - `paymaster-processing` - Creating UserOperation
  - `transaction-submitted` - UserOp sent to bundler
  - `transaction-confirmed` - Market created successfully
  - `error` - Error occurred
- [ ] Add heartbeat mechanism (ping every 30s)
- [ ] Handle client disconnections
- [ ] Add authentication/authorization if needed

**SSE Endpoint Pattern:**
```javascript
fastify.get('/api/market-events', async (request, reply) => {
  const connectionId = crypto.randomUUID();
  
  reply.sse(
    (async function* () {
      // Send initial connection message
      yield { 
        event: 'connected',
        data: JSON.stringify({ connectionId, timestamp: Date.now() })
      };
      
      // Keep connection alive with heartbeat
      const heartbeat = setInterval(() => {
        reply.sse.send({ event: 'ping', data: 'alive' });
      }, 30000);
      
      // Store connection for broadcasting
      sseConnections.set(connectionId, reply);
      
      // Wait for connection close
      await new Promise(resolve => {
        reply.raw.on('close', () => {
          clearInterval(heartbeat);
          sseConnections.delete(connectionId);
          resolve();
        });
      });
    })()
  );
});
```

---

### Phase 5: Frontend - SSE Client Integration

**Tasks:**
- [ ] Create `useMarketEvents` React hook
- [ ] Initialize EventSource connection to SSE endpoint
- [ ] Handle incoming SSE messages
- [ ] Update UI based on event type
- [ ] Add connection status indicator
- [ ] Implement reconnection logic
- [ ] Display loading states and progress indicators
- [ ] Show success/error messages
- [ ] Clean up EventSource on unmount

**React Hook Pattern:**
```javascript
function useMarketEvents() {
  const [events, setEvents] = useState([]);
  const [status, setStatus] = useState('connecting');
  
  useEffect(() => {
    const eventSource = new EventSource('/api/market-events');
    
    eventSource.onopen = () => {
      setStatus('connected');
    };
    
    eventSource.addEventListener('event-detected', (e) => {
      const data = JSON.parse(e.data);
      setEvents(prev => [...prev, { type: 'detected', ...data }]);
    });
    
    eventSource.addEventListener('transaction-submitted', (e) => {
      const data = JSON.parse(e.data);
      setEvents(prev => [...prev, { type: 'submitted', ...data }]);
    });
    
    eventSource.addEventListener('transaction-confirmed', (e) => {
      const data = JSON.parse(e.data);
      setEvents(prev => [...prev, { type: 'confirmed', ...data }]);
    });
    
    eventSource.onerror = () => {
      setStatus('error');
      eventSource.close();
    };
    
    return () => eventSource.close();
  }, []);
  
  return { events, status };
}
```

---

### Phase 6: Frontend - UI Components

**Tasks:**
- [ ] Create `MarketCreationProgress` component
- [ ] Display real-time transaction status
- [ ] Show step-by-step progress:
  1. Detecting bonding curve event
  2. Preparing gasless transaction
  3. Submitting to Base Paymaster
  4. Waiting for confirmation
  5. Market created successfully
- [ ] Add loading animations
- [ ] Display transaction hash with Basescan link
- [ ] Show error states with retry option
- [ ] Add toast notifications for key events

---

### Phase 7: Testing & Quality Assurance

**Backend Testing:**
- [ ] Unit tests for PaymasterService
- [ ] Unit tests for EventListenerService
- [ ] Integration tests for SSE broadcasting
- [ ] Test error handling and edge cases
- [ ] Test Smart Account creation and funding
- [ ] Test paymaster UserOperation submission
- [ ] Verify gas sponsorship is working

**Frontend Testing:**
- [ ] Test SSE connection and reconnection
- [ ] Test UI updates with mock events
- [ ] Test error state handling
- [ ] Test on different browsers

**End-to-End Testing:**
- [ ] Deploy to Base testnet (Sepolia)
- [ ] Test full flow: bonding curve → paymaster → market creation
- [ ] Verify transactions appear on Basescan
- [ ] Verify gas is sponsored (user pays nothing)
- [ ] Test with multiple concurrent users
- [ ] Load test SSE endpoint

**Security Testing:**
- [ ] Audit private key management
- [ ] Review Smart Account permissions
- [ ] Test rate limiting on endpoints
- [ ] Verify authentication on SSE endpoint
- [ ] Review error messages (no sensitive data leaks)

---

### Phase 8: Deployment & Monitoring

**Deployment Tasks:**
- [ ] Deploy backend to production server
- [ ] Configure production environment variables
- [ ] Set up Base mainnet RPC endpoints
- [ ] Fund Smart Account with ETH (for deployment only)
- [ ] Deploy bonding curve contract with updated event
- [ ] Verify Paymaster RPC URL is production-ready

**Monitoring:**
- [ ] Set up logging (Winston or Pino with Fastify)
- [ ] Monitor event listener health
- [ ] Track paymaster transaction success rate
- [ ] Monitor SSE connection count
- [ ] Set up alerts for errors/failures
- [ ] Dashboard for transaction analytics

**Documentation:**
- [ ] Document API endpoints
- [ ] Create deployment guide
- [ ] Write troubleshooting guide
- [ ] Document Smart Account management
- [ ] Create runbook for common issues

---

## 🎨 Key Design Decisions

### 1. **Viem Native ERC-4337 Support**
- **Decision:** Use Viem's built-in `account-abstraction` module
- **Rationale:** Eliminates need for Permissionless.js, reduces dependencies, better integration with existing Viem setup
- **Impact:** Simpler codebase, better TypeScript support

### 2. **Solady Smart Account**
- **Decision:** Use `toSoladySmartAccount` for Smart Account implementation
- **Rationale:** Gas-efficient, well-audited, compatible with Base Paymaster
- **Alternative:** `toCoinbaseSmartAccount` (consider for future)

### 3. **EntryPoint v0.6**
- **Decision:** Use EntryPoint v0.6 address
- **Rationale:** Required for Base Paymaster compatibility
- **Address:** `0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789`

### 4. **Backend-Controlled Smart Account**
- **Decision:** Backend owns and controls the Smart Account
- **Rationale:** Simplifies gas sponsorship, backend acts on behalf of users
- **Security:** Backend private key must be securely stored

### 5. **SSE for Real-Time Updates**
- **Decision:** Use Server-Sent Events instead of WebSockets
- **Rationale:** Simpler implementation, one-way communication sufficient, better for serverless environments
- **Implementation:** `fastify-sse-v2` plugin

### 6. **Event-Driven Architecture**
- **Decision:** Watch blockchain events to trigger market creation
- **Rationale:** Decouples bonding curve from market factory, enables gasless flow
- **Trade-off:** Slight delay vs direct contract calls

---

## 📋 Acceptance Criteria

### ✅ Functional Requirements
- [ ] User can trigger bonding curve action (on-chain)
- [ ] System detects `MarketCreationRequested` event within 5 seconds
- [ ] Backend creates gasless transaction using Paymaster
- [ ] Market is created without user paying gas
- [ ] Frontend receives real-time updates via SSE
- [ ] Transaction hash is displayed to user with Basescan link
- [ ] Errors are handled gracefully with retry mechanism

### ✅ Non-Functional Requirements
- [ ] System handles at least 10 concurrent market creations
- [ ] SSE connections remain stable for 5+ minutes
- [ ] Event listener recovers from disconnections automatically
- [ ] Transaction confirmation time < 30 seconds (Base average)
- [ ] 99% success rate for paymaster transactions
- [ ] Backend logs all events for debugging

---

## 🚧 Known Challenges & Mitigations

### Challenge 1: Smart Account Deployment
- **Issue:** Smart Account must be deployed before first use
- **Mitigation:** Auto-deploy on first UserOperation, or pre-deploy during setup

### Challenge 2: Paymaster Rate Limits
- **Issue:** Base Paymaster may have rate limits or policies
- **Mitigation:** Implement retry logic, monitor usage, have fallback mechanism

### Challenge 3: Event Listener Reliability
- **Issue:** WebSocket connections can drop
- **Mitigation:** Implement automatic reconnection, use multiple RPC providers as fallback

### Challenge 4: SSE Connection Management
- **Issue:** Many open connections can consume resources
- **Mitigation:** Implement connection limits, add authentication, clean up stale connections

### Challenge 5: Transaction Finality
- **Issue:** Base has 2-second block times, but confirmation can vary
- **Mitigation:** Wait for transaction receipt, provide clear status updates to user

---

## 📚 Reference Documentation

### External Resources
1. **Base Paymaster Documentation**  
   https://docs.base.org/cookbook/go-gasless  
   Retrieved: November 4, 2025

2. **Viem Account Abstraction**  
   https://viem.sh/account-abstraction  
   Retrieved: November 4, 2025

3. **Viem Bundler Client**  
   https://viem.sh/account-abstraction/clients/bundler  
   Retrieved: November 6, 2025

4. **Viem Paymaster Client**  
   https://viem.sh/account-abstraction/clients/paymaster  
   Retrieved: November 6, 2025

5. **Fastify Documentation**  
   https://fastify.dev/  
   Retrieved: November 4, 2025

6. **FastifySSE Plugin**  
   https://github.com/mpetrunic/fastify-sse-v2  
   Retrieved: November 4, 2025

7. **ERC-4337 Specification**  
   https://eips.ethereum.org/EIPS/eip-4337  
   Retrieved: November 4, 2025

### Project Files
- `/mnt/project/SecondOrder_fun_InfoFi_Integration__Complete_Technical_Specification.md`
- `/mnt/project/SecondOrder_fun_Backend_Architecture_Options___Recommendations.md`
- `/mnt/project/SecondOrder_fun_Frontend_Guidelines.md`

---

## 🎯 Next Steps

1. **Review & Approval**
   - Review this work plan with team
   - Confirm technical approach
   - Identify any missing requirements

2. **Environment Setup**
   - Provision Base RPC endpoints (Alchemy/Infura)
   - Get Base Paymaster API key from Coinbase Developer Platform
   - Set up development environment

3. **Start Implementation**
   - Begin with Phase 1 (Environment Setup)
   - Proceed sequentially through phases
   - Test each phase before moving to next

---

**Document Version:** 1.0  
**Last Updated:** November 6, 2025  
**Status:** Ready for Implementation Review
