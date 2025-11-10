# Base Paymaster Integration - Implementation Plan

**Project:** SecondOrder.fun - Decoupling Bonding Curve from InfoFi Market Creation  
**Date:** November 6, 2025  
**Status:** Ready for Review & Approval

---

## Executive Summary

Decouple the bonding curve from InfoFi market creation by using Base Paymaster for gasless transactions. Backend listens for `PositionUpdate` events and sponsors market creation via ERC-4337.

### Key Benefits

✅ **Decoupled Architecture** - No direct contract-to-contract calls  
✅ **Gasless Transactions** - Base Paymaster sponsors up to $15k/month  
✅ **Better Error Handling** - Backend retry logic with exponential backoff  
✅ **Real-Time Updates** - SSE streams to frontend  
✅ **Reduced Gas Costs** - Eliminates 800K gas forwarding  

---

## Current vs Proposed Architecture

### Current Flow (Direct Call)

```text
User → BondingCurve → Raffle.recordParticipant() 
  → IInfoFiMarketFactory.onPositionUpdate() [800K gas forwarded]
  → Market created (if threshold crossed)
```

### Proposed Flow (Event-Driven + Paymaster)

```text
User → BondingCurve → Raffle.recordParticipant() 
  → Emits PositionUpdate event
Backend Listener → Detects threshold crossing
  → PaymasterService creates UserOperation
  → Base Paymaster sponsors gas
  → IInfoFiMarketFactory.onPositionUpdate() [gasless]
  → Market created
Frontend ← SSE updates
```

---

## Implementation Phases

### Phase 1: Smart Contract Changes

#### 1.1 Modify Raffle.sol

**File:** `contracts/src/core/Raffle.sol`

**Changes:**
- ❌ Remove lines 237-254: Direct InfoFi factory calls
- ❌ Remove lines 290-300: Direct InfoFi calls in removeParticipant
- ❌ Remove line 44: `address public infoFiFactory;`
- ❌ Remove lines 56-58: `InfoFiSkippedInsufficientGas` event
- ❌ Remove lines 83-86: `setInfoFiFactory()` function
- ✅ Keep line 271: `emit PositionUpdate(...)` - Backend listens to this

**Result:** Reduces gas by ~100K per ticket purchase

#### 1.2 Update InfoFiMarketFactory.sol

**File:** `contracts/src/infofi/InfoFiMarketFactory.sol`

**Changes:**
```solidity
// Add new role
bytes32 public constant PAYMASTER_ROLE = keccak256("PAYMASTER_ROLE");

// Update access control
function onPositionUpdate(...) external onlyRole(PAYMASTER_ROLE) {
    // existing logic
}

// Add admin function
function setPaymasterAccount(address account) external onlyRole(DEFAULT_ADMIN_ROLE) {
    grantRole(PAYMASTER_ROLE, account);
}
```

#### 1.3 Update Deployment Scripts

Grant `PAYMASTER_ROLE` to backend Smart Account address after deployment.

---

### Phase 2: Environment Setup

#### 2.1 Install Dependencies

```bash
npm install viem@^2.21.0 fastify-sse-v2 --save
```

#### 2.2 Add Environment Variables

**File:** `.env`

```bash
# Base Paymaster Configuration
BASE_RPC_URL=https://mainnet.base.org
BASE_WEBSOCKET_RPC=wss://base-mainnet.ws.alchemy.com/v2/YOUR_KEY
PAYMASTER_RPC_URL=https://api.developer.coinbase.com/rpc/v1/base/YOUR_CDP_KEY
ENTRY_POINT_ADDRESS=0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789
BACKEND_SMART_ACCOUNT_KEY=0x...

# Testnet
BASE_SEPOLIA_RPC_URL=https://sepolia.base.org
PAYMASTER_RPC_URL_TESTNET=https://api.developer.coinbase.com/rpc/v1/base-sepolia/YOUR_CDP_KEY
```

---

### Phase 3: Backend Services

#### 3.1 Create PaymasterService

**File:** `backend/src/services/paymasterService.js` (NEW)

**Key Methods:**
- `initialize()` - Creates Smart Account using Viem Account Abstraction
- `createMarket(params, logger)` - Submits gasless UserOperation
- `getSmartAccountAddress()` - Returns Smart Account address

**Stack:**
- `viem/account-abstraction` - Native ERC-4337 support
- `toSoladySmartAccount` - Gas-efficient Smart Account
- `createBundlerClient` - Submits UserOperations
- `createPaymasterClient` - Sponsors gas

#### 3.2 Create SSEService

**File:** `backend/src/services/sseService.js` (NEW)

**Key Methods:**
- `addConnection(id, reply)` - Register SSE connection
- `removeConnection(id)` - Clean up connection
- `broadcast(message)` - Send to all clients

#### 3.3 Enhance positionUpdateListener.js

**File:** `backend/src/listeners/positionUpdateListener.js`

**New Logic:**
```javascript
// Calculate probability
const newProbabilityBps = Math.floor((newTickets / totalTickets) * 10000);

// Check threshold (1% = 100 bps)
if (oldProbabilityBps < 100 && newProbabilityBps >= 100) {
  // Emit SSE: market-creation-started
  sseService.broadcast({ event: 'market-creation-started', data: {...} });
  
  // Call Paymaster
  const result = await paymasterService.createMarket({...}, logger);
  
  if (result.success) {
    // Emit SSE: market-creation-confirmed
    // Update database
  } else {
    // Emit SSE: market-creation-failed
  }
}
```

#### 3.4 Create SSE Routes

**File:** `backend/fastify/routes/sseRoutes.js` (NEW)

**Endpoints:**
- `GET /api/market-events` - SSE stream
- `GET /api/market-events/health` - Connection count

#### 3.5 Update server.js

```javascript
import fastifySSE from 'fastify-sse-v2';
import sseRoutes from './routes/sseRoutes.js';

await app.register(fastifySSE);
await app.register(sseRoutes, { prefix: '/api' });
```

---

### Phase 4: Frontend Integration

#### 4.1 Create useMarketEvents Hook

**File:** `src/hooks/useMarketEvents.js` (NEW)

```javascript
export function useMarketEvents() {
  const [events, setEvents] = useState([]);
  const [status, setStatus] = useState('connecting');
  
  useEffect(() => {
    const eventSource = new EventSource('/api/market-events');
    
    eventSource.addEventListener('market-creation-started', (e) => {
      // Add to events
    });
    
    eventSource.addEventListener('market-creation-confirmed', (e) => {
      // Add to events
    });
    
    return () => eventSource.close();
  }, []);
  
  return { events, status };
}
```

#### 4.2 Create MarketCreationProgress Component

**File:** `src/components/MarketCreationProgress.jsx` (NEW)

Displays real-time market creation progress with icons and Basescan links.

---

### Phase 5: Testing

#### 5.1 Unit Tests
- PaymasterService initialization
- Smart Account creation
- UserOperation submission
- SSE broadcasting

#### 5.2 Integration Tests
- Full flow: ticket purchase → market creation
- Error handling: Paymaster failures
- Concurrent users

#### 5.3 Testnet Deployment
1. Deploy to Base Sepolia
2. Configure Paymaster on Coinbase CDP
3. Test with real transactions
4. Verify gas sponsorship working

---

## Critical Decisions - APPROVED ✅

### 1. Smart Account Funding
**Decision:** Use Paymaster for deployment  
**Implementation:** First UserOperation will deploy Smart Account via Paymaster sponsorship

### 2. Paymaster Policies
**Decision:** No limits (one market per player per season)  
**Rationale:** Binary market structure means only one market creation per player threshold crossing  
**Implementation:** No rate limiting needed in Paymaster configuration

### 3. Retry Logic
**Decision:** 3 retries with exponential backoff  
**Implementation:**
- Retry 1: 5 seconds
- Retry 2: 15 seconds
- Retry 3: 45 seconds
- Max 3 retries, then log error and continue

### 4. Backward Compatibility
**Decision:** Not required  
**Implementation:** Can deploy new contracts without supporting old seasons

---

## Deployment Checklist

### Pre-Deployment

- [ ] Generate new private key for Smart Account
- [ ] Get Coinbase CDP API key
- [ ] Test on Base Sepolia (Paymaster will deploy account on first UserOp)

### Contract Deployment
- [ ] Deploy updated Raffle.sol (without InfoFi calls)
- [ ] Deploy updated InfoFiMarketFactory.sol (with PAYMASTER_ROLE)
- [ ] Grant PAYMASTER_ROLE to Smart Account address
- [ ] Verify contracts on Basescan

### Backend Deployment
- [ ] Deploy PaymasterService
- [ ] Deploy SSEService
- [ ] Update positionUpdateListener
- [ ] Deploy SSE routes
- [ ] Test SSE connections

### Frontend Deployment
- [ ] Deploy useMarketEvents hook
- [ ] Deploy MarketCreationProgress component
- [ ] Test real-time updates

### Post-Deployment
- [ ] Monitor Paymaster usage
- [ ] Check transaction success rate
- [ ] Verify SSE connections stable
- [ ] Monitor database sync

---

## Success Metrics

- ✅ 99%+ UserOperation success rate
- ✅ <30 second market creation time
- ✅ 100% database sync accuracy
- ✅ SSE connections stable for 5+ minutes
- ✅ Zero gas cost to users
- ✅ <$50/day Paymaster costs

---

## Rollback Plan

If issues arise:

1. **Immediate:** Stop backend listener
2. **Short-term:** Revert to direct contract calls (redeploy old Raffle.sol)
3. **Long-term:** Fix issues, test on testnet, redeploy

---

## Next Steps

1. **Review this plan** - Confirm approach and ask questions
2. **Approve for implementation** - Get team sign-off
3. **Start Phase 1** - Smart contract modifications
4. **Proceed sequentially** - Complete each phase before next

---

**Ready for Review** ✅
