# InfoFi Market Creation - Quick Start

## What Changed

Backend now automatically creates InfoFi markets when players cross the 1% threshold.

## How It Works

1. **Season Starts** → SeasonStartedListener detects event
2. **Listener Starts** → PositionUpdateListener begins watching bonding curve
3. **Player Buys** → PositionUpdate event fires
4. **Market Created** → If player crosses 1%, market created in database
5. **Probabilities Update** → All players' odds recalculated on each transaction

## Quick Test (5 minutes)

```bash
# Terminal 1: Anvil (if not running)
anvil --gas-limit 30000000

# Terminal 2: Backend
npm run dev:backend

# Terminal 3: Deploy & test
export $(cat .env | xargs)

# Deploy
cd contracts
forge script script/Deploy.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast
cd ..
node scripts/update-env-addresses.js

# Create season
cd contracts
forge script script/CreateSeason.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast

# Wait 60 seconds, then start season
sleep 61
cast send $RAFFLE_ADDRESS_LOCAL "startSeason(uint256)" 1 --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY

# Buy tickets (crosses 1% threshold)
cast send $SOF_ADDRESS_LOCAL "approve(address,uint256)" $CURVE_ADDRESS 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY
cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" 11000 3500000000000000000000 --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY
```

## What to Look For

**Backend Logs:**
- ✅ "SeasonStarted Event: Season 1 has started"
- ✅ "Starting PositionUpdate listener for season 1"
- ✅ "PositionUpdate listener started for season 1"
- ✅ "PositionUpdate: Season 1, Player 0x... (0 → 11000 tickets)"
- ✅ "Updated 1 markets"

**Database:**
```sql
SELECT * FROM infofi_markets WHERE season_id = 1;
```

Should show 1 market with:
- `player_address`: Your address
- `market_type`: 'WINNER_PREDICTION'
- `current_probability_bps`: 10000 (100%)

## Files Modified

- `backend/fastify/server.js` - Wired up listener orchestration

## Documentation

- `INFOFI_MARKET_CREATION_READY.md` - Full overview
- `INFOFI_MARKET_CREATION_TEST_GUIDE.md` - Detailed test guide with 9 steps

## Next: Buy from Second Account

To see probabilities update:

```bash
# Transfer SOF to account[1]
cast send $SOF_ADDRESS_LOCAL "transfer(address,uint256)" 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 5000000000000000000000 --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY

# Account[1] buys 6000 tickets
ACCOUNT1_PRIVATE_KEY=0x47e179ec197488593b187f80a00eb0da4dc3c1ed112a5dc6986d0497f52d84fd
cast send $SOF_ADDRESS_LOCAL "approve(address,uint256)" $CURVE_ADDRESS 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff --rpc-url $RPC_URL_LOCAL --private-key $ACCOUNT1_PRIVATE_KEY
cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" 6000 2000000000000000000000 --rpc-url $RPC_URL_LOCAL --private-key $ACCOUNT1_PRIVATE_KEY
```

**Expected:** Both players' probabilities update in database:
- Account[0]: 10000 → 6470 bps (64.7%)
- Account[1]: 3529 bps (35.3%)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| No "Starting PositionUpdate listener" log | Restart backend after code changes |
| Markets not created | Check player position > 1% threshold |
| Database connection error | Verify SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY |
| Probabilities not updating | Check backend logs for updateAllPlayerProbabilities calls |

