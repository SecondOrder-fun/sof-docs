# Reset Sequences Migration

## Purpose

When clearing InfoFi markets during local development (e.g., when restarting Anvil), the database auto-increment sequences continue from where they left off. This causes market IDs to be non-sequential (e.g., 13, 14 instead of 1, 2).

This migration adds SQL functions to reset the sequences.

## Apply Migration

Run this SQL in your Supabase SQL Editor:

```sql
-- Copy and paste the contents of:
-- backend/src/db/migrations/reset_sequences.sql
```

Or use the Supabase CLI:

```bash
# If you have supabase CLI installed
supabase db push backend/src/db/migrations/reset_sequences.sql
```

## Manual Reset (Alternative)

If you don't want to create the function, you can manually reset sequences:

```sql
-- Reset infofi_markets sequence
SELECT setval(pg_get_serial_sequence('infofi_markets', 'id'), 1, false);

-- This will make the next inserted row have id = 1
```

## Usage

Once the migration is applied, the `clearAllInfoFiMarkets()` function will automatically reset the sequence when clearing markets.

## Verification

After restarting the backend:

```bash
curl http://127.0.0.1:3000/api/infofi/markets/1
# Should return the first market with id: 1
```
