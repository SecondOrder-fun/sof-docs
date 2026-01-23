# Manual Market Creation Testing Guide

## Quick Start

Follow these steps to test the Paymaster-sponsored market creation via the Admin panel.

## Prerequisites

- [x] Backend running (`npm run dev:backend`)
- [x] Frontend running (`npm run dev:frontend`)
- [x] Contracts deployed to Base Sepolia
- [ ] Paymaster allowlist configured in CDP dashboard

## Step 1: Configure Paymaster Allowlist

1. Go to [Coinbase Developer Platform](https://portal.cdp.coinbase.com/)
2. Navigate to **Onchain Tools > Paymaster**
3. Click **"Configure Allowlist"**
4. Add this entry:

   ```text
   Contract Address: 0xB654A23D56B677C573d92BB69760A12ea3cDf9f9
   Function Signature: onPositionUpdate(uint256,address,uint256,uint256,uint256)
   Function Selector: 0x1a1d1f6f
   Network: Base Sepolia (84532)
   ```

5. Click **Save**

## Step 2: Verify Backend Initialization

Check backend logs for:

```text
✅ PaymasterService initialized with viem wallet client
   Network: Base Sepolia
   Account: 0x1eD4aC856D7a072C3a336C0971a47dB86A808Ff4
```

If you see an error, check:

- `PAYMASTER_RPC_URL_TESTNET` is set in `.env`
- `BACKEND_WALLET_PRIVATE_KEY` is set in `.env`
- `DEFAULT_NETWORK=TESTNET` is set in `.env`

## Step 3: Access Admin Panel

1. Open browser: `http://localhost:5173/admin`
2. Scroll to **"Manual Market Creation"** section
3. You should see:
   - Season dropdown
   - Player Address input
   - "Create Market" button

## Step 4: Get Test Data

You need:

1. **Season ID**: An active season with participants
2. **Player Address**: A player who has bought tickets in that season

### Option A: Use Existing Season

Check the frontend or database for active seasons and players.

### Option B: Create Test Season and Buy Tickets

```bash
# 1. Create a season (via frontend or script)
# 2. Buy tickets to create a participant
# 3. Note the season ID and player address
```

## Step 5: Create Market Manually

1. **Select Season**: Choose from dropdown (e.g., "Season 1")
2. **Enter Player Address**: Paste the player's Ethereum address

   Example: `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`

3. **Click "Create Market"**

## Step 6: Monitor Backend Logs

Watch for these log messages:

```text
Admin requested manual market creation
🔄 Attempt 1/3: Creating market for player 0x7099...
✅ Market creation transaction submitted: 0xabc123...
```

## Step 7: Verify on BaseScan

1. Copy the transaction hash from logs
2. Open: `https://sepolia.basescan.org/tx/0x[HASH]`
3. Check:
   - ✅ **Status**: Success
   - ✅ **From**: Your backend wallet (0x1eD4aC...)
   - ✅ **To**: InfoFiMarketFactory (0xB654A23D...)
   - ✅ **Gas Paid By**: Should show Paymaster sponsorship

## Step 8: Verify in Database

Check that the market was created:

```sql
SELECT * FROM infofi_markets
WHERE season_id = [YOUR_SEASON_ID]
AND player_address = '[YOUR_PLAYER_ADDRESS]';
```

Should return a row with:

- `is_active = true`
- `contract_address` populated
- `created_at` timestamp

## Expected Results

### Success Case

**Frontend**:

- ✅ Green success message: "Market created successfully!"
- ✅ Transaction hash displayed
- ✅ Form clears

**Backend Logs**:

```text
✅ Market creation transaction submitted: 0x...
```

**BaseScan**:

- ✅ Transaction confirmed
- ✅ Gas sponsored by Paymaster
- ✅ Function call to `onPositionUpdate`

**Database**:

- ✅ New row in `infofi_markets` table

### Failure Cases

#### Case 1: "Player has zero tickets"

**Error**: `Player has zero tickets in this season; no market should be created`

**Fix**: Use a player address that has actually bought tickets

#### Case 2: "Transaction not sponsored"

**Error**: Transaction fails with "insufficient funds"

**Fix**:

1. Verify contract is in Paymaster allowlist
2. Check function selector matches (`0x1a1d1f6f`)
3. Ensure network is Base Sepolia (84532)

#### Case 3: "Paymaster not configured"

**Error**: `PAYMASTER_RPC_URL_TESTNET not configured`

**Fix**: Add to `.env`:

```bash
PAYMASTER_RPC_URL_TESTNET=https://api.developer.coinbase.com/rpc/v1/base-sepolia/YOUR_API_KEY
```

## Troubleshooting

### Backend Won't Start

**Check**:

```bash
# Verify environment variables
grep PAYMASTER .env
grep BACKEND_WALLET .env
grep DEFAULT_NETWORK .env
```

### Season Dropdown Empty

**Check**:

- Are there any active seasons?
- Is the backend API responding?
- Check browser console for errors

### Transaction Fails

**Check Backend Logs**:

```text
❌ Attempt 1/3 failed: [error message]
```

**Common Errors**:

- "Insufficient funds" → Paymaster not sponsoring
- "Execution reverted" → Contract error (check allowlist)
- "Invalid address" → Check player address format

## Success Checklist

- [ ] Paymaster allowlist configured
- [ ] Backend initialized successfully
- [ ] Admin panel loads
- [ ] Season dropdown populated
- [ ] Market creation succeeds
- [ ] Transaction hash received
- [ ] BaseScan shows sponsored transaction
- [ ] Database has new market entry

## Next Steps After Success

1. Test automatic market creation (buy tickets to cross 1% threshold)
2. Monitor Paymaster credit usage in CDP dashboard
3. Test with multiple players
4. Verify market settlement works
5. Deploy to production when ready

---

**Need Help?**

Check these files:

- `PAYMASTER_BACKEND_IMPLEMENTATION.md` - Implementation details
- `PAYMASTER_DEPLOYMENT_CHECKLIST.md` - Full deployment guide
- Backend logs: Look for PaymasterService messages
- Frontend console: Check for API errors
