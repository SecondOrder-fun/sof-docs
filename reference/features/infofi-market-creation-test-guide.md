# InfoFi Market Creation Test Guide

## Overview

This guide walks you through testing the complete InfoFi market creation flow when a player crosses the 1% of max supply threshold.

**Key Components:**

- **SeasonStartedListener**: Watches for `SeasonStarted` events and starts a `PositionUpdateListener` for each season
- **PositionUpdateListener**: Watches for `PositionUpdate` events on the bonding curve and creates InfoFi markets when players cross 1% threshold
- **Database**: Stores markets in `infofi_markets` table with `season_id`, `player_address`, and `market_type`

## Prerequisites

Ensure all three services are running:

```bash
# Terminal 1: Anvil
anvil --gas-limit 30000000

# Terminal 2: Backend
npm run dev:backend

# Terminal 3: Frontend (optional, for visual verification)
npm run dev
```

## Test Flow

### Step 1: Deploy Fresh Contracts

```bash
cd contracts
forge script script/Deploy.s.sol \
  --rpc-url http://127.0.0.1:8545 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  --broadcast
```

Update environment:

```bash
cd ..
node scripts/update-env-addresses.js
node scripts/copy-abis.js
```

### Step 2: Create a Season

```bash
export $(cat .env | xargs)

cd contracts
forge script script/CreateSeason.s.sol \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY \
  --broadcast
```

**Expected Backend Logs:**

```
✅ SeasonStarted Event: Season 1 has started
   BondingCurve: 0x...
   RaffleToken: 0x...
🎧 Starting PositionUpdate listener for season 1
✅ PositionUpdate listener started for season 1
```

### Step 3: Wait for Season to Start

The season has a 60-second delay before it can be started. Wait, then:

```bash
cast send $RAFFLE_ADDRESS_LOCAL "startSeason(uint256)" 1 \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY
```

### Step 4: Calculate 1% Threshold

From the season creation logs, note the max supply. For a typical 100k/100-step season:

- Max supply: 1,000,000 tickets
- 1% threshold: 10,000 tickets

### Step 5: Buy Tickets to Cross Threshold

First, approve SOF:

```bash
cast send $SOF_ADDRESS_LOCAL "approve(address,uint256)" \
  $CURVE_ADDRESS \
  0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY
```

Then buy 11,000 tickets (to cross 1% threshold):

```bash
cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" \
  11000 \
  3500000000000000000000 \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY
```

**Expected Backend Logs:**

```
📊 PositionUpdate Event: Season 1, Player 0x70997970C51812dc3A010C7d01b50e0d17dc79C8, Tickets: 0 → 11000, Total: 11000
   Fetching participants for season 1...
   Found 1 participants in season 1
   Fetching positions for 1 players...
   0x70997970C51812dc3A010C7d01b50e0d17dc79C8: 11000 tickets
   Fetching positions for all 1 players...
   Updating probabilities in database...
✅ PositionUpdate: Season 1, Player 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (0 → 11000 tickets)
   Total supply: 11000 | Updated 1 markets | Player probability: 10000 bps
   Updated player probabilities:
     0x70997970C51812dc3A010C7d01b50e0d17dc79C8: 11000 tickets → 10000 bps
```

### Step 6: Verify Market Created in Database

Check Supabase for the new market:

```sql
SELECT * FROM infofi_markets
WHERE season_id = 1
  AND player_address = '0x70997970c51812dc3a010c7d01b50e0d17dc79c8'
  AND market_type = 'WINNER_PREDICTION';
```

**Expected Result:**

```text
id: 1
season_id: 1
player_address: 0x70997970c51812dc3a010c7d01b50e0d17dc79c8
player_id: NULL (will be populated later)
market_type: WINNER_PREDICTION
contract_address: NULL
initial_probability_bps: 10000
current_probability_bps: 10000
is_active: true
is_settled: false
created_at: 2025-10-25T...
updated_at: 2025-10-25T...
```

### Step 7: Buy More Tickets from Second Account

Transfer SOF to account[1]:

```bash
ACCOUNT1_ADDRESS=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

cast send $SOF_ADDRESS_LOCAL "transfer(address,uint256)" \
  $ACCOUNT1_ADDRESS \
  5000000000000000000000 \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $PRIVATE_KEY
```

Approve and buy 6,000 tickets from account[1]:

```bash
ACCOUNT1_PRIVATE_KEY=0x47e179ec197488593b187f80a00eb0da4dc3c1ed112a5dc6986d0497f52d84fd

cast send $SOF_ADDRESS_LOCAL "approve(address,uint256)" \
  $CURVE_ADDRESS \
  0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $ACCOUNT1_PRIVATE_KEY

cast send $CURVE_ADDRESS "buyTokens(uint256,uint256)" \
  6000 \
  2000000000000000000000 \
  --rpc-url $RPC_URL_LOCAL \
  --private-key $ACCOUNT1_PRIVATE_KEY
```

**Expected Backend Logs:**

```
📊 PositionUpdate Event: Season 1, Player 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266, Tickets: 0 → 6000, Total: 17000
   Fetching participants for season 1...
   Found 2 participants in season 1
   Fetching positions for 2 players...
   0x70997970C51812dc3A010C7d01b50e0d17dc79C8: 11000 tickets
   0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266: 6000 tickets
   Updating probabilities in database...
✅ PositionUpdate: Season 1, Player 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (0 → 6000 tickets)
   Total supply: 17000 | Updated 2 markets | Player probability: 3529 bps
   Updated player probabilities:
     0x70997970C51812dc3A010C7d01b50e0d17dc79C8: 11000 tickets → 6470 bps
     0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266: 6000 tickets → 3529 bps
```

**Key Observation:** Both players' probabilities were updated! Account[0] went from 10000 bps (100%) to 6470 bps (64.7%) because the total supply increased.

### Step 8: Verify Both Markets in Database

```sql
SELECT player_address, initial_probability_bps, current_probability_bps
FROM infofi_markets
WHERE season_id = 1
ORDER BY created_at;
```

**Expected Result:**

```text
player_address                           | initial_probability_bps | current_probability_bps
0x70997970c51812dc3a010c7d01b50e0d17dc79c8 | 10000                   | 6470
0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 | 3529                    | 3529
```

### Step 9: Verify Frontend Display (Optional)

Navigate to the raffle page in the frontend. You should see:

- Season 1 active
- Two players with their positions
- InfoFi markets displayed with current probabilities
- Real-time odds updating as prices change

## Troubleshooting

### Backend Not Listening

**Symptom:** No "🎧 Starting PositionUpdate listener" message after season starts

**Solution:**

1. Check `RAFFLE_ADDRESS_LOCAL` is set in `.env`
2. Verify backend restarted after code changes
3. Check backend logs for errors during listener startup

### Markets Not Created

**Symptom:** PositionUpdate event logged but no markets in database

**Solution:**

1. Verify player position > 1% threshold: `ticketCount > (maxSupply / 100)`
2. Check database connection: `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` set
3. Look for database errors in backend logs

### Probabilities Not Updating

**Symptom:** Markets created but probabilities don't change on subsequent buys

**Solution:**

1. Verify `updateAllPlayerProbabilities` is being called (check logs)
2. Check that all players are being fetched: `getParticipants` should return all addresses
3. Verify probability calculation: `(ticketCount * 10000) / totalTickets`

## Success Criteria

✅ SeasonStarted listener fires and logs season details

✅ PositionUpdateListener starts automatically for new season

✅ First buy crossing 1% threshold creates market in database

✅ Subsequent buys update all players' probabilities

✅ Markets visible on frontend with correct odds

✅ Probabilities sum to 10000 (100%) across all players

## Key Files

- `backend/fastify/server.js` - Listener orchestration
- `backend/src/listeners/seasonStartedListener.js` - Season discovery
- `backend/src/listeners/positionUpdateListener.js` - Market creation & probability updates
- `backend/shared/supabaseClient.js` - Database operations (`updateAllPlayerProbabilities`)
