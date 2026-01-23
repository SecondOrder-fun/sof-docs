# InfoFiPriceOracle Backend Integration Plan

**Status**: Ready for Implementation  
**Date**: Oct 26, 2025  
**Architecture Decision**: Option A (Backend calls oracle on-chain)  
**Signer**: Anvil account[0] (local), Base Paymaster (production)  
**Update Strategy**: Real-time with retry logic  
**Error Handling**: Retry with exponential backoff → Alert admin on cutoff

---

## Executive Summary

This plan integrates the `InfoFiPriceOracle` smart contract into the backend event listener system. The backend will:

1. **Listen for PositionUpdate events** → Call `updateRaffleProbability()` on oracle
2. **Listen for bet placement** → Call `updateMarketSentiment()` on oracle
3. **Retry failed calls** with exponential backoff (up to 5 attempts)
4. **Alert admin** if oracle calls fail after max retries

This enables real-time hybrid pricing updates (70% raffle + 30% market sentiment) for the InfoFi prediction markets.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│ Smart Contracts (On-Chain)                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │ SOFBondingCurve  │  │ InfoFiMarket     │                │
│  │  (emits events)  │  │  (emits events)  │                │
│  └────────┬─────────┘  └────────┬─────────┘                │
│           │                     │                           │
│           └─────────────────────┘                           │
│                     │                                        │
│              PositionUpdate                                 │
│              BetPlaced                                      │
│                     │                                        │
└─────────────────────┼────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Backend Event Listeners (Node.js)                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ positionUpdateListener.js (ENHANCED)                 │  │
│  │ - Listens for PositionUpdate events                  │  │
│  │ - Calls oracle.updateRaffleProbability()             │  │
│  │ - Implements retry logic with exponential backoff    │  │
│  │ - Alerts admin on failure cutoff                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ betPlacedListener.js (NEW)                           │  │
│  │ - Listens for BetPlaced events from InfoFiMarket     │  │
│  │ - Calls oracle.updateMarketSentiment()               │  │
│  │ - Implements retry logic with exponential backoff    │  │
│  │ - Alerts admin on failure cutoff                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ oracleCallService.js (NEW)                           │  │
│  │ - Centralized oracle call logic                       │  │
│  │ - Retry mechanism with exponential backoff           │  │
│  │ - Error handling and admin alerts                    │  │
│  │ - Transaction receipt tracking                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Smart Contracts (On-Chain)                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ InfoFiPriceOracle                                    │  │
│  │ - Stores hybrid pricing data                         │  │
│  │ - Calculates: (70% × raffle + 30% × market) / 100   │  │
│  │ - Emits PriceUpdated events                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Phases

### Phase 1: Foundation Setup (Week 1)

#### 1.1 Create Oracle Call Service
**File**: `backend/src/services/oracleCallService.js`

**Responsibilities**:
- Centralized logic for calling oracle functions
- Retry mechanism with exponential backoff
- Error handling and logging
- Admin alert system
- Transaction receipt tracking

**Key Features**:
```javascript
- updateRaffleProbability(fpmmAddress, raffleProbabilityBps)
- updateMarketSentiment(fpmmAddress, marketSentimentBps)
- Retry config: maxAttempts=5, baseDelay=1000ms, maxDelay=30000ms
- Cutoff: Alert admin after 3 failed retries
- Fallback: Continue without oracle update (graceful degradation)
- fpmmAddress is the market ID (SimpleFPMM contract address)
```

#### 1.2 Create Wallet Client Configuration
**File**: `backend/src/lib/walletClient.js`

**Responsibilities**:
- Create wallet client with backend signer (Anvil account[0])
- Configure for local testing (Anvil) and production (Base Paymaster)
- Handle account creation from private key
- Manage gas estimation and transaction parameters

**Key Features**:
```javascript
- Local: Use BACKEND_WALLET_PRIVATE_KEY from .env (account with ETH)
- Production: Use BASE_PAYMASTER_ADDRESS + BACKEND_PRIVATE_KEY
- Support both HTTP and WebSocket transports
- Implement gas price estimation
```

#### 1.3 Update Viem Client Configuration
**File**: `backend/src/lib/viemClient.js` (ENHANCE)

**Changes**:
- Add wallet client export alongside public client
- Configure for both read and write operations
- Add chain configuration for Base/Anvil

---

### Phase 2: Event Listener Enhancement (Week 1-2)

#### 2.1 Enhance positionUpdateListener
**File**: `backend/src/listeners/positionUpdateListener.js` (MODIFY)

**New Responsibilities**:
- After updating database probabilities, call oracle
- Call `oracle.updateRaffleProbability()` for each player
- Implement retry logic via oracleCallService
- Log oracle call results
- Handle oracle failures gracefully

**New Code Section**:
```javascript
// After updating database probabilities:
for (const { player, ticketCount } of playerPositions) {
  const newBps = Math.round((ticketCount * 10000) / totalTicketsNum);
  
  try {
    await oracleCallService.updateRaffleProbability(
      marketId,
      newBps,
      logger
    );
  } catch (error) {
    logger.error(`Failed to update oracle for ${player}: ${error.message}`);
    // Continue - don't crash listener
  }
}
```

#### 2.2 Create betPlacedListener
**File**: `backend/src/listeners/betPlacedListener.js` (NEW)

**Responsibilities**:
- Listen for BetPlaced events from InfoFiMarket contracts
- Extract market ID and sentiment data from event
- Call `oracle.updateMarketSentiment()`
- Implement retry logic
- Handle errors gracefully

**Event Structure**:
```solidity
event BetPlaced(
  uint256 indexed marketId,
  address indexed bettor,
  bool outcome,  // true = YES, false = NO
  uint256 amount,
  uint256 newSentimentBps
);
```

**Implementation**:
```javascript
export async function startBetPlacedListener(
  infoFiMarketAddress,
  infoFiMarketAbi,
  oracleAddress,
  oracleAbi,
  logger
) {
  // Watch for BetPlaced events
  // Call oracleCallService.updateMarketSentiment()
  // Implement retry + error handling
}
```

---

### Phase 3: Database & Monitoring (Week 2)

#### 3.1 Add Oracle Call Tracking Table
**File**: Database migration

**Table**: `oracle_call_history`

**Columns**:
```sql
- id (BIGSERIAL PRIMARY KEY)
- market_id (BIGINT, FK to infofi_markets)
- call_type (VARCHAR: 'updateRaffleProbability' | 'updateMarketSentiment')
- parameters (JSONB: {marketId, value, ...})
- status (VARCHAR: 'pending' | 'success' | 'failed' | 'retrying')
- attempt_count (INTEGER)
- transaction_hash (VARCHAR, nullable)
- error_message (TEXT, nullable)
- created_at (TIMESTAMPTZ)
- updated_at (TIMESTAMPTZ)
```

#### 3.2 Add Admin Alert System
**File**: `backend/src/services/adminAlertService.js` (NEW)

**Responsibilities**:
- Send alerts when oracle calls fail after max retries
- Track alert history to avoid spam
- Support multiple notification channels (email, Discord, etc.)

**Alert Triggers**:
- Oracle call fails after 3 retries
- Wallet client initialization fails
- Gas estimation fails repeatedly

---

### Phase 4: Integration & Testing (Week 2-3)

#### 4.1 Update Server Initialization
**File**: `backend/fastify/server.js` (MODIFY)

**Changes**:
```javascript
// Import new services
import { startBetPlacedListener } from '../src/listeners/betPlacedListener.js';
import { oracleCallService } from '../src/services/oracleCallService.js';

// Initialize wallet client
import { walletClient } from '../src/lib/walletClient.js';

// In startListeners():
// Start betPlacedListener for each active market
// Pass oracle address and ABI to listeners
```

#### 4.2 Create Integration Tests
**File**: `tests/integration/oracleIntegration.test.js` (NEW)

**Test Cases**:
```javascript
- ✅ Oracle call succeeds on first attempt
- ✅ Oracle call retries on failure and succeeds
- ✅ Oracle call fails after max retries and alerts admin
- ✅ Multiple concurrent oracle calls are handled correctly
- ✅ Position updates trigger oracle calls for all players
- ✅ Bet placement triggers oracle sentiment updates
- ✅ Transaction receipts are tracked correctly
- ✅ Graceful degradation when oracle is unavailable
```

#### 4.3 Create Unit Tests
**File**: `tests/services/oracleCallService.test.js` (NEW)

**Test Cases**:
```javascript
- ✅ Retry logic with exponential backoff
- ✅ Max attempts cutoff
- ✅ Error message formatting
- ✅ Transaction hash extraction
- ✅ Admin alert triggering
```

---

## File Structure

```
backend/
├── src/
│   ├── lib/
│   │   ├── viemClient.js (ENHANCE)
│   │   └── walletClient.js (NEW)
│   ├── listeners/
│   │   ├── positionUpdateListener.js (MODIFY)
│   │   ├── betPlacedListener.js (NEW)
│   │   └── marketCreatedListener.js (existing)
│   ├── services/
│   │   ├── oracleCallService.js (NEW)
│   │   └── adminAlertService.js (NEW)
│   └── abis/
│       ├── InfoFiPriceOracleAbi.js (existing)
│       └── InfoFiMarketAbi.js (existing)
├── fastify/
│   └── server.js (MODIFY)
└── shared/
    └── supabaseClient.js (existing)

tests/
├── integration/
│   └── oracleIntegration.test.js (NEW)
└── services/
    └── oracleCallService.test.js (NEW)

migrations/
└── add_oracle_call_history_table.sql (NEW)
```

---

## Environment Variables

### Local Development (Anvil)

```bash
# Wallet Configuration
BACKEND_WALLET_PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80  # account with ETH
BACKEND_SIGNER_TYPE=local  # 'local' or 'paymaster'

# Oracle Configuration
INFOFI_ORACLE_ADDRESS_LOCAL=0x...  # Deployed oracle address
INFOFI_MARKET_ADDRESS_LOCAL=0x...  # InfoFiMarket contract address

# Retry Configuration
ORACLE_MAX_RETRIES=5
ORACLE_RETRY_BASE_DELAY_MS=1000
ORACLE_RETRY_MAX_DELAY_MS=30000
ORACLE_ALERT_CUTOFF=3  # Alert admin after 3 failed retries

# Admin Alerts
ADMIN_ALERT_EMAIL=admin@secondorder.fun
ADMIN_ALERT_DISCORD_WEBHOOK=https://discord.com/api/webhooks/...
```

### Production (Base Mainnet)

```bash
# Wallet Configuration
BACKEND_PRIVATE_KEY=0x...  # Secure backend signer
BACKEND_SIGNER_TYPE=paymaster  # Use Base Paymaster
BASE_PAYMASTER_ADDRESS=0x...  # Base Paymaster contract
BASE_PAYMASTER_API_KEY=...  # Paymaster API credentials

# Oracle Configuration
INFOFI_ORACLE_ADDRESS=0x...  # Deployed oracle address
INFOFI_MARKET_ADDRESS=0x...  # InfoFiMarket contract address

# Retry Configuration
ORACLE_MAX_RETRIES=5
ORACLE_RETRY_BASE_DELAY_MS=2000
ORACLE_RETRY_MAX_DELAY_MS=60000
ORACLE_ALERT_CUTOFF=3

# Admin Alerts
ADMIN_ALERT_EMAIL=admin@secondorder.fun
ADMIN_ALERT_DISCORD_WEBHOOK=https://discord.com/api/webhooks/...
```

---

## Implementation Details

### oracleCallService.js Structure

```javascript
export class OracleCallService {
  constructor(walletClient, publicClient, oracleAbi, logger) {
    this.walletClient = walletClient;
    this.publicClient = publicClient;
    this.oracleAbi = oracleAbi;
    this.logger = logger;
  }

  async updateRaffleProbability(marketId, raffleProbabilityBps, logger) {
    return this._callOracleWithRetry(
      'updateRaffleProbability',
      [marketId, raffleProbabilityBps],
      logger
    );
  }

  async updateMarketSentiment(marketId, marketSentimentBps, logger) {
    return this._callOracleWithRetry(
      'updateMarketSentiment',
      [marketId, marketSentimentBps],
      logger
    );
  }

  async _callOracleWithRetry(functionName, args, logger) {
    let lastError;
    
    for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
      try {
        const hash = await this.walletClient.writeContract({
          address: ORACLE_ADDRESS,
          abi: this.oracleAbi,
          functionName: functionName,
          args: args
        });

        // Wait for receipt
        const receipt = await this.publicClient.waitForTransactionReceipt({ hash });
        
        logger.info(`✅ Oracle call succeeded: ${functionName}(${args})`);
        return { success: true, hash, receipt };

      } catch (error) {
        lastError = error;
        
        if (attempt < MAX_RETRIES) {
          const delay = this._calculateBackoffDelay(attempt);
          logger.warn(
            `⚠️  Oracle call failed (attempt ${attempt}/${MAX_RETRIES}): ${error.message}. ` +
            `Retrying in ${delay}ms...`
          );
          await this._sleep(delay);
        }
      }
    }

    // All retries exhausted
    logger.error(`❌ Oracle call failed after ${MAX_RETRIES} attempts: ${lastError.message}`);
    
    // Alert admin
    await adminAlertService.sendAlert({
      severity: 'critical',
      title: `Oracle Call Failed: ${functionName}`,
      message: `Failed to call oracle.${functionName}(${args}) after ${MAX_RETRIES} retries`,
      error: lastError
    });

    throw lastError;
  }

  _calculateBackoffDelay(attempt) {
    const baseDelay = BASE_DELAY_MS;
    const maxDelay = MAX_DELAY_MS;
    const exponentialDelay = baseDelay * Math.pow(2, attempt - 1);
    return Math.min(exponentialDelay, maxDelay);
  }

  async _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### betPlacedListener.js Structure

```javascript
export async function startBetPlacedListener(
  infoFiMarketAddress,
  infoFiMarketAbi,
  oracleCallService,
  logger
) {
  const unwatch = publicClient.watchContractEvent({
    address: infoFiMarketAddress,
    abi: infoFiMarketAbi,
    eventName: 'BetPlaced',
    onLogs: async (logs) => {
      for (const log of logs) {
        const { marketId, bettor, outcome, amount, newSentimentBps } = log.args;

        try {
          logger.info(
            `📊 BetPlaced Event: Market ${marketId}, ` +
            `Bettor ${bettor}, Outcome ${outcome}, Amount ${amount}`
          );

          // Call oracle to update sentiment
          await oracleCallService.updateMarketSentiment(
            marketId,
            newSentimentBps,
            logger
          );

          // Store in database
          await db.recordBetPlaced({
            marketId,
            bettor,
            outcome,
            amount,
            sentimentBps: newSentimentBps,
            transactionHash: log.transactionHash
          });

        } catch (error) {
          logger.error(`Failed to process BetPlaced event: ${error.message}`);
          // Continue listening - don't crash
        }
      }
    }
  });

  return unwatch;
}
```

---

## Deployment Checklist

- [ ] **Environment Setup**
  - [ ] Create `.env` with all required variables
  - [ ] Verify wallet client initialization
  - [ ] Test oracle address is correct

- [ ] **Database**
  - [ ] Run migration for `oracle_call_history` table
  - [ ] Verify table structure

- [ ] **Smart Contracts**
  - [ ] Deploy InfoFiPriceOracle
  - [ ] Deploy/update InfoFiMarket with BetPlaced events
  - [ ] Verify oracle address in factory

- [ ] **Backend Services**
  - [ ] Create oracleCallService
  - [ ] Create walletClient
  - [ ] Create adminAlertService
  - [ ] Update positionUpdateListener
  - [ ] Create betPlacedListener
  - [ ] Update server.js initialization

- [ ] **Testing**
  - [ ] Unit tests pass
  - [ ] Integration tests pass
  - [ ] Local Anvil testing successful
  - [ ] Retry logic verified
  - [ ] Admin alerts working

- [ ] **Monitoring**
  - [ ] Oracle call history being tracked
  - [ ] Logs show oracle updates
  - [ ] Admin alerts configured

- [ ] **Documentation**
  - [ ] Update README with oracle integration
  - [ ] Document retry strategy
  - [ ] Document admin alert system

---

## Success Criteria

✅ **Real-Time Updates**: Oracle is called within 5 seconds of event emission  
✅ **Reliability**: 99%+ oracle call success rate (with retries)  
✅ **Error Handling**: Failed calls alert admin without crashing backend  
✅ **Data Accuracy**: All player probabilities updated atomically  
✅ **Performance**: <500ms overhead per oracle call  
✅ **Monitoring**: All oracle calls tracked in database  
✅ **Graceful Degradation**: Backend continues if oracle unavailable  

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Oracle contract out of gas | Estimate gas before calling, use paymaster |
| Network congestion | Exponential backoff with max delay cap |
| Wallet account issues | Validate account on startup, test with Anvil |
| Admin alert spam | Implement alert deduplication and rate limiting |
| Database connection loss | Retry database writes, log failures |
| Concurrent oracle calls | Use queue system if needed (future optimization) |

---

## Future Optimizations

1. **Queue System**: Batch oracle calls to reduce transaction count
2. **Gas Optimization**: Use multicall for multiple updates
3. **Caching**: Cache oracle prices locally to reduce reads
4. **Monitoring Dashboard**: Real-time oracle call metrics
5. **Fallback Oracle**: Secondary oracle for redundancy
6. **Cross-Chain**: Support multiple chains with different oracles

