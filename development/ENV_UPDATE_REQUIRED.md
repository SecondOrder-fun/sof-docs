# Environment Update Required: ConditionalTokenSOF

## Issue

The `.env` file is missing the `CONDITIONAL_TOKENS` address because:

1. ✅ The `update-env-addresses.js` script was updated to look for `ConditionalTokenSOF`
2. ❌ The current deployment broadcast file still references `ConditionalTokensMock` (old name)
3. ❌ Therefore, the address wasn't extracted during the last `update-env-addresses.js` run

## Solution

**Redeploy the contracts** to generate a new broadcast file with `ConditionalTokenSOF`:

```bash
# Start Anvil (if not already running)
anvil --gas-limit 30000000

# Deploy contracts (in new terminal)
cd contracts
forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --private-key $PRIVATE_KEY --broadcast

# Update .env with new addresses
cd ..
node scripts/update-env-addresses.js
```

After redeployment, the script will correctly extract:
- `CONDITIONAL_TOKENS_ADDRESS_LOCAL` (for backend)
- `VITE_CONDITIONAL_TOKENS_ADDRESS_LOCAL` (for frontend)

## What Was Fixed

### ✅ update-env-addresses.js
**Line 34 updated:**
```javascript
// Before
ConditionalTokens: "CONDITIONAL_TOKENS",

// After
ConditionalTokenSOF: "CONDITIONAL_TOKENS",
```

### ✅ .env.example Already Has Placeholders
```bash
# Frontend
VITE_CONDITIONAL_TOKENS_ADDRESS_LOCAL=
VITE_CONDITIONAL_TOKENS_ADDRESS_TESTNET=

# Backend
CONDITIONAL_TOKENS_ADDRESS_LOCAL=
CONDITIONAL_TOKENS_ADDRESS_TESTNET=
```

## Verification After Redeployment

Check that the address was added:
```bash
grep CONDITIONAL_TOKENS .env
```

Expected output:
```
CONDITIONAL_TOKENS_ADDRESS_LOCAL=0x...
VITE_CONDITIONAL_TOKENS_ADDRESS_LOCAL=0x...
```

## Current Status

- ✅ Script updated to look for `ConditionalTokenSOF`
- ✅ `.env.example` has placeholders
- ⏳ **Needs redeployment** to populate `.env`

## Next Step

**Redeploy contracts to Anvil** to generate the new broadcast file with ConditionalTokenSOF address.
