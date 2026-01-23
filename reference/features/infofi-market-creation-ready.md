# InfoFi Market Creation - Ready for Testing

## Summary of Changes

The backend has been updated to automatically create InfoFi markets when players cross the 1% of max supply threshold. Here's what was implemented:

## Architecture

```text
SeasonStarted Event (from Raffle contract)
    ↓
SeasonStartedListener (watches Raffle contract)
    ↓
Calls onSeasonCreated callback
    ↓
Starts PositionUpdateListener for that season's bonding curve
    ↓
PositionUpdateListener watches for PositionUpdate events
    ↓
When player crosses 1% threshold:
    - Fetches all participants in season
    - Calculates win probabilities for all players
    - Creates/updates markets in database
```

## Key Changes Made

### 1. Backend Server Integration (`backend/fastify/server.js`)

- Added import for `startPositionUpdateListener` and `SOFBondingCurveAbi`
- Created `onSeasonCreated` callback that starts a position update listener when a season begins
- Wired callback into `startSeasonStartedListener`
- Updated graceful shutdown to stop all position update listeners

### 2. How It Works

**When a season starts:**

1. `SeasonStartedListener` detects `SeasonStarted` event
2. Retrieves bonding curve and raffle token addresses from contract
3. Calls `onSeasonCreated` callback with season data
4. Backend starts `PositionUpdateListener` for that season's bonding curve

**When a player buys tickets:**

1. `PositionUpdateListener` detects `PositionUpdate` event on bonding curve
2. Fetches all participants in the season
3. Calculates win probability for each player: `(ticketCount * 10000) / totalTickets`
4. For players crossing 1% threshold:
   - Creates new market in `infofi_markets` table if doesn't exist
   - Updates market with new probability
5. All players' probabilities updated on each transaction

## Testing

Follow the comprehensive test guide: `INFOFI_MARKET_CREATION_TEST_GUIDE.md`

**Quick Start:**

```bash
# 1. Deploy contracts
cd contracts && forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 --broadcast

# 2. Update environment
cd .. && node scripts/update-env-addresses.js && node scripts/copy-abis.js

# 3. Create season
export $(cat .env | xargs) && cd contracts && forge script script/CreateSeason.s.sol --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY --broadcast

# 4. Start season (after 60 seconds)
cast send $RAFFLE_ADDRESS_LOCAL "startSeason(uint256)" 1 --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY

# 5. Buy tickets to cross 1% threshold
cast send $SOF_ADDRESS_LOCAL "approve(address,uint256)" $CURVE_ADDRESS 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY
cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" 11000 3500000000000000000000 --rpc-url $RPC_URL_LOCAL --private-key $PRIVATE_KEY

# 6. Check backend logs for market creation
# Look for: "✅ PositionUpdate: Season 1, Player 0x... (0 → 11000 tickets)"
# And: "Updated 1 markets"

# 7. Verify in database
# SELECT * FROM infofi_markets WHERE season_id = 1;
```

## Expected Behavior

### Backend Logs

When a season starts:

```text
✅ SeasonStarted Event: Season 1 has started
   BondingCurve: 0x94099942864EA81cCF197E9D71ac53310b1468D8
   RaffleToken: 0x06B1D212B8da92b83AF328De5eef4E211Da02097
🎧 Starting PositionUpdate listener for season 1
✅ PositionUpdate listener started for season 1
```

When a player buys tickets crossing 1% threshold:

```text
📊 PositionUpdate Event: Season 1, Player 0x70997970C51812dc3A010C7d01b50e0d17dc79C8, Tickets: 0 → 11000, Total: 11000
   Fetching participants for season 1...
   Found 1 participants in season 1
   Fetching positions for 1 players...
   0x70997970C51812dc3A010C7d01b50e0d17dc79C8: 11000 tickets
   Updating probabilities in database...
✅ PositionUpdate: Season 1, Player 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (0 → 11000 tickets)
   Total supply: 11000 | Updated 1 markets | Player probability: 10000 bps
```

### Database

Markets created in `infofi_markets` table:

- `season_id`: Season number
- `player_address`: Player's wallet address
- `market_type`: 'WINNER_PREDICTION'
- `initial_probability_bps`: Probability when market created (basis points)
- `current_probability_bps`: Current probability (updates on each transaction)
- `is_active`: true
- `is_settled`: false

## Files Modified

- `backend/fastify/server.js` - Added listener orchestration
- `backend/src/listeners/seasonStartedListener.js` - Already had callback support
- `backend/src/listeners/positionUpdateListener.js` - Already implemented market creation
- `backend/shared/supabaseClient.js` - Already had `updateAllPlayerProbabilities` method

## Next Steps

1. Restart backend: `npm run dev:backend`
2. Follow test guide to verify market creation
3. Check frontend displays markets with correct probabilities
4. Monitor backend logs for any errors
5. Verify database entries match expected values

## Troubleshooting

**Backend not starting PositionUpdateListener:**
- Check `RAFFLE_ADDRESS_LOCAL` is set in `.env`
- Verify backend restarted after code changes
- Check for errors in backend logs

**Markets not created:**
- Verify player position > 1% threshold
- Check `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are set
- Look for database errors in backend logs

**Probabilities not updating:**
- Verify `updateAllPlayerProbabilities` is called (check logs)
- Ensure all participants fetched correctly
- Check probability calculation math

