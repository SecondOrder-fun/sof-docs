# Users Page Fix - Using Database Instead of Blockchain

## Issue

The Users page was trying to fetch players directly from the blockchain via the `InfoFiMarketFactory.getSeasonPlayers()` function, which was slow and inefficient.

## Solution

Changed the architecture to use the **Supabase database** as the source of truth for player data, which is already being populated by the backend event listeners.

## Changes Made

### 1. Added Backend API Endpoint (`backend/fastify/routes/userRoutes.js`)

Created a new `GET /api/users` endpoint that queries the `players` table in Supabase:

```javascript
// Get all players (addresses that have participated in seasons)
fastify.get('/', async (request, reply) => {
  try {
    const { db } = await import('../../shared/supabaseClient.js');
    
    // Query all players from database
    const { data: players, error } = await db.client
      .from('players')
      .select('address')
      .order('created_at', { ascending: false });
    
    if (error) {
      fastify.log.error({ error }, 'Failed to fetch players from database');
      return reply.status(500).send({ error: 'Failed to fetch players' });
    }
    
    // Return array of addresses
    const addresses = (players || []).map(p => p.address);
    reply.send({ players: addresses, count: addresses.length });
  } catch (error) {
    fastify.log.error(error);
    return reply.status(500).send({ error: 'Failed to fetch players' });
  }
});
```

### 2. Updated Frontend Component (`src/routes/UsersIndex.jsx`)

Simplified the component to fetch from the backend API instead of querying blockchain:

**Before:**
- Used `useAllSeasons()` hook
- Looped through all seasons
- Called `getSeasonPlayersOnchain()` for each season
- Aggregated results from blockchain

**After:**
- Simple `fetch()` call to `http://localhost:3000/api/users`
- Displays players from database
- Much faster and more reliable

```javascript
useEffect(() => {
  let cancelled = false;
  async function load() {
    setLoading(true);
    setError(null);
    
    try {
      // Fetch players from backend API (which queries Supabase database)
      const response = await fetch('http://localhost:3000/api/users');
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const data = await response.json();
      
      if (!cancelled) {
        setPlayers(data.players || []);
      }
    } catch (err) {
      if (!cancelled) {
        setError(err.message);
        setPlayers([]);
      }
    } finally {
      if (!cancelled) setLoading(false);
    }
  }
  
  load();
  return () => { cancelled = true; };
}, []);
```

## How It Works

### Backend Event Listeners

The backend already has event listeners that populate the `players` table:

1. **`infofiListener.js`** - Listens to `MarketCreated` events from `InfoFiMarketFactory`
2. **`positionTrackerListener.js`** - Listens to `PositionSnapshot` events
3. **`raffleListener.js`** - Listens to `PositionUpdate` events

When a player buys tickets, these listeners automatically:
- Call `db.getOrCreatePlayerIdByAddress(playerAddress)`
- Insert the player into the `players` table if they don't exist
- Track their participation in seasons

### Database Schema

```sql
CREATE TABLE players (
  id BIGSERIAL PRIMARY KEY,
  address VARCHAR(42) NOT NULL UNIQUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Testing

### 1. Start Backend Server

```bash
npm run dev:backend
```

### 2. Verify API Endpoint

```bash
curl http://localhost:3000/api/users
```

Expected response:
```json
{
  "players": [
    "0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc",
    "0x70997970c51812dc3a010c7d01b50e0d17dc79c8",
    "0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266"
  ],
  "count": 3
}
```

### 3. Check Frontend

Navigate to `http://127.0.0.1:5173/users` and you should see the list of players.

## Benefits

✅ **Much faster** - Database query instead of multiple blockchain calls
✅ **More reliable** - No RPC failures or network issues
✅ **Consistent data** - Single source of truth
✅ **Better UX** - Instant loading, no blockchain delays
✅ **Scalable** - Can handle thousands of players efficiently
✅ **Already populated** - Backend listeners keep it up-to-date automatically

## Important Notes

- **Backend must be running** for the Users page to work
- **Redis must be running** for the backend to start (username service dependency)
- Players are automatically added to the database when they buy tickets
- The database is the source of truth, not the blockchain

## Related Files

- `backend/fastify/routes/userRoutes.js` - API endpoint
- `src/routes/UsersIndex.jsx` - Frontend component
- `backend/shared/supabaseClient.js` - Database service
- `backend/src/services/infofiListener.js` - Event listener that populates players table
