# Verifying Oracle Updates After Placing Bets

## Issue

After placing a 10 SOF NO bet on a market, the displayed odds remain at 70-30 instead of updating to reflect the new market sentiment.

## How to Verify Oracle is Being Updated

### 1. Check Oracle Price Directly (via cast)

```bash
# Get the oracle price for market ID 0
cast call $INFOFI_ORACLE_ADDRESS "getPrice(uint256)" 0 --rpc-url $RPC_URL

# This returns: (raffleProbabilityBps, marketSentimentBps, hybridPriceBps, lastUpdate, active)
# Example output: (10000, 5000, 8500, 1234567890, true)
# Means: 100% raffle, 50% market sentiment, 85% hybrid, timestamp, active
```

### 2. Check Market Pools

```bash
# Get market info to see YES/NO pools
cast call $INFOFI_MARKET_ADDRESS "getMarket(uint256)" 0 --rpc-url $RPC_URL

# Look for totalYesPool and totalNoPool values
```

### 3. Calculate Expected Sentiment

If you placed 10 SOF on NO:
- Before: totalYesPool = X, totalNoPool = Y, totalPool = X + Y
- After: totalYesPool = X, totalNoPool = Y + 10, totalPool = X + Y + 10
- Market sentiment (YES%) = (X * 10000) / (X + Y + 10)

### 4. Check Frontend React Query Cache

Open browser DevTools → React Query DevTools → Look for:
- Query key: `['oraclePrice', marketId, 'LOCAL']`
- Check `data.hybridPriceBps`, `data.marketSentimentBps`
- Check `dataUpdatedAt` timestamp

## Expected Behavior

After placing a NO bet:
1. **Market sentiment should decrease** (fewer YES bets relative to total)
2. **Hybrid price should decrease slightly** (30% weight on market sentiment)
3. **Frontend should update within 10 seconds** (React Query refetch interval)

## Example Calculation

**Before bet:**
- Raffle probability: 100% (10000 bps)
- Total YES pool: 100 SOF
- Total NO pool: 0 SOF
- Market sentiment: 100% (all bets are YES)
- Hybrid price: 70% × 100% + 30% × 100% = **100%**

**After 10 SOF NO bet:**
- Raffle probability: 100% (unchanged - player still has all tickets)
- Total YES pool: 100 SOF
- Total NO pool: 10 SOF
- Market sentiment: (100 / 110) × 100% = **90.9%**
- Hybrid price: 70% × 100% + 30% × 90.9% = 70% + 27.3% = **97.3%**

So the odds should shift from 100-0 to approximately 97.3-2.7.

## Troubleshooting Steps

### Step 1: Verify Oracle is Set

```bash
cast call $INFOFI_MARKET_ADDRESS "oracle()" --rpc-url $RPC_URL
# Should return the oracle address, not 0x0000...
```

### Step 2: Verify Oracle Has PRICE_UPDATER_ROLE for Market Contract

```bash
# Get PRICE_UPDATER_ROLE hash
cast call $INFOFI_ORACLE_ADDRESS "PRICE_UPDATER_ROLE()" --rpc-url $RPC_URL

# Check if market contract has this role
cast call $INFOFI_ORACLE_ADDRESS "hasRole(bytes32,address)" <ROLE_HASH> $INFOFI_MARKET_ADDRESS --rpc-url $RPC_URL
# Should return true
```

### Step 3: Check Recent Events

```bash
# Check for OracleUpdated events from InfoFiMarket
cast logs --address $INFOFI_MARKET_ADDRESS --from-block latest-100 --rpc-url $RPC_URL

# Check for PriceUpdated events from Oracle
cast logs --address $INFOFI_ORACLE_ADDRESS --from-block latest-100 --rpc-url $RPC_URL
```

### Step 4: Force Frontend Refresh

In the browser console:
```javascript
// Invalidate React Query cache
queryClient.invalidateQueries({ queryKey: ['oraclePrice'] })

// Or manually refetch
const { refetch } = useHybridPriceLive(marketId)
refetch()
```

## Common Issues

1. **Oracle not set on InfoFiMarket contract** - Market updates won't reach oracle
2. **Oracle not initialized for this market** - `raffleProbabilityBps` is 0, causing incorrect calculations
3. **Frontend polling hasn't triggered** - Wait 10 seconds or force refresh
4. **Market sentiment calculation error** - Check if totalPool is 0 (division by zero protection)
5. **React Query stale data** - Cache might be serving old data

## Fix If Oracle Not Updating

If the oracle isn't being updated, you may need to:

1. **Set oracle on market contract** (if not set):
```bash
cast send $INFOFI_MARKET_ADDRESS "setOracle(address)" $INFOFI_ORACLE_ADDRESS --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

2. **Grant PRICE_UPDATER_ROLE to market contract**:
```bash
# Get role hash
ROLE=$(cast call $INFOFI_ORACLE_ADDRESS "PRICE_UPDATER_ROLE()" --rpc-url $RPC_URL)

# Grant role
cast send $INFOFI_ORACLE_ADDRESS "grantRole(bytes32,address)" $ROLE $INFOFI_MARKET_ADDRESS --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

3. **Initialize raffle probability** (if not set):
```bash
# Calculate player's win probability
# If player has 15000 tickets out of 15000 total = 100% = 10000 bps
cast send $INFOFI_ORACLE_ADDRESS "updateRaffleProbability(uint256,uint256)" 0 10000 --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```
