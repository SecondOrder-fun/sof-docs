# Paymaster Deployment Checklist

## ✅ Completed Steps

- [x] **Step 1**: Obtain Paymaster service URL from Coinbase Developer Platform
- [x] **Step 2**: Install Base Account SDK (`@base-org/account`)
- [x] **Step 2**: Implement PaymasterService with Base Account SDK
- [x] **Step 2**: Create comprehensive documentation

## ⏳ Pending Steps

### Step 3: Configure Paymaster Allowlist

- [ ] Log into [Coinbase Developer Platform](https://portal.cdp.coinbase.com/)
- [ ] Navigate to **Onchain Tools > Paymaster**
- [ ] Click **"Configure Allowlist"**
- [ ] Add InfoFiMarketFactory contract:
  - Contract: `0xB654A23D56B677C573d92BB69760A12ea3cDf9f9`
  - Function: `onPositionUpdate(uint256,address,uint256,uint256,uint256)`
  - Selector: `0x1a1d1f6f`
  - Network: Base Sepolia (84532)
- [ ] Save and verify allowlist is active

### Step 4: Test on Base Sepolia

- [ ] Ensure `.env` has correct Paymaster URL:

  ```bash
  PAYMASTER_RPC_URL_TESTNET=https://api.developer.coinbase.com/rpc/v1/base-sepolia/GHRTIThbJv3KbrmeXIB4QW7aDTOihzpJ
  ```

- [ ] Start backend: `npm run dev:backend`
- [ ] Verify initialization log:

  ```text
  ✅ PaymasterService initialized with Base Account SDK
     Network: Base Sepolia
  ```

- [ ] Buy tickets to cross 1% threshold (trigger market creation)
- [ ] Monitor backend logs for:

  ```text
  🎯 Threshold crossed: Player 0x... reached 1000 bps (≥1%)
  🚀 Submitting gasless market creation via Paymaster...
  ✅ Market creation submitted: 0x...
  ```

- [ ] Verify transaction on [BaseScan](https://sepolia.basescan.org/)
- [ ] Confirm gas was sponsored (paid by Paymaster, not EOA)

### Step 5: Monitor and Debug

- [ ] Check Paymaster credit usage in CDP dashboard
- [ ] Review backend error logs if any failures occur
- [ ] Verify InfoFi markets created in database
- [ ] Test multiple market creations (different players)

### Step 6: Production Deployment (When Ready)

- [ ] Update `.env` with mainnet Paymaster URL:

  ```bash
  DEFAULT_NETWORK=MAINNET
  PAYMASTER_RPC_URL=https://api.developer.coinbase.com/rpc/v1/base/YOUR_MAINNET_KEY
  ```

- [ ] Configure mainnet allowlist in CDP dashboard
- [ ] Deploy contracts to Base mainnet
- [ ] Update contract addresses in `.env`
- [ ] Test with small amounts first
- [ ] Monitor Paymaster credits closely
- [ ] Set up alerts for low credit balance

## Troubleshooting Checklist

### If Market Creation Fails

- [ ] Check Paymaster allowlist includes correct contract address
- [ ] Verify function selector is `0x1a1d1f6f`
- [ ] Confirm network is Base Sepolia (84532)
- [ ] Check Paymaster credits are available
- [ ] Review backend error logs for specific error messages
- [ ] Verify `PAYMASTER_RPC_URL_TESTNET` is correct

### If "No accounts available" Error

This indicates Base Account SDK requires user authentication. Options:

- [ ] **Option A**: Implement backend wallet fallback (use viem wallet client directly)
- [ ] **Option B**: Use hybrid approach (frontend triggers, backend monitors)
- [ ] **Option C**: Implement custom server-side smart account

### If Transaction Not Sponsored

- [ ] Verify contract is in Paymaster allowlist
- [ ] Check function signature matches exactly
- [ ] Confirm network matches (testnet vs mainnet)
- [ ] Review CDP dashboard for policy violations

## Success Criteria

✅ Backend initializes PaymasterService without errors
✅ Position update events trigger market creation attempts
✅ Transactions submitted via `wallet_sendCalls`
✅ Paymaster sponsors transactions (gas paid by Paymaster)
✅ InfoFi markets created on-chain
✅ Markets visible in database
✅ No gas fees charged to users or backend wallet

## Resources

- [CDP Dashboard](https://portal.cdp.coinbase.com/)
- [Base Sepolia Explorer](https://sepolia.basescan.org/)
- [Paymaster Docs](https://docs.cdp.coinbase.com/paymaster/introduction/welcome)
- [Base Account SDK Docs](https://docs.base.org/base-account)
- [Implementation Guide](./docs/02-architecture/BASE_ACCOUNT_SDK_INTEGRATION.md)
- [Setup Guide](./docs/02-architecture/PAYMASTER_SETUP_GUIDE.md)

## Notes

- **Base Gasless Campaign**: Provides up to $15k in free Paymaster credits
- **Retry Logic**: 3 attempts with 5s, 15s, 45s delays
- **Error Tracking**: Failed attempts logged to database
- **Monitoring**: Check CDP dashboard regularly for credit usage

---

**Last Updated**: November 19, 2025
**Status**: Implementation complete, testing pending
