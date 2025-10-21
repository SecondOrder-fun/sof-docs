# FPMM Migration Summary

**Date**: 2025-10-21  
**Status**: ✅ Core Contracts Complete | ⚠️ Deployment & Testing Pending

## Overview

Migrated InfoFi prediction markets from Constant Sum Market Maker (CSMM) to Fixed Product Market Maker (FPMM) with Gnosis Conditional Token Framework integration.

## Key Changes

### 1. **New Contracts**

#### `RaffleOracleAdapter.sol`
- Bridges raffle VRF resolution to Gnosis CTF
- Prepares binary conditions for each player: [WIN, LOSE]
- Batch resolution support for gas efficiency
- Replaces UMA oracle dependency

#### `InfoFiFPMMV2.sol`
- **SOLPToken**: SecondOrder Liquidity Provider ERC-20 token
- **SimpleFPMM**: Custom FPMM implementation (x * y = k invariant)
  - Binary outcome markets (YES/NO)
  - 2% trading fee
  - Liquidity provision with SOLP rewards
  - Slippage protection
- **InfoFiFPMMV2**: Manager contract for FPMM markets

#### `InfoFiMarketFactory.sol` (V2)
- Auto-creates FPMM markets at 1% threshold
- Integrates with RaffleOracleAdapter
- Treasury-funded initial liquidity (100 SOF per market)
- Batch market resolution after VRF

#### Interface Wrappers
- `IConditionalTokens.sol`: Solidity 0.8.20 interface for Gnosis CTF
- `IFixedProductMarketMaker.sol`: Interface for FPMM operations

### 2. **Removed/Deprecated**

- ❌ `SeasonCSMM.sol` - Deleted
- ❌ `InfoFiMarketFactory.sol` (V1) - Renamed to `.txt` (deprecated)
- ❌ `test/SeasonCSMM.t.sol` - Renamed to `.deprecated`
- ❌ `test/integration/InfoFiCSMMIntegration.t.sol` - Renamed to `.deprecated`

### 3. **Architecture Changes**

#### Before (CSMM)
```
InfoFiMarketFactory
  └─> SeasonCSMM (per season)
       └─> Player markets (constant sum pricing)
```

#### After (FPMM)
```
InfoFiMarketFactory
  ├─> RaffleOracleAdapter (CTF oracle)
  └─> InfoFiFPMMV2
       └─> SimpleFPMM (per player)
            ├─> SOLP tokens (liquidity shares)
            └─> x * y = k pricing
```

### 4. **Token Economics**

#### SOLP Token
- **Name**: `SOLP-S{seasonId}-{playerAddress}`
- **Symbol**: `SOLP`
- **Purpose**: Represents LP shares in player markets
- **Minting**: Proportional to liquidity provided
- **Burning**: On liquidity removal

#### Trading Fees
- **Fee**: 2% of trade amount
- **Current Split**: 100% to protocol treasury
- **Future**: Upgradeable split between protocol and LPs

#### Initial Liquidity
- **Amount**: 100 SOF per market
- **Source**: Treasury wallet
- **Distribution**: 50 SOF YES reserve, 50 SOF NO reserve
- **LP Tokens**: Minted to treasury

### 5. **Pricing Mechanism**

#### CSMM (Old)
```
P(YES) = YES_reserve / (YES_reserve + NO_reserve)
Linear pricing, no slippage
```

#### FPMM (New)
```
k = YES_reserve * NO_reserve (constant)
P(YES) = NO_reserve / (YES_reserve + NO_reserve)
Non-linear pricing with slippage
```

### 6. **Resolution Flow**

1. **Market Creation** (at 1% threshold)
   - Factory calls `oracleAdapter.preparePlayerCondition()`
   - CTF condition created with questionId = hash(seasonId, player)
   - SimpleFPMM deployed with 100 SOF initial liquidity
   - SOLP token minted to treasury

2. **Trading** (during season)
   - Users call `SimpleFPMM.buy(buyYes, amountIn, minAmountOut)`
   - 2% fee collected
   - Reserves updated via x * y = k
   - Outcome tokens tracked (future: CTF integration)

3. **Resolution** (after VRF)
   - Admin calls `factory.resolveSeasonMarkets(seasonId, winner)`
   - Batch resolution via `oracleAdapter.batchResolveSeasonMarkets()`
   - Payout vector: [1, 0] for winner, [0, 1] for losers
   - CTF conditions marked as resolved

4. **Claims** (future implementation)
   - Winners redeem via CTF `redeemPositions()`
   - Losers get 0 payout
   - LPs can withdraw remaining liquidity

## Deployment Requirements

### New Constructor Parameters

#### `RaffleOracleAdapter`
```solidity
constructor(
    address _conditionalTokens,  // Gnosis CTF contract
    address _admin               // Admin & resolver role
)
```

#### `InfoFiFPMMV2`
```solidity
constructor(
    address _conditionalTokens,  // Gnosis CTF contract
    address _collateralToken,    // SOF token
    address _treasury,           // Protocol treasury
    address _admin               // Admin & factory role
)
```

#### `InfoFiMarketFactory`
```solidity
constructor(
    address _raffle,             // Raffle contract
    address _oracle,             // InfoFiPriceOracle
    address _oracleAdapter,      // RaffleOracleAdapter
    address _fpmmManager,        // InfoFiFPMMV2
    address _sofToken,           // SOF token
    address _treasury,           // Treasury wallet
    address _admin               // Admin role
)
```

### Treasury Setup

**Anvil Testing**:
- Use deployer EOA as treasury
- Pre-fund with sufficient SOF (e.g., 10,000 SOF)

**Production**:
- Use multi-sig wallet (e.g., Gnosis Safe)
- Approve InfoFiMarketFactory to spend SOF
- Monitor treasury balance for market creation

## Testing Status

### ✅ Compilation
- All contracts compile successfully
- No Solidity version conflicts
- Interface wrappers working correctly

### ⚠️ Pending Tests

1. **Unit Tests**
   - `RaffleOracleAdapter.t.sol`
   - `SimpleFPMM.t.sol`
   - `InfoFiFPMMV2.t.sol`
   - `InfoFiMarketFactory.t.sol`

2. **Integration Tests**
   - End-to-end market creation
   - Trading flow with slippage
   - VRF resolution + batch settlement
   - Liquidity provision/removal

3. **Invariant Tests**
   - FPMM k-invariant preservation
   - Fee accounting
   - LP token supply == liquidity

## Frontend Integration

### API Changes

#### Old CSMM Endpoints
```javascript
// DEPRECATED
GET /api/markets/:seasonId/:player/price
POST /api/markets/:seasonId/:player/bet
```

#### New FPMM Endpoints
```javascript
GET /api/fpmm/:seasonId/:player/prices
POST /api/fpmm/:seasonId/:player/buy
POST /api/fpmm/:seasonId/:player/add-liquidity
POST /api/fpmm/:seasonId/:player/remove-liquidity
GET /api/fpmm/:seasonId/:player/lp-balance
```

### Contract Hooks

#### `useInfoFiMarket` (Update Required)
```javascript
// Old: CSMM contract calls
const { placeBet } = useInfoFiMarket()

// New: SimpleFPMM contract calls
const { buy, addLiquidity, removeLiquidity } = useInfoFiFPMM()
```

### New Components Needed

1. **LiquidityProvisionWidget**
   - Add/remove liquidity UI
   - SOLP balance display
   - APY calculator (future)

2. **FPMMPriceChart**
   - Non-linear price curve visualization
   - Slippage indicator
   - Reserve ratio display

3. **SOLPTokenDisplay**
   - LP token balance
   - Share of pool percentage
   - Claimable fees (future)

## Migration Checklist

### Smart Contracts
- [x] Create RaffleOracleAdapter
- [x] Create SimpleFPMM
- [x] Create InfoFiFPMMV2
- [x] Update InfoFiMarketFactory
- [x] Create interface wrappers
- [x] Remove CSMM contracts
- [x] Deprecate old tests
- [ ] Write new unit tests
- [ ] Write integration tests
- [ ] Update Deploy.s.sol
- [ ] Update E2E scripts

### Backend
- [ ] Update market sync service
- [ ] Add FPMM price calculation
- [ ] Add liquidity tracking
- [ ] Update database schema (if needed)
- [ ] Add SOLP token indexing

### Frontend
- [ ] Update useInfoFiMarket hook
- [ ] Create useInfoFiFPMM hook
- [ ] Build LiquidityProvisionWidget
- [ ] Update BuySellWidget for FPMM
- [ ] Add slippage controls
- [ ] Display SOLP balances
- [ ] Update price charts

### Documentation
- [x] Migration summary (this file)
- [ ] Update API documentation
- [ ] Update frontend guidelines
- [ ] Update E2E runbook
- [ ] Add liquidity provision guide

## Rollout Strategy

### Phase 1: Local Testing (Current)
1. Deploy to Anvil
2. Run E2E tests
3. Verify market creation
4. Test trading with slippage
5. Test resolution flow

### Phase 2: Testnet Deployment
1. Deploy to Sepolia/Base Sepolia
2. Fund treasury with test SOF
3. Create test markets
4. Community testing
5. Bug fixes

### Phase 3: Production
1. Deploy to mainnet
2. Set up multi-sig treasury
3. Gradual rollout (monitor gas costs)
4. Enable LP fee split (after monitoring)

## Known Limitations

1. **CTF Integration**: SimpleFPMM doesn't fully integrate with CTF for outcome token minting/burning (future enhancement)
2. **LP Rewards**: Fee split to LPs not yet implemented (infrastructure in place)
3. **Price Oracle**: Hybrid pricing oracle not yet updated for FPMM prices
4. **Slippage**: No automatic slippage calculation in frontend (user must set manually)

## Gas Cost Estimates

| Operation | CSMM (Old) | FPMM (New) | Delta |
|-----------|------------|------------|-------|
| Market Creation | ~200k | ~400k | +100% |
| Place Bet | ~80k | ~120k | +50% |
| Add Liquidity | N/A | ~150k | New |
| Resolution | ~100k | ~150k | +50% |

**Note**: Higher gas costs offset by better price discovery and liquidity provision features.

## Questions & Decisions

### Resolved
- ✅ LP Token Name: **SOLP** (SecondOrder Liquidity Provider)
- ✅ Initial Fee Split: **100% to protocol** (upgradeable later)
- ✅ Initial Liquidity: **100 SOF per market** from treasury
- ✅ Treasury Type: **EOA for testing, multi-sig for production**

### Pending
- ❓ Should we implement complete CTF integration for outcome tokens?
- ❓ When to enable LP fee rewards?
- ❓ Should we add liquidity mining incentives?
- ❓ How to handle market resolution disputes?

## References

- [Gnosis Conditional Tokens](https://github.com/gnosis/conditional-tokens-contracts)
- [FPMM Whitepaper](https://docs.gnosis.io/conditionaltokens/docs/introduction3/)
- [Uniswap V2 (x*y=k)](https://uniswap.org/whitepaper.pdf)
