# Backend FPMM Integration Guide

**Date**: 2025-10-21  
**Status**: ✅ Complete

## Overview

Backend integration for FPMM-based InfoFi markets, providing REST API endpoints for market data, pricing calculations, and liquidity operations.

## Architecture

```
Frontend → API Routes → FPMM Service → Viem → Blockchain
                ↓
            Supabase (future: market indexing)
```

## New Backend Components

### 1. **FPMM Service** (`backend/shared/fpmmService.js`)

Core service for interacting with FPMM contracts using Viem.

#### Key Functions

**Market Data**
- `getMarketAddress(fpmmManagerAddress, seasonId, playerAddress)` - Get FPMM contract address
- `getLpTokenAddress(fpmmManagerAddress, seasonId, playerAddress)` - Get SOLP token address
- `getCompleteMarketData(fpmmManagerAddress, seasonId, playerAddress)` - Get all market data in one call

**Pricing**
- `getMarketPrices(fpmmAddress)` - Get current YES/NO prices (in bps)
- `getMarketReserves(fpmmAddress)` - Get YES/NO reserve amounts
- `calcBuyAmount(fpmmAddress, buyYes, amountIn)` - Calculate output for trade
- `calculatePriceImpact(fpmmAddress, buyYes, amountIn)` - Calculate slippage

**Liquidity**
- `getLpTokenBalance(lpTokenAddress, userAddress)` - Get SOLP balance
- `getLpTokenTotalSupply(lpTokenAddress)` - Get total SOLP supply
- `getUserLpPosition(lpTokenAddress, userAddress, fpmmAddress)` - Get LP position details

#### Configuration

```javascript
// Environment variables required
CHAIN_ID=8453                    // Base mainnet (or 84532 for Sepolia)
RPC_URL=http://127.0.0.1:8545   // RPC endpoint
INFOFI_FPMM_MANAGER_ADDRESS=0x... // InfoFiFPMMV2 contract address
```

### 2. **API Routes** (`backend/fastify/routes/fpmmRoutes.js`)

RESTful endpoints for FPMM operations.

#### Endpoints

##### GET `/api/fpmm/market/:seasonId/:playerAddress`
Get complete market data for a player.

**Response:**
```json
{
  "exists": true,
  "seasonId": 1,
  "playerAddress": "0x...",
  "marketAddress": "0x...",
  "lpTokenAddress": "0x...",
  "prices": {
    "yesPrice": 5500,
    "noPrice": 4500
  },
  "reserves": {
    "yesReserve": "45.5",
    "noReserve": "54.5"
  },
  "lpTotalSupply": "100.0",
  "totalLiquidity": "100.0"
}
```

##### GET `/api/fpmm/prices/:seasonId/:playerAddress`
Get current market prices.

**Response:**
```json
{
  "seasonId": 1,
  "playerAddress": "0x...",
  "marketAddress": "0x...",
  "yesPrice": 5500,
  "noPrice": 4500
}
```

##### GET `/api/fpmm/reserves/:seasonId/:playerAddress`
Get market reserves.

**Response:**
```json
{
  "seasonId": 1,
  "playerAddress": "0x...",
  "marketAddress": "0x...",
  "yesReserve": "45.5",
  "noReserve": "54.5"
}
```

##### POST `/api/fpmm/calculate-buy`
Calculate trade output and price impact.

**Request Body:**
```json
{
  "seasonId": 1,
  "playerAddress": "0x...",
  "buyYes": true,
  "amountIn": "10"
}
```

**Response:**
```json
{
  "seasonId": 1,
  "playerAddress": "0x...",
  "marketAddress": "0x...",
  "buyYes": true,
  "amountIn": "10",
  "amountOut": "9.5",
  "priceImpact": "2.15",
  "expectedPrice": "5612",
  "currentPrice": 5500
}
```

##### GET `/api/fpmm/lp-position/:seasonId/:playerAddress/:userAddress`
Get user's LP position for a market.

**Response:**
```json
{
  "seasonId": 1,
  "playerAddress": "0x...",
  "userAddress": "0x...",
  "marketAddress": "0x...",
  "lpTokenAddress": "0x...",
  "lpTokenBalance": "5.0",
  "shareOfPool": "5.00",
  "claimableSOF": "5.0"
}
```

##### GET `/api/fpmm/lp-balance/:lpTokenAddress/:userAddress`
Get user's SOLP token balance.

**Response:**
```json
{
  "lpTokenAddress": "0x...",
  "userAddress": "0x...",
  "balance": "5.0"
}
```

##### GET `/api/fpmm/health`
Health check endpoint.

**Response:**
```json
{
  "status": "ok",
  "service": "fpmm",
  "fpmmManagerConfigured": true
}
```

## Frontend Integration

### API Client Example

```javascript
// Get market data
const response = await fetch(
  `/api/fpmm/market/${seasonId}/${playerAddress}`
);
const marketData = await response.json();

// Calculate trade
const response = await fetch('/api/fpmm/calculate-buy', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    seasonId: 1,
    playerAddress: '0x...',
    buyYes: true,
    amountIn: '10'
  })
});
const tradeData = await response.json();

// Get LP position
const response = await fetch(
  `/api/fpmm/lp-position/${seasonId}/${playerAddress}/${userAddress}`
);
const lpPosition = await response.json();
```

### React Query Hooks (To Be Created)

```javascript
// useMarketData.js
export function useMarketData(seasonId, playerAddress) {
  return useQuery({
    queryKey: ['fpmm', 'market', seasonId, playerAddress],
    queryFn: () => fetchMarketData(seasonId, playerAddress),
    refetchInterval: 10000, // Refresh every 10s
  });
}

// useCalculateTrade.js
export function useCalculateTrade() {
  return useMutation({
    mutationFn: (params) => calculateTrade(params),
  });
}

// useLpPosition.js
export function useLpPosition(seasonId, playerAddress, userAddress) {
  return useQuery({
    queryKey: ['fpmm', 'lp', seasonId, playerAddress, userAddress],
    queryFn: () => fetchLpPosition(seasonId, playerAddress, userAddress),
    enabled: !!userAddress,
  });
}
```

## Database Integration (Future)

### Proposed Schema Updates

```sql
-- FPMM Markets table
CREATE TABLE fpmm_markets (
  id SERIAL PRIMARY KEY,
  season_id INTEGER REFERENCES raffles(id),
  player_id INTEGER REFERENCES players(id),
  market_address VARCHAR(42) NOT NULL,
  lp_token_address VARCHAR(42) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(season_id, player_id)
);

-- FPMM Price History
CREATE TABLE fpmm_price_history (
  id SERIAL PRIMARY KEY,
  market_id INTEGER REFERENCES fpmm_markets(id),
  yes_price INTEGER NOT NULL, -- basis points
  no_price INTEGER NOT NULL,  -- basis points
  yes_reserve DECIMAL NOT NULL,
  no_reserve DECIMAL NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- FPMM Trades
CREATE TABLE fpmm_trades (
  id SERIAL PRIMARY KEY,
  market_id INTEGER REFERENCES fpmm_markets(id),
  trader_address VARCHAR(42) NOT NULL,
  buy_yes BOOLEAN NOT NULL,
  amount_in DECIMAL NOT NULL,
  amount_out DECIMAL NOT NULL,
  price_impact DECIMAL,
  tx_hash VARCHAR(66) NOT NULL,
  block_number BIGINT NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

-- LP Positions
CREATE TABLE fpmm_lp_positions (
  id SERIAL PRIMARY KEY,
  market_id INTEGER REFERENCES fpmm_markets(id),
  provider_address VARCHAR(42) NOT NULL,
  lp_token_balance DECIMAL NOT NULL,
  last_updated TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(market_id, provider_address)
);
```

## Testing

### Manual Testing Checklist

1. **Start Backend**
   ```bash
   cd backend/fastify
   node server.js
   ```

2. **Test Health Endpoint**
   ```bash
   curl http://localhost:3000/api/fpmm/health
   ```

3. **Test Market Data** (after deploying contracts)
   ```bash
   curl http://localhost:3000/api/fpmm/market/1/0xYourPlayerAddress
   ```

4. **Test Price Calculation**
   ```bash
   curl -X POST http://localhost:3000/api/fpmm/calculate-buy \
     -H "Content-Type: application/json" \
     -d '{
       "seasonId": 1,
       "playerAddress": "0x...",
       "buyYes": true,
       "amountIn": "10"
     }'
   ```

### Integration Tests (To Be Created)

```javascript
// tests/api/fpmmRoutes.test.js
describe('FPMM API Routes', () => {
  test('GET /api/fpmm/market/:seasonId/:playerAddress', async () => {
    const response = await request(app)
      .get('/api/fpmm/market/1/0x...')
      .expect(200);
    
    expect(response.body).toHaveProperty('exists');
    expect(response.body).toHaveProperty('prices');
  });

  test('POST /api/fpmm/calculate-buy', async () => {
    const response = await request(app)
      .post('/api/fpmm/calculate-buy')
      .send({
        seasonId: 1,
        playerAddress: '0x...',
        buyYes: true,
        amountIn: '10'
      })
      .expect(200);
    
    expect(response.body).toHaveProperty('amountOut');
    expect(response.body).toHaveProperty('priceImpact');
  });
});
```

## Error Handling

### Common Errors

**Market Not Found (404)**
```json
{
  "error": "Market not found"
}
```
*Cause*: Player hasn't crossed 1% threshold or market not created yet.

**FPMM Manager Not Configured (500)**
```json
{
  "error": "FPMM manager address not configured"
}
```
*Cause*: `INFOFI_FPMM_MANAGER_ADDRESS` not set in environment.

**Missing Parameters (400)**
```json
{
  "error": "Missing required parameters"
}
```
*Cause*: Required request parameters not provided.

**Contract Read Failed (500)**
```json
{
  "error": "Failed to fetch market data",
  "message": "execution reverted"
}
```
*Cause*: Contract call failed (wrong address, network issue, etc.).

## Performance Considerations

### Caching Strategy

1. **Market Prices**: Cache for 5-10 seconds (high volatility)
2. **Market Reserves**: Cache for 10 seconds
3. **LP Positions**: Cache for 30 seconds (less volatile)
4. **Market Addresses**: Cache for 1 hour (rarely changes)

### Rate Limiting

Current: 100 requests per minute per IP (via Fastify rate-limit plugin)

### Optimization Tips

1. Use `getCompleteMarketData()` instead of multiple calls
2. Batch requests when fetching multiple markets
3. Implement Redis caching for frequently accessed data
4. Use WebSocket for real-time price updates (future)

## Deployment Checklist

- [ ] Set `INFOFI_FPMM_MANAGER_ADDRESS` in production `.env`
- [ ] Set `CHAIN_ID` to production chain (8453 for Base)
- [ ] Set `RPC_URL` to production RPC endpoint
- [ ] Configure CORS for production domain
- [ ] Set up monitoring for API endpoints
- [ ] Implement rate limiting per user (not just IP)
- [ ] Add API key authentication for sensitive endpoints
- [ ] Set up error tracking (Sentry, etc.)
- [ ] Configure logging aggregation
- [ ] Test all endpoints on testnet first

## Migration from CSMM

### Breaking Changes

1. **Endpoint Changes**
   - Old: `/api/infofi/markets/:seasonId/:player`
   - New: `/api/fpmm/market/:seasonId/:playerAddress`

2. **Response Format**
   - Old: CSMM returned linear prices
   - New: FPMM returns non-linear prices with reserves

3. **Price Calculation**
   - Old: Constant sum (P = reserve ratio)
   - New: Constant product (P from x*y=k)

### Migration Steps

1. Update frontend to use new endpoints
2. Update price display logic for FPMM
3. Add slippage warnings for large trades
4. Implement LP position tracking
5. Remove old CSMM API calls

## Future Enhancements

1. **WebSocket Support**: Real-time price updates
2. **Historical Data**: Price charts and trade history
3. **Analytics**: Volume, liquidity depth, APY calculations
4. **Notifications**: Price alerts, LP rewards notifications
5. **Advanced Queries**: Top markets, trending players, etc.

## References

- [FPMM Migration Summary](./FPMM_MIGRATION_SUMMARY.md)
- [Viem Documentation](https://viem.sh)
- [Fastify Documentation](https://www.fastify.io)
- [Gnosis FPMM Whitepaper](https://docs.gnosis.io/conditionaltokens/)
