# Paymaster Setup Guide - Step by Step

## Current Status

✅ **Step 1 Complete**: Paymaster service URL obtained from Coinbase Developer Platform
✅ **Step 2 Complete**: Base Account SDK installed and integrated
⏳ **Step 3 Pending**: Configure allowlist and test

## Step 3: Configure Paymaster Allowlist

### 1. Get InfoFi Factory Contract Address

From your `.env` file:

```bash
# Testnet
VITE_INFOFI_FACTORY_ADDRESS_TESTNET=0xB654A23D56B677C573d92BB69760A12ea3cDf9f9
INFOFI_FACTORY_ADDRESS_TESTNET=0xB654A23D56B677C573d92BB69760A12ea3cDf9f9
```

### 2. Configure Allowlist in CDP Dashboard

1. Go to [Coinbase Developer Platform](https://portal.cdp.coinbase.com/)
2. Navigate to **Onchain Tools > Paymaster**
3. Click **"Configure Allowlist"**
4. Add the following entry:

   **Contract Address**: `0xB654A23D56B677C573d92BB69760A12ea3cDf9f9`

   **Function Selector**: `0x1a1d1f6f`

   **Function Name**: `onPositionUpdate(uint256,address,uint256,uint256,uint256)`

   **Network**: Base Sepolia (Chain ID: 84532)

5. Click **"Save"**

### 3. Verify Allowlist Configuration

The allowlist should show:

```text
Contract: 0xB654A23D56B677C573d92BB69760A12ea3cDf9f9
Function: onPositionUpdate(uint256,address,uint256,uint256,uint256)
Selector: 0x1a1d1f6f
Network: Base Sepolia (84532)
Status: Active
```

## Step 4: Test Gasless Market Creation

### 1. Ensure Backend is Running

```bash
# Terminal 1: Start backend
npm run dev:backend

# Look for initialization message:
# ✅ PaymasterService initialized with Base Account SDK
#    Network: Base Sepolia
```

### 2. Trigger Market Creation

You need to buy tickets to cross the 1% threshold:

```bash
# Option A: Use frontend to buy tickets
# Navigate to http://localhost:5173
# Connect wallet and buy enough tickets to reach 1%

# Option B: Use contract script
cd contracts
forge script script/BuyTickets.s.sol \
  --rpc-url $RPC_URL_TESTNET \
  --private-key $PRIVATE_KEY \
  --broadcast
```

### 3. Monitor Backend Logs

Watch for these log messages:

```text
📊 PositionUpdate Event: Season 1, Player 0x..., Tickets: 0 → 2000, Total: 200000
🎯 Threshold crossed: Player 0x... reached 1000 bps (≥1%)
🚀 Submitting gasless market creation via Paymaster...
🔄 Attempt 1/3: Creating market for player 0x...
✅ Market creation submitted: 0x...
✅ Market creation confirmed: 0x...
```

### 4. Verify on BaseScan

Check the transaction on Base Sepolia:

```text
https://sepolia.basescan.org/tx/0x[TRANSACTION_HASH]

Look for:
- From: Smart Account Address (not your EOA)
- To: InfoFiMarketFactory (0xB654A23D56B677C573d92BB69760A12ea3cDf9f9)
- Gas Paid By: Paymaster (sponsored)
- Status: Success
```

## Troubleshooting

### Error: "Paymaster not configured"

**Solution**: Verify `PAYMASTER_RPC_URL_TESTNET` in `.env`:

```bash
PAYMASTER_RPC_URL_TESTNET=https://api.developer.coinbase.com/rpc/v1/base-sepolia/GHRTIThbJv3KbrmeXIB4QW7aDTOihzpJ
```

### Error: "Transaction not sponsored"

**Possible Causes**:

1. **Contract not in allowlist**

   - Verify contract address matches exactly
   - Check network is Base Sepolia (84532)

2. **Function not allowed**

   - Verify function selector is `0x1a1d1f6f`
   - Check function signature matches exactly

3. **Insufficient Paymaster credits**
   - Check CDP dashboard for remaining credits
   - Base Gasless Campaign provides $15k free credits

### Error: "No accounts available"

**Issue**: Base Account SDK requires user authentication for wallet operations.

**Backend Workaround**: The backend needs to use a server-side signing approach. We may need to:

1. Use a different account abstraction approach for backend
2. Or use traditional EOA signing with gas payment
3. Or implement a hybrid: frontend triggers, backend monitors

**Immediate Fix**: For now, we'll need to adjust the implementation to use a backend wallet directly instead of relying on Base Account SDK's provider.

## Alternative: Backend Wallet Approach

If Base Account SDK doesn't work for backend (server-side) operations, we can use:

```javascript
// Use viem's wallet client directly
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { baseSepolia } from "viem/chains";

const account = privateKeyToAccount(process.env.BACKEND_WALLET_PRIVATE_KEY);
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(process.env.PAYMASTER_RPC_URL_TESTNET),
});

// Send transaction (Paymaster will sponsor if in allowlist)
const hash = await walletClient.writeContract({
  address: infoFiFactoryAddress,
  abi: infoFiFactoryAbi,
  functionName: "onPositionUpdate",
  args: [seasonId, player, oldTickets, newTickets, totalTickets],
});
```

## Next Steps

1. ✅ Configure Paymaster allowlist in CDP dashboard
2. ⏳ Test gasless market creation on testnet
3. ⏳ Monitor Paymaster credit usage
4. ⏳ Adjust implementation if Base Account SDK doesn't work for backend
5. ⏳ Deploy to mainnet when ready

## Resources

- [CDP Dashboard](https://portal.cdp.coinbase.com/)
- [Base Sepolia Explorer](https://sepolia.basescan.org/)
- [Paymaster Documentation](https://docs.cdp.coinbase.com/paymaster/introduction/welcome)
- [Base Gasless Campaign](https://base.org/gasless)
