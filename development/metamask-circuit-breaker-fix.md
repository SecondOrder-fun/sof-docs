# MetaMask Circuit Breaker Issue - Fix Documentation

## Problem Description

When working with local Anvil development, MetaMask can enter a "circuit breaker" state where it refuses to make any RPC requests. This happens when:

1. **Anvil is not running** or not accessible at `http://127.0.0.1:8545`
2. **MetaMask repeatedly fails** to connect to the RPC endpoint
3. **MetaMask's internal circuit breaker trips** to prevent hammering a failing endpoint

### Error Signature

```json
{
  "code": -32603,
  "message": "Execution prevented because the circuit breaker is open",
  "data": {
    "cause": {
      "message": "Execution prevented because the circuit breaker is open",
      "isBrokenCircuitError": true
    }
  }
}
```

## Root Causes

1. **Anvil Not Running**: The most common cause - Anvil process stopped or never started
2. **Port Conflicts**: Another process using port 8545
3. **Network Mismatch**: MetaMask connected to wrong network
4. **Page Reload During Transaction**: Reloading while MetaMask is processing can trigger this

## Solutions Implemented

### 1. Enhanced Error Detection

Added circuit breaker detection in `useRaffleWrite.js`:

```javascript
const buildFriendlyMessage = (abi, err, fallback = 'Transaction failed') => {
  // Check for MetaMask circuit breaker error
  if (err?.message?.includes('circuit breaker') || err?.data?.cause?.isBrokenCircuitError) {
    return 'MetaMask circuit breaker tripped. Please switch to another network and back to reset the connection, or restart MetaMask.';
  }
  // ... rest of error handling
};
```

### 2. Preflight RPC Check

Added RPC connectivity check before transactions:

```javascript
onMutate: async () => {
  if (publicClient && raffleContractConfig.address) {
    try {
      // First check if we can connect to the RPC
      await publicClient.getBlockNumber();
    } catch (rpcErr) {
      if (rpcErr?.message?.includes('circuit breaker') || rpcErr?.data?.cause?.isBrokenCircuitError) {
        throw new Error('MetaMask circuit breaker tripped. Please switch to another network and back to reset the connection.');
      }
      if (rpcErr?.message?.includes('fetch') || rpcErr?.message?.includes('ECONNREFUSED')) {
        throw new Error('Cannot connect to RPC. Make sure Anvil is running on http://127.0.0.1:8545');
      }
      throw rpcErr;
    }
    // ... rest of preflight checks
  }
}
```

### 3. User-Friendly Alert Component

Created `MetaMaskCircuitBreakerAlert.jsx` to guide users through recovery:

- Detects circuit breaker errors automatically
- Provides step-by-step recovery instructions
- Offers quick refresh button
- Can be dismissed once resolved

## How to Recover (User Steps)

### Method 1: Network Switch (Recommended)

1. **Open MetaMask** extension
2. **Click network dropdown** (top left)
3. **Switch to a different network** (e.g., Ethereum Mainnet)
4. **Wait 2-3 seconds**
5. **Switch back to "Localhost 8545"**
6. **Try your transaction again**

### Method 2: Restart MetaMask

1. **Close all browser tabs** using the dApp
2. **Disable MetaMask extension** in browser settings
3. **Re-enable MetaMask extension**
4. **Reload the dApp page**

### Method 3: Restart Anvil

1. **Stop Anvil** (Ctrl+C in terminal)
2. **Restart Anvil**: `anvil -p 8545 --chain-id 31337`
3. **Follow Method 1** to reset MetaMask
4. **Redeploy contracts** if needed

## Prevention Tips

### For Developers

1. **Always check Anvil is running** before starting the frontend:
   ```bash
   # Check if Anvil is running
   curl http://127.0.0.1:8545 -X POST -H "Content-Type: application/json" \
     --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
   ```

2. **Use the dev stack script** to start everything together:
   ```bash
   npm run dev:stack
   ```

3. **Don't reload during transactions** - Wait for transactions to complete

4. **Monitor Anvil logs** for connection issues

### For Users

1. **Check the network indicator** in MetaMask before transactions
2. **Look for the circuit breaker alert** in the UI
3. **Follow the recovery steps** in the alert
4. **Contact support** if issue persists

## Testing the Fix

To verify the circuit breaker handling works:

1. **Stop Anvil** while the dApp is running
2. **Try to create a season** in the Admin Panel
3. **Verify** the MetaMaskCircuitBreakerAlert appears
4. **Follow recovery steps** and confirm it works

## Related Files

- `/src/hooks/useRaffleWrite.js` - Error detection and handling
- `/src/components/common/MetaMaskCircuitBreakerAlert.jsx` - User alert component
- `/src/components/admin/CreateSeasonForm.jsx` - Integration point

## Future Improvements

1. **Auto-detect Anvil status** and show warning before user attempts transaction
2. **Add retry logic** with exponential backoff for transient failures
3. **Implement connection health monitoring** to prevent circuit breaker from tripping
4. **Add telemetry** to track how often this occurs in production

## References

- [MetaMask Circuit Breaker Pattern](https://github.com/MetaMask/eth-json-rpc-middleware)
- [Viem Error Handling](https://viem.sh/docs/error-handling.html)
- [Wagmi Transaction Lifecycle](https://wagmi.sh/react/guides/write-to-contract)
