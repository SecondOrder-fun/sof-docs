# SecondOrder.fun Troubleshooting Guide

## Backend-Driven InfoFi Market Creation

### Issue: Markets Not Being Created After Ticket Purchase

**Symptoms:**
- User buys tickets and crosses 1% threshold
- No InfoFi market appears in database
- Backend logs show no market creation attempts

**Diagnosis Steps:**

1. **Check Backend is Running:**
   ```bash
   # Verify backend process is active
   ps aux | grep "node.*fastify"
   
   # Check backend logs
   tail -f backend/logs/server.log
   ```

2. **Verify PositionUpdate Events Are Being Emitted:**
   ```bash
   # From contracts directory
   cast logs --address $CURVE_ADDRESS \
     --from-block latest \
     --to-block latest \
     --rpc-url $RPC_URL
   ```

3. **Check Backend Wallet Balance:**
   ```bash
   # Backend needs ETH for gas
   cast balance $BACKEND_WALLET_ADDRESS --rpc-url $RPC_URL
   ```

**Common Causes & Solutions:**

#### Backend Not Listening to Events

**Cause:** Backend listener not started or crashed

**Solution:**
```bash
# Restart backend
cd backend
npm run dev

# Check logs for "Bonding curve listener started"
```

#### Backend Wallet Out of Gas

**Cause:** Backend wallet has insufficient ETH for market creation transactions

**Solution:**
```bash
# Fund backend wallet
cast send $BACKEND_WALLET_ADDRESS \
  --value 1ether \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY
```

#### Threshold Not Crossed

**Cause:** Player position is below 1% of total tickets

**Solution:**
- Check player's actual position percentage
- Verify threshold configuration in backend (default: 100 basis points = 1%)

#### Historical Events Not Scanned

**Cause:** Backend restarted and missed events during downtime

**Solution:**
- Backend automatically scans on startup
- Check logs for "Scanning for missed PositionUpdate events"
- Verify `event_processing_state` table has correct `last_block`

---

### Issue: Market Creation Transaction Failing

**Symptoms:**
- Backend logs show "Market creation failed"
- Transaction reverts on-chain
- Error: "AccessControlUnauthorizedAccount"

**Diagnosis:**

1. **Check Backend Wallet Has BACKEND_ROLE:**
   ```bash
   # Check if backend wallet has the role
   cast call $INFOFI_FACTORY_ADDRESS \
     "hasRole(bytes32,address)" \
     $(cast keccak "BACKEND_ROLE") \
     $BACKEND_WALLET_ADDRESS \
     --rpc-url $RPC_URL
   ```

2. **Verify InfoFiMarketFactory Address:**
   ```bash
   # Ensure backend is calling correct contract
   echo $INFOFI_FACTORY_ADDRESS
   ```

**Solutions:**

#### Missing BACKEND_ROLE

**Cause:** Backend wallet not granted BACKEND_ROLE during deployment

**Solution:**
```bash
# Grant role from admin wallet
cast send $INFOFI_FACTORY_ADDRESS \
  "grantRole(bytes32,address)" \
  $(cast keccak "BACKEND_ROLE") \
  $BACKEND_WALLET_ADDRESS \
  --rpc-url $RPC_URL \
  --private-key $ADMIN_PRIVATE_KEY
```

#### Gas Price Too High

**Cause:** Gas price exceeds backend's max limit (100 gwei default)

**Solution:**
- Wait for gas prices to decrease
- Or increase `MAX_GAS_PRICE` in backend configuration
- Or manually create market via admin panel

---

### Issue: Duplicate Market Creation Attempts

**Symptoms:**
- Multiple market creation transactions for same player
- Database shows duplicate market records
- Backend logs show repeated "Threshold crossed" messages

**Diagnosis:**

Check if multiple backend instances are running:
```bash
ps aux | grep "node.*fastify" | wc -l
# Should return 1, not more
```

**Solution:**

Kill duplicate processes:
```bash
pkill -f "node.*fastify"
# Then restart single instance
cd backend && npm run dev
```

---

## Gas Cost Issues

### Issue: User Gas Costs Higher Than Expected

**Expected:** ~100k gas for ticket purchase
**Actual:** ~500k gas

**Cause:** User is calling old contract that still has on-chain market creation

**Solution:**
- Verify user is interacting with correct bonding curve address
- Check that bonding curve does NOT have `infoFiMarketFactory` set
- Ensure frontend is using updated ABIs

---

## Admin Panel Issues

### Issue: Backend Wallet Balance Not Showing

**Symptoms:**
- Admin panel shows "Backend wallet not configured"
- Balance displays as 0

**Diagnosis:**

1. **Check Environment Variables:**
   ```bash
   echo $BACKEND_WALLET_ADDRESS
   # Should show valid address
   ```

2. **Verify API Endpoint:**
   ```bash
   curl http://localhost:3000/api/admin/backend-wallet
   ```

**Solution:**

Set correct environment variable:
```bash
# In .env file
BACKEND_WALLET_ADDRESS=0xYourBackendWalletAddress

# Restart backend
```

---

### Issue: Manual Market Creation Fails

**Symptoms:**
- Admin panel shows "Failed to create market"
- Error: "Invalid player address format"

**Solution:**

Ensure player address:
- Starts with `0x`
- Is exactly 42 characters long
- Contains only hexadecimal characters (0-9, a-f, A-F)

---

## Database Issues

### Issue: Markets Not Appearing in Database

**Symptoms:**
- Backend logs show "Market created successfully"
- Database query returns no results

**Diagnosis:**

1. **Check Database Connection:**
   ```bash
   # Test Supabase connection
   curl -H "apikey: $SUPABASE_ANON_KEY" \
     "$SUPABASE_URL/rest/v1/infofi_markets?select=*&limit=1"
   ```

2. **Verify Table Schema:**
   ```sql
   -- In Supabase SQL editor
   SELECT column_name, data_type 
   FROM information_schema.columns 
   WHERE table_name = 'infofi_markets';
   ```

**Solution:**

Run migrations if schema is missing:
```bash
cd backend/src/db/migrations
# Apply migrations in order
psql $DATABASE_URL < 001_initial_schema.sql
psql $DATABASE_URL < 002_infofi_markets.sql
# etc.
```

---

## Testing Issues

### Issue: E2E Test Fails at Market Creation Step

**Symptoms:**
- Tickets purchased successfully
- Backend shows no market creation attempt
- Test times out waiting for market

**Solution:**

1. **Ensure Backend is Running During Test:**
   ```bash
   # In separate terminal
   cd backend && npm run dev
   ```

2. **Wait for Backend to Process Event:**
   ```javascript
   // In test, add delay after ticket purchase
   await new Promise(resolve => setTimeout(resolve, 5000)); // 5 second wait
   ```

3. **Check Test Uses Correct Addresses:**
   ```javascript
   // Verify .env.test has correct contract addresses
   console.log('CURVE_ADDRESS:', process.env.CURVE_ADDRESS);
   console.log('INFOFI_FACTORY_ADDRESS:', process.env.INFOFI_FACTORY_ADDRESS);
   ```

---

## Performance Issues

### Issue: Market Creation Takes Too Long (>2 minutes)

**Expected:** 30-60 seconds
**Actual:** >2 minutes

**Diagnosis:**

1. **Check Network Congestion:**
   ```bash
   cast gas-price --rpc-url $RPC_URL
   # If >100 gwei, network is congested
   ```

2. **Check Backend Processing Time:**
   - Look for "Market creation took X ms" in logs
   - Should be <5000ms for transaction submission

**Solutions:**

#### Network Congestion

Wait for gas prices to decrease, or increase `MAX_GAS_PRICE` in backend config

#### Backend Overloaded

- Check CPU/memory usage
- Consider scaling backend horizontally
- Implement queue system for high-volume periods

---

## Recovery Procedures

### Recovering from Backend Downtime

**Scenario:** Backend was down for 2 hours, missed several ticket purchases

**Recovery Steps:**

1. **Backend Automatically Scans on Startup:**
   ```bash
   cd backend && npm run dev
   # Watch logs for "Scanning for missed PositionUpdate events"
   ```

2. **Verify Historical Scan Completed:**
   ```bash
   # Check event_processing_state table
   psql $DATABASE_URL -c "SELECT * FROM event_processing_state WHERE event_type = 'position_updates';"
   ```

3. **Manual Recovery if Needed:**
   ```bash
   # Run historical scan script
   node backend/scripts/scan-historical-events.js \
     --from-block 12345 \
     --to-block 12567
   ```

---

### Recovering from Failed Market Creation

**Scenario:** Market creation transaction failed due to gas spike, player is above threshold but has no market

**Recovery Steps:**

1. **Use Admin Panel:**
   - Navigate to "Manual Markets" tab
   - Select season
   - Enter player address
   - Click "Create Market"

2. **Or Use Backend API:**
   ```bash
   curl -X POST http://localhost:3000/api/admin/create-market \
     -H "Content-Type: application/json" \
     -d '{"seasonId": 1, "playerAddress": "0x..."}'
   ```

---

## Monitoring & Alerts

### Recommended Monitoring Setup

1. **Backend Wallet Balance Alert:**
   - Alert when balance < 0.2 ETH
   - Critical alert when balance < 0.1 ETH

2. **Market Creation Latency:**
   - Warning if >60 seconds
   - Alert if >120 seconds

3. **Failed Transaction Rate:**
   - Alert if >10% of market creation attempts fail

4. **Event Processing Lag:**
   - Alert if `last_block` is >100 blocks behind current block

---

## Common Error Messages

### "AccessControlUnauthorizedAccount"

**Meaning:** Backend wallet doesn't have BACKEND_ROLE

**Fix:** Grant role using admin wallet (see above)

---

### "Curve: no fees"

**Meaning:** Trying to extract fees when none accumulated

**Fix:** Wait for users to buy/sell tickets to accumulate fees

---

### "Market already exists for this player"

**Meaning:** Attempting to create duplicate market

**Fix:** Check database for existing market before creating

---

### "Insufficient funds for gas"

**Meaning:** Backend wallet needs more ETH

**Fix:** Fund backend wallet (see above)

---

## Debug Mode

### Enable Verbose Logging

```bash
# In .env
DEBUG=true
LOG_LEVEL=debug

# Restart backend
cd backend && npm run dev
```

### Check Specific Event Logs

```bash
# Filter for market creation events
tail -f backend/logs/server.log | grep "Market creation"

# Filter for position updates
tail -f backend/logs/server.log | grep "PositionUpdate"
```

---

## Contact & Support

For issues not covered in this guide:

1. Check GitHub Issues: [github.com/your-repo/issues](https://github.com)
2. Review contract events on block explorer
3. Check Supabase logs for database errors
4. Review backend logs for detailed error messages

---

## Quick Reference

### Key Environment Variables

```bash
BACKEND_WALLET_ADDRESS=0x...    # Backend wallet for market creation
BACKEND_WALLET_PRIVATE_KEY=0x... # Private key (keep secure!)
INFOFI_FACTORY_ADDRESS=0x...    # InfoFiMarketFactory contract
MAX_GAS_PRICE=100000000000      # 100 gwei max
RETRY_DELAY=5000                # 5 seconds between retries
MAX_RETRIES=3                   # Maximum retry attempts
```

### Key Contract Addresses

```bash
# Check .env file for current deployment
cat .env | grep ADDRESS
```

### Useful Commands

```bash
# Check backend wallet balance
cast balance $BACKEND_WALLET_ADDRESS --rpc-url $RPC_URL

# Check if role granted
cast call $INFOFI_FACTORY_ADDRESS "hasRole(bytes32,address)" $(cast keccak "BACKEND_ROLE") $BACKEND_WALLET_ADDRESS --rpc-url $RPC_URL

# View recent events
cast logs --address $CURVE_ADDRESS --from-block latest --rpc-url $RPC_URL

# Fund backend wallet
cast send $BACKEND_WALLET_ADDRESS --value 1ether --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```
