# Backend Repository .env.example

This is the `.env.example` file for the new `sof-backend` repository.

```bash
# Server Configuration
PORT=3000
NODE_ENV=development

# Database Configuration
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Redis Configuration
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# Blockchain Configuration - Local
RPC_URL=http://127.0.0.1:8545
LOCAL_CHAIN_ID=31337
PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
BACKEND_WALLET_PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
BACKEND_WALLET_ADDRESS=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

# Blockchain Configuration - Testnet (Base Sepolia)
RPC_URL_TESTNET=https://sepolia.base.org
TESTNET_CHAIN_ID=84532
PRIVATE_KEY_TESTNET=your-testnet-private-key

# Blockchain Configuration - Mainnet (Base)
RPC_URL_MAINNET=https://mainnet.base.org
MAINNET_CHAIN_ID=8453
PRIVATE_KEY_MAINNET=your-mainnet-private-key

# Contract Addresses - Local
RAFFLE_ADDRESS_LOCAL=0x...
SEASON_FACTORY_ADDRESS_LOCAL=0x...
SOF_ADDRESS_LOCAL=0x...
BONDING_CURVE_ADDRESS_LOCAL=0x...
PRIZE_DISTRIBUTOR_ADDRESS_LOCAL=0x...
SOF_FAUCET_ADDRESS_LOCAL=0x...
INFOFI_FACTORY_ADDRESS_LOCAL=0x...
INFOFI_ORACLE_ADDRESS_LOCAL=0x...

# Contract Addresses - Testnet
RAFFLE_ADDRESS_TESTNET=0x...
SEASON_FACTORY_ADDRESS_TESTNET=0x...
SOF_ADDRESS_TESTNET=0x...
BONDING_CURVE_ADDRESS_TESTNET=0x...
PRIZE_DISTRIBUTOR_ADDRESS_TESTNET=0x...
SOF_FAUCET_ADDRESS_TESTNET=0x...
INFOFI_FACTORY_ADDRESS_TESTNET=0x...
INFOFI_ORACLE_ADDRESS_TESTNET=0x...

# Contract Addresses - Mainnet
RAFFLE_ADDRESS=0x...
SEASON_FACTORY_ADDRESS=0x...
SOF_ADDRESS=0x...
BONDING_CURVE_ADDRESS=0x...
PRIZE_DISTRIBUTOR_ADDRESS=0x...
INFOFI_FACTORY_ADDRESS=0x...
INFOFI_ORACLE_ADDRESS=0x...

# VRF Configuration - Local (Mock)
VRF_COORDINATOR_LOCAL=0x000000000000000000000000000000000000cAFe
VRF_KEY_HASH_LOCAL=0x0000000000000000000000000000000000000000000000000000000000000000
VRF_SUBSCRIPTION_ID_LOCAL=0

# VRF Configuration - Testnet (Base Sepolia)
VRF_COORDINATOR_TESTNET=0x5C210eF41CD1a72de73bF76eC39637bB0d3d7BEE
VRF_KEY_HASH_TESTNET=0x9e1344a1247c8a1785d0a4681a27152bffdb43666ae5bf7d14d24a5efd44bf71
VRF_SUBSCRIPTION_ID_TESTNET=your-subscription-id

# VRF Configuration - Mainnet (Base)
VRF_COORDINATOR=0x...
VRF_KEY_HASH=0x...
VRF_SUBSCRIPTION_ID=your-subscription-id

# JWT Configuration
JWT_SECRET=your-jwt-secret-key-change-in-production
JWT_EXPIRES_IN=7d

# CORS Configuration
CORS_ORIGIN=http://localhost:5173,https://secondorder.fun

# Rate Limiting
RATE_LIMIT_MAX=100
RATE_LIMIT_TIMEWINDOW=60000

# Oracle Configuration
ORACLE_MAX_RETRIES=5
ORACLE_ALERT_CUTOFF=3

# Logging
LOG_LEVEL=info
```

## Key Differences from Frontend .env

### Removed Variables

- All `VITE_*` prefixed variables (frontend-only)
- Frontend-specific configuration

### Added Variables

- `PORT` - Backend server port
- `JWT_SECRET` - Authentication secret
- `CORS_ORIGIN` - Allowed frontend origins
- `RATE_LIMIT_*` - API rate limiting
- `ORACLE_*` - Oracle service configuration
- `LOG_LEVEL` - Backend logging level

### Security Notes

1. **Never commit actual private keys**
2. **Change JWT_SECRET in production**
3. **Use environment-specific keys**
4. **Rotate keys regularly**
5. **Use secrets management in production** (Railway Secrets, AWS Secrets Manager, etc.)
