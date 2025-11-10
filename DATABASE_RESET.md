# Database Reset for Local Development

## Problem

When Anvil (local Ethereum node) restarts, the blockchain state resets to genesis, but the Supabase database retains old data. This causes:

- ❌ Backend queries return stale season/raffle data
- ❌ Event listeners skip new events (wrong block numbers)
- ❌ Frontend shows non-existent contracts
- ❌ Prediction markets don't sync (no new data)

## Solution

Use `npm run reset:local-db` to clear all application data from Supabase and Redis before redeploying contracts.

## Usage

### Full Reset Workflow

```bash
# Terminal 1: Start fresh Anvil
npm run anvil

# Terminal 2: Reset database and deploy
npm run reset:local-db
npm run anvil:deploy
npm run dev:backend

# Terminal 3: Start frontend
npm run dev:frontend
```

### Quick Reset (Anvil already running)

```bash
npm run reset:local-db
npm run deploy:anvil
```

## What Gets Cleared

The script clears the following in order:

1. **Redis cache** - All cached data
2. **infofi_positions** - User prediction market positions
3. **infofi_winnings** - Claimable winnings
4. **market_pricing_cache** - Real-time pricing data
5. **arbitrage_opportunities** - Detected arbitrage opportunities
6. **infofi_markets** - All prediction markets
7. **players** - Player records
8. **raffles** - Season/raffle records
9. **event_processing_state** - Event listener block tracking

## Safety Features

- **Local only**: Script refuses to run on TESTNET or MAINNET
- **Force flag**: Use `--force` to override (not recommended)
- **Non-fatal errors**: Missing tables are skipped gracefully
- **Sequence reset**: Auto-resets ID sequences to start from 1

## Troubleshooting

### Script fails with "address already in use"

Backend is still running. Kill it first:

```bash
npm run kill:zombies
npm run reset:local-db
```

### Redis connection error

Redis is not running. Start it:

```bash
redis-server
```

Or skip Redis clearing (non-fatal):

```bash
# Script will continue even if Redis fails
npm run reset:local-db
```

### Table not found errors

Some tables may not exist yet. The script skips them automatically:

```
⚠️  Table event_processing_state does not exist (skipping)
```

This is normal and safe.

## Manual Database Inspection

Check if reset worked:

```bash
# Check raffles
curl -s http://localhost:3000/api/raffles | jq '.raffles | length'

# Check markets
curl -s http://localhost:3000/api/infofi/markets | jq '.markets'

# Check Redis
redis-cli DBSIZE
```

All should return 0 or empty after reset.

## Integration with Deployment

### Current Workflow (Manual)

```bash
npm run reset:local-db  # Manual step
npm run anvil:deploy    # Deploys contracts
```

### Future Enhancement (Automated)

Could integrate into `anvil:deploy` script:

```json
"anvil:deploy": "npm run reset:local-db && [existing deploy command]"
```

**Not implemented yet** to give developers control over when to reset.

## Script Location

- **Script**: `scripts/reset-local-db.js`
- **npm command**: `npm run reset:local-db`
- **Source**: Uses Supabase client + redis-cli

## Related Documentation

- [ABI Management](./ABI_MANAGEMENT.md) - How ABIs are synced
- [Project Requirements](../instructions/project-requirements.md) - System architecture
- [Data Schema](../instructions/data-schema.md) - Database structure
