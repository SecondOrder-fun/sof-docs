# Raffle Transaction History Implementation Plan

## Executive Summary

This plan implements a **persistent, scalable transaction history system** for raffle ticket purchases using PostgreSQL best practices including:

- **Event sourcing pattern** for immutable transaction records
- **Partitioning by season** for performance at scale
- **Materialized views** for fast aggregated queries
- **Idempotent writes** using transaction hashes
- **Automatic backfill** from blockchain events on startup

## Problem Statement

**Current Issue:** Portfolio page shows no ticket purchase history (see screenshot)

- Users can't see their transaction history
- No persistent record of ticket purchases
- All data fetched from chain in real-time (slow, unreliable)

**Requirements:**

1. ✅ Persistent storage in Supabase (not chain-dependent)
2. ✅ Fast queries for portfolio display
3. ✅ Scalable to millions of transactions
4. ✅ Historical backfill from existing on-chain data
5. ✅ Real-time updates as new transactions occur

## Database Architecture

### Design Pattern: Event Sourcing + Partitioning

**Why Event Sourcing?**

- Immutable transaction log (audit trail)
- Can reconstruct state at any point in time
- Natural fit for blockchain events
- Supports time-travel queries

**Why Partitioning?**

- Prevents single monolithic table from becoming unwieldy
- Queries naturally filter by season (most common access pattern)
- Easy to archive old seasons
- Better index performance

### Schema Design

#### Core Table: `raffle_transactions` (Partitioned by Season)

```sql
-- Parent table (partitioned by season_id)
CREATE TABLE raffle_transactions (
    id BIGSERIAL,
    season_id BIGINT NOT NULL,
    user_address VARCHAR(42) NOT NULL,
    player_id BIGINT REFERENCES players(id),

    -- Transaction details
    transaction_type VARCHAR(20) NOT NULL, -- 'BUY', 'SELL', 'CLAIM', 'TRANSFER'
    ticket_amount NUMERIC NOT NULL, -- Positive for buy, negative for sell
    sof_amount NUMERIC NOT NULL, -- Amount of SOF spent/received
    price_per_ticket NUMERIC, -- Calculated price

    -- Blockchain data
    tx_hash VARCHAR(66) NOT NULL UNIQUE, -- For idempotency
    block_number BIGINT NOT NULL,
    block_timestamp TIMESTAMPTZ NOT NULL,

    -- Position tracking
    tickets_before NUMERIC NOT NULL DEFAULT 0,
    tickets_after NUMERIC NOT NULL,

    -- Metadata
    created_at TIMESTAMPTZ DEFAULT NOW(),

    PRIMARY KEY (id, season_id) -- Composite key for partitioning
) PARTITION BY RANGE (season_id);

-- Create indexes on parent table (inherited by partitions)
CREATE INDEX idx_raffle_tx_user ON raffle_transactions(user_address, season_id, block_timestamp DESC);
CREATE INDEX idx_raffle_tx_hash ON raffle_transactions(tx_hash);
CREATE INDEX idx_raffle_tx_block ON raffle_transactions(block_number);
CREATE INDEX idx_raffle_tx_player ON raffle_transactions(player_id, season_id);

-- Trigger to auto-populate player_id from user_address
CREATE OR REPLACE FUNCTION populate_raffle_tx_player_id()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.player_id IS NULL AND NEW.user_address IS NOT NULL THEN
        SELECT id INTO NEW.player_id
        FROM players
        WHERE address = NEW.user_address;

        -- If player doesn't exist, create them
        IF NEW.player_id IS NULL THEN
            INSERT INTO players (address, created_at)
            VALUES (NEW.user_address, NOW())
            ON CONFLICT (address) DO UPDATE SET updated_at = NOW()
            RETURNING id INTO NEW.player_id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER raffle_tx_player_id_trigger
    BEFORE INSERT ON raffle_transactions
    FOR EACH ROW
    EXECUTE FUNCTION populate_raffle_tx_player_id();
```

#### Partition Creation Strategy

```sql
-- Function to create partition for a season
CREATE OR REPLACE FUNCTION create_raffle_tx_partition(season_num BIGINT)
RETURNS VOID AS $$
DECLARE
    partition_name TEXT;
BEGIN
    partition_name := 'raffle_transactions_season_' || season_num;

    -- Check if partition already exists
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables
        WHERE tablename = partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF raffle_transactions
             FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            season_num,
            season_num + 1
        );

        RAISE NOTICE 'Created partition: %', partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Create partitions for existing seasons (run during migration)
DO $$
DECLARE
    season_record RECORD;
BEGIN
    FOR season_record IN
        SELECT DISTINCT id FROM seasons ORDER BY id
    LOOP
        PERFORM create_raffle_tx_partition(season_record.id);
    END LOOP;
END $$;

-- Trigger to auto-create partition when new season is created
CREATE OR REPLACE FUNCTION auto_create_raffle_tx_partition()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM create_raffle_tx_partition(NEW.id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER season_partition_trigger
    AFTER INSERT ON seasons
    FOR EACH ROW
    EXECUTE FUNCTION auto_create_raffle_tx_partition();
```

#### Materialized View: User Position Summary

```sql
-- Aggregated view for fast portfolio queries
CREATE MATERIALIZED VIEW user_raffle_positions AS
SELECT
    user_address,
    player_id,
    season_id,
    COUNT(*) as transaction_count,
    SUM(CASE WHEN transaction_type = 'BUY' THEN ticket_amount ELSE 0 END) as total_bought,
    SUM(CASE WHEN transaction_type = 'SELL' THEN ABS(ticket_amount) ELSE 0 END) as total_sold,
    SUM(ticket_amount) as current_tickets, -- Net position
    SUM(sof_amount) as total_sof_spent,
    AVG(CASE WHEN transaction_type = 'BUY' THEN price_per_ticket ELSE NULL END) as avg_buy_price,
    MIN(block_timestamp) as first_transaction_at,
    MAX(block_timestamp) as last_transaction_at,
    ARRAY_AGG(tx_hash ORDER BY block_timestamp) as transaction_hashes
FROM raffle_transactions
GROUP BY user_address, player_id, season_id;

-- Indexes for fast lookups
CREATE UNIQUE INDEX idx_user_raffle_pos_unique ON user_raffle_positions(user_address, season_id);
CREATE INDEX idx_user_raffle_pos_player ON user_raffle_positions(player_id, season_id);

-- Refresh strategy (called after each transaction batch)
CREATE OR REPLACE FUNCTION refresh_user_positions(season_num BIGINT DEFAULT NULL)
RETURNS VOID AS $$
BEGIN
    IF season_num IS NULL THEN
        REFRESH MATERIALIZED VIEW CONCURRENTLY user_raffle_positions;
    ELSE
        -- Partial refresh for specific season (requires custom logic)
        -- For now, refresh entire view
        REFRESH MATERIALIZED VIEW CONCURRENTLY user_raffle_positions;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

## Backend Implementation

### Phase 1: Service Layer

#### File: `backend/src/services/raffleTransactionService.js`

```javascript
import { publicClient } from "../lib/viemClient.js";
import { db } from "../config/supabaseClient.js";
import { queryLogsInChunks } from "../utils/blockRangeQuery.js";
import SOFBondingCurveAbi from "../abis/SOFBondingCurveAbi.js";

/**
 * Service for recording and querying raffle transaction history
 */
class RaffleTransactionService {
  /**
   * Record a transaction from PositionUpdate event (idempotent via tx_hash)
   */
  async recordTransaction({
    seasonId,
    userAddress,
    transactionType,
    ticketAmount,
    sofAmount,
    txHash,
    blockNumber,
    blockTimestamp,
    ticketsBefore,
    ticketsAfter,
  }) {
    try {
      // Calculate price per ticket
      const pricePerTicket =
        ticketAmount !== 0 ? Math.abs(sofAmount / ticketAmount) : null;

      const { data, error } = await db.client
        .from("raffle_transactions")
        .insert({
          season_id: seasonId,
          user_address: userAddress,
          transaction_type: transactionType,
          ticket_amount: ticketAmount,
          sof_amount: sofAmount,
          price_per_ticket: pricePerTicket,
          tx_hash: txHash,
          block_number: blockNumber,
          block_timestamp: blockTimestamp,
          tickets_before: ticketsBefore,
          tickets_after: ticketsAfter,
        })
        .select()
        .single();

      if (error) {
        // Check if duplicate (idempotency)
        if (error.code === "23505") {
          // Unique constraint violation on tx_hash
          return { alreadyRecorded: true, txHash };
        }
        throw error;
      }

      return { success: true, data };
    } catch (error) {
      console.error("Failed to record raffle transaction:", error);
      throw error;
    }
  }

  /**
   * Sync historical transactions for a season from blockchain
   */
  async syncSeasonTransactions(
    seasonId,
    bondingCurveAddress,
    fromBlock = null
  ) {
    try {
      // Get season's last synced block
      const { data: season } = await db.client
        .from("seasons")
        .select("created_at, last_tx_sync_block")
        .eq("id", seasonId)
        .single();

      if (!season) {
        return { error: "Season not found", seasonId };
      }

      const startBlock =
        fromBlock !== null
          ? BigInt(fromBlock)
          : BigInt(season.last_tx_sync_block || 0);

      const latestBlock = await publicClient.getBlockNumber();

      if (startBlock >= latestBlock) {
        return {
          success: true,
          recorded: 0,
          message: "Already up to date",
        };
      }

      // Get PositionUpdate event definition
      const positionUpdateEvent = SOFBondingCurveAbi.find(
        (item) => item.type === "event" && item.name === "PositionUpdate"
      );

      // Fetch events in chunks
      const logs = await queryLogsInChunks(
        publicClient,
        {
          address: bondingCurveAddress,
          event: {
            type: "event",
            name: "PositionUpdate",
            inputs: positionUpdateEvent.inputs,
          },
          fromBlock: startBlock,
          toBlock: latestBlock,
          // Filter by seasonId if possible (depends on event indexing)
        },
        10000n // 10k block chunks
      );

      // Filter logs for this season
      const seasonLogs = logs.filter(
        (log) => Number(log.args.seasonId) === seasonId
      );

      let recorded = 0;
      let skipped = 0;

      for (const log of seasonLogs) {
        const { player, oldTickets, newTickets } = log.args;

        // Get block timestamp
        const block = await publicClient.getBlock({
          blockNumber: log.blockNumber,
        });

        // Determine transaction type and amounts
        const ticketDelta = Number(newTickets) - Number(oldTickets);
        const transactionType = ticketDelta > 0 ? "BUY" : "SELL";

        // Get transaction to extract SOF amount (requires tx receipt)
        const tx = await publicClient.getTransaction({
          hash: log.transactionHash,
        });

        // Estimate SOF amount from value (simplified - may need bonding curve calculation)
        const sofAmount = Number(tx.value) / 1e18; // Convert wei to SOF

        try {
          const result = await this.recordTransaction({
            seasonId,
            userAddress: player,
            transactionType,
            ticketAmount: Math.abs(ticketDelta),
            sofAmount,
            txHash: log.transactionHash,
            blockNumber: Number(log.blockNumber),
            blockTimestamp: new Date(
              Number(block.timestamp) * 1000
            ).toISOString(),
            ticketsBefore: Number(oldTickets),
            ticketsAfter: Number(newTickets),
          });

          if (result.alreadyRecorded) {
            skipped++;
          } else {
            recorded++;
          }
        } catch (error) {
          console.error(
            `Failed to record tx ${log.transactionHash}:`,
            error.message
          );
        }
      }

      // Update last synced block
      await db.client
        .from("seasons")
        .update({
          last_tx_sync_block: latestBlock.toString(),
        })
        .eq("id", seasonId);

      // Refresh materialized view
      await this.refreshUserPositions(seasonId);

      return {
        success: true,
        recorded,
        skipped,
        totalEvents: seasonLogs.length,
        fromBlock: startBlock.toString(),
        toBlock: latestBlock.toString(),
      };
    } catch (error) {
      console.error("Error syncing season transactions:", error);
      throw error;
    }
  }

  /**
   * Sync all active seasons
   */
  async syncAllActiveSeasons(bondingCurveAddress) {
    const { data: seasons } = await db.client
      .from("seasons")
      .select("id, bonding_curve_address")
      .eq("is_active", true);

    const results = [];
    for (const season of seasons || []) {
      const curveAddress = season.bonding_curve_address || bondingCurveAddress;
      const result = await this.syncSeasonTransactions(season.id, curveAddress);
      results.push({ seasonId: season.id, ...result });
    }

    return results;
  }

  /**
   * Get user's transaction history for a season
   */
  async getUserTransactions(userAddress, seasonId, options = {}) {
    const {
      limit = 50,
      offset = 0,
      orderBy = "block_timestamp",
      order = "desc",
    } = options;

    const { data, error } = await db.client
      .from("raffle_transactions")
      .select("*")
      .eq("user_address", userAddress)
      .eq("season_id", seasonId)
      .order(orderBy, { ascending: order === "asc" })
      .range(offset, offset + limit - 1);

    if (error) throw error;
    return data;
  }

  /**
   * Get user's aggregated position for a season
   */
  async getUserPosition(userAddress, seasonId) {
    const { data, error } = await db.client
      .from("user_raffle_positions")
      .select("*")
      .eq("user_address", userAddress)
      .eq("season_id", seasonId)
      .single();

    if (error && error.code !== "PGRST116") throw error; // Ignore "not found"
    return data;
  }

  /**
   * Get user's positions across all seasons
   */
  async getAllUserPositions(userAddress) {
    const { data, error } = await db.client
      .from("user_raffle_positions")
      .select("*")
      .eq("user_address", userAddress)
      .order("season_id", { ascending: false });

    if (error) throw error;
    return data;
  }

  /**
   * Refresh materialized view
   */
  async refreshUserPositions(seasonId = null) {
    const { error } = await db.client.rpc("refresh_user_positions", {
      season_num: seasonId,
    });

    if (error) {
      console.error("Failed to refresh user positions:", error);
    }
  }
}

export const raffleTransactionService = new RaffleTransactionService();
```

### Phase 2: Event Listener Integration

#### Modify: `backend/src/listeners/positionUpdateListener.js`

Add transaction recording after line 115:

```javascript
// After logging the PositionUpdate event
logger.debug(
  `📊 PositionUpdate Event: Season ${seasonIdNum}, Player ${player}, ` +
    `Tickets: ${oldTicketsNum} → ${newTicketsNum}, Total: ${totalTicketsNum}`
);

// NEW: Record transaction in database
try {
  const ticketDelta = newTicketsNum - oldTicketsNum;
  const transactionType = ticketDelta > 0 ? "BUY" : "SELL";

  await raffleTransactionService.recordTransaction({
    seasonId: seasonIdNum,
    userAddress: player,
    transactionType,
    ticketAmount: Math.abs(ticketDelta),
    sofAmount: 0, // Will be populated from tx receipt in backfill
    txHash: log.transactionHash,
    blockNumber: Number(log.blockNumber),
    blockTimestamp: new Date().toISOString(), // Approximate, backfill will correct
    ticketsBefore: oldTicketsNum,
    ticketsAfter: newTicketsNum,
  });

  logger.debug(`   ✅ Transaction recorded: ${log.transactionHash}`);
} catch (txError) {
  logger.warn(`   ⚠️  Failed to record transaction: ${txError.message}`);
}
```

### Phase 3: API Endpoints

#### File: `backend/fastify/routes/raffleTransactionRoutes.js`

```javascript
import { raffleTransactionService } from "../../src/services/raffleTransactionService.js";

export default async function raffleTransactionRoutes(fastify) {
  // Get user's transaction history for a season
  fastify.get(
    "/transactions/:userAddress/:seasonId",
    async (request, reply) => {
      const { userAddress, seasonId } = request.params;
      const { limit, offset, orderBy, order } = request.query;

      try {
        const transactions = await raffleTransactionService.getUserTransactions(
          userAddress,
          parseInt(seasonId),
          { limit, offset, orderBy, order }
        );

        return { transactions };
      } catch (error) {
        fastify.log.error("Failed to fetch transactions:", error);
        return reply.code(500).send({ error: error.message });
      }
    }
  );

  // Get user's aggregated position for a season
  fastify.get("/positions/:userAddress/:seasonId", async (request, reply) => {
    const { userAddress, seasonId } = request.params;

    try {
      const position = await raffleTransactionService.getUserPosition(
        userAddress,
        parseInt(seasonId)
      );

      return { position };
    } catch (error) {
      fastify.log.error("Failed to fetch position:", error);
      return reply.code(500).send({ error: error.message });
    }
  });

  // Get user's positions across all seasons
  fastify.get("/positions/:userAddress", async (request, reply) => {
    const { userAddress } = request.params;

    try {
      const positions = await raffleTransactionService.getAllUserPositions(
        userAddress
      );

      return { positions };
    } catch (error) {
      fastify.log.error("Failed to fetch all positions:", error);
      return reply.code(500).send({ error: error.message });
    }
  });

  // Admin: Sync transactions for a season
  fastify.post("/admin/sync/:seasonId", async (request, reply) => {
    const { seasonId } = request.params;
    const { bondingCurveAddress } = request.body;

    try {
      const result = await raffleTransactionService.syncSeasonTransactions(
        parseInt(seasonId),
        bondingCurveAddress
      );

      return result;
    } catch (error) {
      fastify.log.error("Failed to sync transactions:", error);
      return reply.code(500).send({ error: error.message });
    }
  });
}
```

### Phase 4: Startup Sync

#### Modify: `backend/fastify/server.js`

Add historical sync function:

```javascript
// After syncHistoricalPositions function
async function syncHistoricalTransactions() {
  try {
    app.log.info("🔄 Starting historical transaction sync...");

    const results = await raffleTransactionService.syncAllActiveSeasons(
      process.env.BONDING_CURVE_ADDRESS_TESTNET // Or from config
    );

    const totalRecorded = results.reduce(
      (sum, r) => sum + (r.recorded || 0),
      0
    );
    app.log.info(
      `✅ Historical transaction sync complete: ${totalRecorded} new transactions`
    );
  } catch (error) {
    app.log.error(`❌ Historical transaction sync failed: ${error.message}`);
  }
}

// Call on startup (after syncHistoricalPositions)
syncHistoricalTransactions().catch((err) =>
  app.log.error("Transaction sync startup error:", err)
);
```

## Frontend Implementation

### Phase 5: Transaction History Component

#### File: `src/components/portfolio/TransactionHistory.jsx`

```javascript
import { useQuery } from "@tanstack/react-query";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { formatUnits } from "viem";

const TransactionHistory = ({ address, seasonId }) => {
  const { data: transactions, isLoading } = useQuery({
    queryKey: ["raffleTransactions", address, seasonId],
    queryFn: async () => {
      const response = await fetch(
        `${
          import.meta.env.VITE_BACKEND_URL
        }/api/raffle/transactions/${address}/${seasonId}`
      );
      if (!response.ok) throw new Error("Failed to fetch transactions");
      const data = await response.json();
      return data.transactions;
    },
    enabled: !!address && !!seasonId,
    staleTime: 30000,
  });

  if (isLoading) return <div>Loading transaction history...</div>;
  if (!transactions || transactions.length === 0) {
    return <div className="text-muted-foreground">No transactions yet</div>;
  }

  return (
    <div className="space-y-2">
      {transactions.map((tx) => (
        <Card key={tx.id} className="p-3">
          <div className="flex justify-between items-start">
            <div>
              <div className="font-medium">
                {tx.transaction_type === "BUY" ? "🎫 Bought" : "💸 Sold"}{" "}
                {tx.ticket_amount} tickets
              </div>
              <div className="text-sm text-muted-foreground">
                {new Date(tx.block_timestamp).toLocaleString()}
              </div>
            </div>
            <div className="text-right">
              <div className="font-medium">{tx.sof_amount.toFixed(2)} SOF</div>
              <div className="text-xs text-muted-foreground">
                @ {tx.price_per_ticket?.toFixed(4)} SOF/ticket
              </div>
            </div>
          </div>
          <div className="mt-2 text-xs text-muted-foreground">
            Position: {tx.tickets_before} → {tx.tickets_after} tickets
          </div>
        </Card>
      ))}
    </div>
  );
};

export default TransactionHistory;
```

### Phase 6: Update AccountPage.jsx

Add transaction history to the raffle ticket accordion:

```javascript
// Inside RaffleEntryRow component
<AccordionContent>
  <div className="space-y-4">
    {/* Existing position display */}
    <div>Current Position: {ticketBalance} tickets</div>

    {/* NEW: Transaction History */}
    <div>
      <h4 className="font-medium mb-2">Transaction History</h4>
      <TransactionHistory address={address} seasonId={row.seasonId} />
    </div>
  </div>
</AccordionContent>
```

## Migration & Deployment Plan

### Step 1: Database Migration

```sql
-- Run in Supabase SQL Editor

-- 1. Create partitioned table
-- (Copy schema from above)

-- 2. Create partitions for existing seasons
-- (Copy partition creation code)

-- 3. Create materialized view
-- (Copy view creation code)

-- 4. Add last_tx_sync_block to seasons table
ALTER TABLE seasons
ADD COLUMN IF NOT EXISTS last_tx_sync_block BIGINT DEFAULT 0;
```

### Step 2: Backend Deployment

1. ✅ Create `raffleTransactionService.js`
2. ✅ Add transaction recording to `positionUpdateListener.js`
3. ✅ Create `raffleTransactionRoutes.js`
4. ✅ Register routes in `server.js`
5. ✅ Add startup sync call
6. ✅ Deploy to Railway

### Step 3: Historical Backfill

```bash
# After deployment, trigger backfill via API
curl -X POST https://your-backend.railway.app/api/raffle/admin/sync/1 \
  -H "Content-Type: application/json" \
  -d '{"bondingCurveAddress": "0x..."}'
```

### Step 4: Frontend Deployment

1. ✅ Create `TransactionHistory.jsx` component
2. ✅ Update `AccountPage.jsx` to include history
3. ✅ Deploy to Vercel

## Performance Considerations

### Query Optimization

- **Partitioning** reduces query scan size (only relevant season)
- **Materialized view** provides O(1) lookups for aggregated data
- **Indexes** on user_address + season_id for fast filtering

### Scaling Strategy

- **Current**: Single partition per season
- **Future (>100 seasons)**: Archive old seasons to separate tablespace
- **Future (>1M tx/season)**: Sub-partition by month within season

### Refresh Strategy

- **Real-time**: Refresh materialized view after each transaction batch
- **Scheduled**: Fallback refresh every 5 minutes via cron
- **On-demand**: Manual refresh via admin API

## Testing Plan

### Unit Tests

```javascript
// tests/backend/raffleTransactionService.test.js
describe("RaffleTransactionService", () => {
  it("should record transaction idempotently", async () => {
    // Test duplicate tx_hash handling
  });

  it("should sync historical transactions", async () => {
    // Test backfill logic
  });

  it("should calculate correct aggregates", async () => {
    // Test materialized view data
  });
});
```

### Integration Tests

```javascript
// tests/e2e/transaction-history.test.js
describe("Transaction History E2E", () => {
  it("should display user transaction history", async () => {
    // Test full flow: event → DB → API → UI
  });
});
```

## Monitoring & Alerts

### Key Metrics

- Transaction recording success rate
- Backfill completion time
- Materialized view refresh duration
- API response times

### Alerts

- Failed transaction recordings (>5% error rate)
- Backfill lag (>1 hour behind chain)
- Slow queries (>1s response time)

## Rollback Plan

If issues arise:

1. **Disable transaction recording** in listener (comment out)
2. **Keep existing chain queries** as fallback
3. **Fix issues** in staging environment
4. **Re-enable** with confidence

## Success Criteria

✅ Users see complete transaction history in Portfolio
✅ Queries return in <500ms
✅ Historical backfill completes in <5 minutes
✅ No duplicate transactions recorded
✅ System handles 1000+ transactions/day

## Timeline Estimate

- **Phase 1-2 (Backend)**: 4-6 hours
- **Phase 3-4 (API + Sync)**: 2-3 hours
- **Phase 5-6 (Frontend)**: 2-3 hours
- **Testing & Deployment**: 2-3 hours
- **Total**: 10-15 hours

## Next Steps

1. Review and approve this plan
2. Create Supabase migration script
3. Implement backend service
4. Add event listener integration
5. Create API endpoints
6. Build frontend components
7. Test in staging
8. Deploy to production
9. Monitor and iterate
