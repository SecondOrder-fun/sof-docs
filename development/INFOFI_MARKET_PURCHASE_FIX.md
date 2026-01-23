# InfoFi Market Purchase Fix

## Problem Summary

InfoFi market purchases were failing with the error: **"placeBet simulation failed for all marketId candidates"**

## Root Cause

The system has **two separate InfoFi implementations**:

1. **Old System (InfoFiMarket.sol)**: Standalone contract with `placeBet(marketId, prediction, amount)` function
2. **New System (InfoFiFPMMV2.sol + InfoFiMarketFactory.sol)**: FPMM-based prediction markets using Gnosis ConditionalTokens

**The Issue**: The frontend was calling `placeBet()` on the old InfoFiMarket contract, but markets are actually being created through the new FPMM system. Since no markets exist in the old contract, all transactions failed.

## Architecture Overview

### Current System (FPMM-based)

```
InfoFiMarketFactory
    ↓ (creates markets when player crosses 1% threshold)
InfoFiFPMMV2
    ↓ (deploys individual FPMM contracts)
SimpleFPMM (per player/season)
    ↓ (buy/sell functions)
User trades YES/NO positions
```

### How It Works

1. **Market Creation**: When a player's position crosses 1% of total tickets, `InfoFiMarketFactory` automatically creates a market
2. **FPMM Deployment**: `InfoFiFPMMV2.createMarket()` deploys a `SimpleFPMM` contract for that specific player/season
3. **Trading**: Users call `buy(buyYes, amountIn, minAmountOut)` on the FPMM contract (NOT `placeBet` on InfoFiMarket)

## Solution Implemented

### 1. Updated `placeBetTx()` Function

**File**: `src/services/onchainInfoFi.js`

**Changes**:
- Now requires `seasonId` and `player` parameters
- Gets the FPMM contract address via `InfoFiFPMMV2.getMarket(seasonId, player)`
- Calls `buy(buyYes, amountIn, minAmountOut)` on the FPMM contract
- Includes 2% slippage protection using `calcBuyAmount()`
- Approves SOF tokens for the FPMM contract (not InfoFiMarket)

### 2. Updated Frontend Component

**File**: `src/components/infofi/InfoFiMarketCard.jsx`

**Changes**:
- Passes `seasonId` and `market.player` to `placeBetTx()`
- These are required to look up the correct FPMM contract

### 3. Added INFOFI_FPMM Configuration

**Files Updated**:
- `src/config/contracts.js` - Added `INFOFI_FPMM` address
- `.env.example` - Added `VITE_INFOFI_FPMM_ADDRESS_LOCAL` and `_TESTNET` entries
- `scripts/update-env-addresses.js` - Added `InfoFiFPMMV2` to contract name mapping

## Deployment Requirements

### 1. Redeploy or Update .env

After running the deployment script, ensure your `.env` file has:

```bash
VITE_INFOFI_FPMM_ADDRESS_LOCAL=<address_from_deployment>
INFOFI_FPMM_ADDRESS_LOCAL=<address_from_deployment>
```

### 2. Run Update Script

```bash
# From project root
node scripts/update-env-addresses.js
```

This will automatically extract the `InfoFiFPMMV2` address from the deployment broadcast and update your `.env` file.

### 3. Restart Frontend

```bash
npm run dev
```

## Testing the Fix

### 1. Create a Season

```bash
# From contracts directory
forge script script/CreateSeason.s.sol --rpc-url $RPC_URL --private-key $PRIVATE_KEY --broadcast
```

### 2. Buy Tickets (to cross 1% threshold)

```bash
# Buy enough tickets to trigger market creation (1% of total supply)
cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" 2000 3500000000000000000000 --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

### 3. Verify Market Creation

Check that the FPMM contract exists:

```bash
cast call $INFOFI_FPMM_ADDRESS "getMarket(uint256,address)" $SEASON_ID $PLAYER_ADDRESS --rpc-url $RPC_URL
```

Should return a non-zero address.

### 4. Place Bet via Frontend

1. Navigate to the Prediction Markets page
2. Find the market for your player
3. Enter an amount and click "Trade"
4. Approve SOF tokens when prompted
5. Confirm the transaction

## Key Differences: Old vs New System

| Feature | Old (InfoFiMarket) | New (FPMM) |
|---------|-------------------|------------|
| **Function** | `placeBet(marketId, prediction, amount)` | `buy(buyYes, amountIn, minAmountOut)` |
| **Market ID** | Single uint256 for all markets | Separate contract per player/season |
| **Pricing** | Fixed odds | Automated Market Maker (AMM) with dynamic pricing |
| **Liquidity** | No liquidity pools | Initial liquidity provided by treasury |
| **Slippage** | None | 2% slippage protection |
| **Token Approval** | Approve InfoFiMarket | Approve individual FPMM contract |

## Future Improvements

1. **Remove Old System**: The `InfoFiMarket.sol` contract is now deprecated and should be removed to avoid confusion
2. **Better Error Messages**: Add user-friendly error messages when FPMM doesn't exist yet
3. **Market Discovery**: Improve UI to show when markets will be created (at 1% threshold)
4. **Liquidity Provision**: Allow users to provide liquidity to FPMM pools for additional yield

## Related Files

- `contracts/src/infofi/InfoFiFPMMV2.sol` - FPMM manager contract
- `contracts/src/infofi/InfoFiMarketFactory.sol` - Automatic market creation
- `contracts/src/infofi/InfoFiMarket.sol` - **DEPRECATED** (old system)
- `src/services/onchainInfoFi.js` - Frontend integration
- `src/components/infofi/InfoFiMarketCard.jsx` - Market UI component

## Troubleshooting

### "No FPMM market exists for this player yet"

**Cause**: The player hasn't crossed the 1% threshold yet, so no market has been created.

**Solution**: Wait for the player to buy more tickets, or manually create a market via the factory.

### "FPMM buy failed: insufficient allowance"

**Cause**: SOF token approval for the FPMM contract is insufficient.

**Solution**: The code should handle this automatically, but you can manually approve:

```bash
cast send $SOF_ADDRESS "approve(address,uint256)" $FPMM_ADDRESS 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

### "Failed to get FPMM address"

**Cause**: The `INFOFI_FPMM` address is not configured in your `.env` file.

**Solution**: Run `node scripts/update-env-addresses.js` and restart the frontend.
