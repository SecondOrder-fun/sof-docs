# Raffle Transaction History - Deployment Steps

## ✅ Implementation Complete

All backend code has been implemented. Follow these steps to deploy:

## Step 1: Run Database Migration

### Option A: Via Supabase Dashboard (Recommended)

1. Go to https://supabase.com/dashboard
2. Select your project
3. Navigate to **SQL Editor**
4. Click **New Query**
5. Copy the entire contents of `/backend/migrations/001_raffle_transactions.sql`
6. Paste into the SQL editor
7. Click **Run** (or press Cmd/Ctrl + Enter)
8. Verify output shows: `Migration complete! Created X season partitions`

### Option B: Via Supabase CLI

```bash
# From project root
cd backend/migrations
supabase db push 001_raffle_transactions.sql
```

## Step 2: Verify Database Setup

Run this query in Supabase SQL Editor to verify:

```sql
-- Check if table exists
SELECT tablename FROM pg_tables
WHERE tablename LIKE 'raffle_transactions%'
ORDER BY tablename;

-- Check if materialized view exists
SELECT matviewname FROM pg_matviews
WHERE matviewname = 'user_raffle_positions';

-- Check if functions exist
SELECT routine_name FROM information_schema.routines
WHERE routine_name IN ('create_raffle_tx_partition', 'refresh_user_positions', 'populate_raffle_tx_player_id');
```

Expected output:

- Multiple `raffle_transactions_season_X` tables (one per season)
- `user_raffle_positions` materialized view
- 3 functions listed

## Step 3: Configure Environment Variables

Ensure your `.env` has the bonding curve address:

```bash
# For testnet
BONDING_CURVE_ADDRESS_TESTNET=0x...

# Or for local
BONDING_CURVE_ADDRESS_LOCAL=0x...
```

## Step 4: Deploy Backend

### Local Testing

```bash
npm run dev:backend
```

Expected logs:

```
✅ Server listening on port 3000
Mounted /api/raffle
🎫 Starting historical transaction sync...
✅ Historical transaction sync complete: X new transactions
```

### Production Deployment (Railway)

```bash
git add .
git commit -m "feat: add raffle transaction history system"
git push origin main
```

Railway will auto-deploy. Check logs for:

- `Mounted /api/raffle`
- `Historical transaction sync complete`

## Step 5: Test API Endpoints

### Get User Transactions

```bash
curl "https://your-backend.railway.app/api/raffle/transactions/0xYourAddress/1"
```

Expected response:

```json
{
  "transactions": [
    {
      "id": 1,
      "season_id": 1,
      "user_address": "0x...",
      "transaction_type": "BUY",
      "ticket_amount": "1000",
      "sof_amount": "100",
      "price_per_ticket": "0.1",
      "tx_hash": "0x...",
      "block_timestamp": "2025-12-07T10:30:00Z",
      "tickets_before": "0",
      "tickets_after": "1000"
    }
  ]
}
```

### Get User Position Summary

```bash
curl "https://your-backend.railway.app/api/raffle/positions/0xYourAddress/1"
```

Expected response:

```json
{
  "position": {
    "user_address": "0x...",
    "season_id": 1,
    "transaction_count": 3,
    "total_bought": "1500",
    "total_sold": "200",
    "current_tickets": "1300",
    "total_sof_spent": "150.5",
    "avg_buy_price": "0.1003",
    "first_transaction_at": "2025-12-07T10:30:00Z",
    "last_transaction_at": "2025-12-07T11:45:00Z"
  }
}
```

## Step 6: Frontend Implementation (Next Phase)

Once backend is verified working, implement frontend components:

1. Create `TransactionHistory.jsx` component
2. Update `AccountPage.jsx` to show transaction history
3. Add real-time updates via React Query

See `/docs/03-development/RAFFLE_TRANSACTION_HISTORY_IMPLEMENTATION_PLAN.md` for full frontend implementation details.

## Troubleshooting

### Issue: "Table already exists" error

**Solution:** The migration is idempotent. If you need to re-run:

```sql
-- Drop everything and start fresh (CAUTION: loses data)
DROP TABLE IF EXISTS raffle_transactions CASCADE;
DROP MATERIALIZED VIEW IF EXISTS user_raffle_positions CASCADE;
DROP FUNCTION IF EXISTS create_raffle_tx_partition CASCADE;
DROP FUNCTION IF EXISTS refresh_user_positions CASCADE;
DROP FUNCTION IF EXISTS populate_raffle_tx_player_id CASCADE;

-- Then re-run the migration
```

### Issue: "No transactions synced"

**Possible causes:**

1. No PositionUpdate events exist on-chain yet
2. Bonding curve address not configured
3. RPC connection issues

**Debug:**

```bash
# Check environment
echo $BONDING_CURVE_ADDRESS_TESTNET

# Check backend logs
railway logs --tail
```

### Issue: "Materialized view is empty"

**Solution:** Manually refresh:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY user_raffle_positions;
```

Or via API:

```bash
curl -X POST "https://your-backend.railway.app/api/raffle/admin/refresh-positions"
```

## Verification Checklist

- [ ] Database migration ran successfully
- [ ] Partitions created for all seasons
- [ ] Materialized view exists
- [ ] Backend deployed and running
- [ ] API endpoints respond correctly
- [ ] Historical sync completed
- [ ] Transaction data visible in Supabase
- [ ] Ready for frontend implementation

## Next Steps

1. ✅ Complete backend deployment (this document)
2. ⏭️ Implement frontend components
3. ⏭️ Test in production with real users
4. ⏭️ Monitor performance and optimize queries
5. ⏭️ Add analytics and insights features

## Support

If you encounter issues:

1. Check backend logs: `railway logs --tail`
2. Check Supabase logs in dashboard
3. Verify RPC connectivity
4. Check environment variables
5. Review implementation plan for details
