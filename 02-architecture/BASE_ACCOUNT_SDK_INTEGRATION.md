# Base Account SDK Integration for Sponsor Gas

## Overview

SecondOrder.fun uses the Base Account SDK to sponsor InfoFi market creation transactions via Coinbase Developer Platform's Paymaster service. This allows the backend to create prediction markets on behalf of users without requiring them to pay gas fees.

## Architecture

### Flow

```text
PositionUpdate Event → Backend Listener → Threshold Check (1%)
    → PaymasterService.createMarket()
    → Base Account SDK (wallet_sendCalls)
    → Coinbase Paymaster
    → InfoFiMarketFactory.onPositionUpdate()
    → Market Created (gasless)
```

### Key Components

1. **Base Account SDK** (`@base-org/account`)

   - Provides `createBaseAccountSDK()` for initialization
   - Exposes `provider` for EIP-5792 wallet RPC methods
   - Handles `wallet_sendCalls` with paymaster capabilities

2. **PaymasterService** (`backend/src/services/paymasterService.js`)

   - Singleton service managing Base Account SDK
   - Encodes function calls using viem's `encodeFunctionData`
   - Submits transactions with `wallet_sendCalls`
   - Implements retry logic (3 attempts: 5s, 15s, 45s delays)

3. **PositionUpdateListener** (`backend/src/listeners/positionUpdateListener.js`)
   - Monitors bonding curve for ticket purchases
   - Triggers market creation when player crosses 1% threshold
   - Calls `paymasterService.createMarket()` for gasless execution

## Implementation Details

### Environment Variables

```bash
# Required
DEFAULT_NETWORK=TESTNET  # or MAINNET
PAYMASTER_RPC_URL_TESTNET=https://api.developer.coinbase.com/rpc/v1/base-sepolia/YOUR_API_KEY

# Optional (for mainnet)
PAYMASTER_RPC_URL=https://api.developer.coinbase.com/rpc/v1/base/YOUR_API_KEY
```

### Initialization

```javascript
import { createBaseAccountSDK, base } from "@base-org/account";

const sdk = createBaseAccountSDK({
  appName: "SecondOrder.fun",
  appLogoUrl: "https://secondorder.fun/logo.png",
  appChainIds: [base.constants.CHAIN_IDS.baseSepolia],
});

const provider = sdk.getProvider();
```

### Sending Sponsored Transactions

```javascript
const functionData = encodeFunctionData({
  abi: [
    {
      name: "onPositionUpdate",
      type: "function",
      inputs: [
        { name: "seasonId", type: "uint256" },
        { name: "player", type: "address" },
        { name: "oldTickets", type: "uint256" },
        { name: "newTickets", type: "uint256" },
        { name: "totalTickets", type: "uint256" },
      ],
    },
  ],
  functionName: "onPositionUpdate",
  args: [seasonId, player, oldTickets, newTickets, totalTickets],
});

const result = await provider.request({
  method: "wallet_sendCalls",
  params: [
    {
      version: "1.0",
      chainId: numberToHex(base.constants.CHAIN_IDS.baseSepolia),
      from: accountAddress,
      calls: [
        {
          to: infoFiFactoryAddress,
          value: "0x0",
          data: functionData,
        },
      ],
      capabilities: {
        paymasterService: {
          url: process.env.PAYMASTER_RPC_URL_TESTNET,
        },
      },
    },
  ],
});
```

## Paymaster Configuration

### Coinbase Developer Platform Setup

1. **Sign up** at [Coinbase Developer Platform](https://www.coinbase.com/developer-platform)
2. **Navigate** to Onchain Tools > Paymaster
3. **Get Paymaster URL** for Base Sepolia
4. **Configure Allowlist**:
   - Contract: InfoFiMarketFactory address
   - Function: `onPositionUpdate(uint256,address,uint256,uint256,uint256)`

### Allowlist Configuration

The Paymaster will only sponsor transactions to approved contracts and functions:

- **Contract**: `0xYOUR_INFOFI_FACTORY_ADDRESS`
- **Function Selector**: `0x1a1d1f6f` (onPositionUpdate)
- **Network**: Base Sepolia (84532)

## Error Handling

### Retry Strategy

- **Max Retries**: 3 attempts
- **Delays**: 5s → 15s → 45s (exponential backoff)
- **Failure Logging**: Full error objects logged for debugging
- **Database Tracking**: Failed attempts recorded in `failed_market_attempts` table

### Common Errors

1. **"Paymaster not configured"**

   - Ensure `PAYMASTER_RPC_URL_TESTNET` is set
   - Verify URL is correct from CDP dashboard

2. **"No accounts available"**

   - Base Account SDK requires user authentication
   - Backend may need to use a different approach for server-side signing

3. **"Transaction not sponsored"**
   - Check contract is in Paymaster allowlist
   - Verify function selector matches
   - Ensure sufficient Paymaster credits

## Testing

### Local Testing

```bash
# 1. Set environment variables
export DEFAULT_NETWORK=TESTNET
export PAYMASTER_RPC_URL_TESTNET=https://api.developer.coinbase.com/rpc/v1/base-sepolia/YOUR_KEY

# 2. Start backend
npm run dev:backend

# 3. Trigger position update (buy tickets to cross 1% threshold)
# Backend will automatically attempt gasless market creation
```

### Verification

```bash
# Check backend logs for:
✅ PaymasterService initialized with Base Account SDK
   Network: Base Sepolia
🔄 Attempt 1/3: Creating market for player 0x...
✅ Market creation submitted: 0x...
```

## Migration from Viem Account Abstraction

### Changes Made

1. **Removed** viem account abstraction imports:

   - `createBundlerClient`
   - `createPaymasterClient`
   - `toCoinbaseSmartAccount`

2. **Added** Base Account SDK:

   - `createBaseAccountSDK`
   - `base.constants.CHAIN_IDS`

3. **Updated** transaction submission:

   - From: `sendUserOperation()` with bundler client
   - To: `wallet_sendCalls()` with paymaster capabilities

4. **Simplified** initialization:
   - No manual smart account creation
   - SDK handles account management
   - Provider-based API instead of client-based

## Benefits

✅ **Official Base Support**: Uses recommended Base Account SDK
✅ **Simpler API**: Provider-based instead of complex bundler setup
✅ **Better Documentation**: Follows official Base docs exactly
✅ **EIP-5792 Compliant**: Uses standard wallet RPC methods
✅ **Automatic Sponsorship**: Paymaster handles gas payment transparently

## References

- [Base Account SDK Documentation](https://docs.base.org/base-account)
- [Sponsor Gas Guide](https://docs.base.org/base-account/guides/sponsor-gas)
- [Coinbase Developer Platform](https://www.coinbase.com/developer-platform)
- [EIP-5792: Wallet Call API](https://eips.ethereum.org/EIPS/eip-5792)
- [ERC-7677: Paymaster Web API](https://erc7677.xyz/)

## Next Steps

1. ✅ Install `@base-org/account` package
2. ✅ Update PaymasterService to use Base Account SDK
3. ✅ Implement `wallet_sendCalls` with paymaster capabilities
4. ⏳ Configure Paymaster allowlist in CDP dashboard
5. ⏳ Test on Base Sepolia testnet
6. ⏳ Monitor gas sponsorship credits
7. ⏳ Deploy to mainnet with production Paymaster URL
