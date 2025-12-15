# Frontend Repository .env.example (Updated)

This is the updated `.env.example` file for the frontend repository after backend split.

```bash
# Backend API Configuration
VITE_BACKEND_URL=http://localhost:3000
VITE_BACKEND_URL_PRODUCTION=https://api.secondorder.fun

# Blockchain Configuration - Local
VITE_RPC_URL=http://127.0.0.1:8545
VITE_CHAIN_ID=31337

# Blockchain Configuration - Testnet (Base Sepolia)
VITE_RPC_URL_TESTNET=https://sepolia.base.org
VITE_CHAIN_ID_TESTNET=84532

# Blockchain Configuration - Mainnet (Base)
VITE_RPC_URL_MAINNET=https://mainnet.base.org
VITE_CHAIN_ID_MAINNET=8453

# Contract Addresses - Local
VITE_RAFFLE_ADDRESS_LOCAL=0x...
VITE_SEASON_FACTORY_ADDRESS_LOCAL=0x...
VITE_SOF_ADDRESS_LOCAL=0x...
VITE_BONDING_CURVE_ADDRESS_LOCAL=0x...
VITE_PRIZE_DISTRIBUTOR_ADDRESS_LOCAL=0x...
VITE_SOF_FAUCET_ADDRESS_LOCAL=0x...
VITE_INFOFI_FACTORY_ADDRESS_LOCAL=0x...
VITE_INFOFI_ORACLE_ADDRESS_LOCAL=0x...

# Contract Addresses - Testnet
VITE_RAFFLE_ADDRESS_TESTNET=0x...
VITE_SEASON_FACTORY_ADDRESS_TESTNET=0x...
VITE_SOF_ADDRESS_TESTNET=0x...
VITE_BONDING_CURVE_ADDRESS_TESTNET=0x...
VITE_PRIZE_DISTRIBUTOR_ADDRESS_TESTNET=0x...
VITE_SOF_FAUCET_ADDRESS_TESTNET=0x...
VITE_INFOFI_FACTORY_ADDRESS_TESTNET=0x...
VITE_INFOFI_ORACLE_ADDRESS_TESTNET=0x...

# Contract Addresses - Mainnet
VITE_RAFFLE_ADDRESS=0x...
VITE_SEASON_FACTORY_ADDRESS=0x...
VITE_SOF_ADDRESS=0x...
VITE_BONDING_CURVE_ADDRESS=0x...
VITE_PRIZE_DISTRIBUTOR_ADDRESS=0x...
VITE_INFOFI_FACTORY_ADDRESS=0x...
VITE_INFOFI_ORACLE_ADDRESS=0x...

# Farcaster Configuration
VITE_FARCASTER_APP_ID=your-farcaster-app-id

# Feature Flags
VITE_ENABLE_TESTNET=true
VITE_ENABLE_MAINNET=false
VITE_ENABLE_FAUCET=true
```

## Key Changes from Monorepo

### Added Variables

- `VITE_BACKEND_URL` - Development backend URL
- `VITE_BACKEND_URL_PRODUCTION` - Production backend URL

### Removed Variables

- All non-VITE prefixed variables (backend-only)
- Database URLs (Supabase)
- Redis configuration
- Private keys (security - backend only)
- JWT secrets
- CORS configuration
- Rate limiting settings

### Important Notes

1. **All frontend env vars must have `VITE_` prefix** to be exposed to the browser
2. **Never put private keys in frontend .env** - they will be exposed in the browser
3. **Backend URL changes based on environment**:
   - Development: `http://localhost:3000`
   - Production: Your deployed backend URL

## Usage in Code

```javascript
// API client configuration
const API_URL = import.meta.env.VITE_BACKEND_URL || "http://localhost:3000";

// Example API call
const response = await fetch(`${API_URL}/api/seasons`, {
  method: "GET",
  headers: {
    "Content-Type": "application/json",
  },
});
```

## Environment-Specific Configuration

Create separate `.env` files for different environments:

- `.env.local` - Local development (gitignored)
- `.env.development` - Development environment
- `.env.staging` - Staging environment
- `.env.production` - Production environment

Vite will automatically load the correct file based on the mode.
