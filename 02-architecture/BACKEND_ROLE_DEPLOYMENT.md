# Backend Role Deployment Integration

## Overview

The `BACKEND_ROLE` on `InfoFiMarketFactory` is now **automatically granted during deployment**. No manual steps required.

## How It Works

### 1. Deployment Script (Deploy.s.sol)

The deployment script automatically:

1. **Determines Backend Wallet**:
   - Checks for `BACKEND_WALLET_ADDRESS` environment variable
   - Falls back to deployer address for local development
   - Uses dedicated wallet for production

2. **Grants BACKEND_ROLE**:
   - Role is granted in `InfoFiMarketFactory` constructor
   - Happens automatically when factory is deployed
   - No separate transaction needed

3. **Verifies Grant**:
   - Checks `hasRole(BACKEND_ROLE, backendWallet)` after deployment
   - Fails deployment if role not granted
   - Logs verification status to console

### 2. Backend Wallet Configuration

#### Local Development (Default)
```bash
# Uses account[0] (deployer) as backend wallet
# No additional configuration needed
forge script script/Deploy.s.sol --rpc-url $RPC_URL --private-key $PRIVATE_KEY --broadcast
```

#### Production (Dedicated Wallet)
```bash
# Set dedicated backend wallet address in .env
BACKEND_WALLET_ADDRESS=0x1234...

# Deploy with custom backend wallet
forge script script/Deploy.s.sol --rpc-url $RPC_URL --private-key $PRIVATE_KEY --broadcast
```

### 3. Backend Service Configuration

The backend service uses the same wallet:

```javascript
// backend/src/lib/viemClient.js
const privateKey = process.env.BACKEND_WALLET_PRIVATE_KEY || process.env.PRIVATE_KEY;
```

**Environment Variables**:
- `BACKEND_WALLET_PRIVATE_KEY` - Dedicated backend service wallet (production)
- `PRIVATE_KEY` - Fallback to deployer wallet (local dev)

## Deployment Logs

Successful deployment shows:

```
Deploying InfoFiMarketFactory (V2 with FPMM)...
Using deployer as backend wallet (local dev mode)
InfoFiMarketFactory deployed at: 0x68B1D87F95878fE05B998F19b66F4baba5De1aed
Backend wallet granted BACKEND_ROLE: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
BACKEND_ROLE verification: SUCCESS
```

## Verification

### Manual Verification (Optional)

If you want to manually verify the role grant:

```bash
# Using cast
cast call $INFOFI_FACTORY_ADDRESS "hasRole(bytes32,address)" \
  $(cast keccak "BACKEND_ROLE") \
  $BACKEND_WALLET_ADDRESS \
  --rpc-url $RPC_URL

# Using the verification script
node scripts/grant-backend-role.js
```

### Automated Verification

The deployment script includes automatic verification:

```solidity
// Verify BACKEND_ROLE was granted
bytes32 backendRole = infoFiFactory.BACKEND_ROLE();
bool hasRole = infoFiFactory.hasRole(backendRole, backendWallet);
console2.log("BACKEND_ROLE verification:", hasRole ? "SUCCESS" : "FAILED");
require(hasRole, "BACKEND_ROLE not granted to backend wallet");
```

If verification fails, deployment reverts with clear error message.

## Troubleshooting

### Issue: Backend can't create markets

**Symptoms**:
- Backend logs show `OnlyBackend()` error
- Transactions revert when calling `onPositionUpdate()`

**Solution**:
1. Check backend wallet address matches deployment:
   ```bash
   # Get backend wallet from backend service
   node -e "const {privateKeyToAccount} = require('viem/accounts'); console.log(privateKeyToAccount(process.env.BACKEND_WALLET_PRIVATE_KEY || process.env.PRIVATE_KEY).address)"
   ```

2. Verify role grant:
   ```bash
   node scripts/grant-backend-role.js
   ```

3. If role not granted, re-deploy or manually grant:
   ```bash
   # Manual grant (requires admin role)
   cast send $INFOFI_FACTORY_ADDRESS \
     "grantRole(bytes32,address)" \
     $(cast keccak "BACKEND_ROLE") \
     $BACKEND_WALLET_ADDRESS \
     --rpc-url $RPC_URL --private-key $ADMIN_PRIVATE_KEY
   ```

### Issue: Deployment fails at verification

**Symptoms**:
- Deployment reverts with "BACKEND_ROLE not granted to backend wallet"

**Cause**:
- Constructor failed to grant role
- Wrong backend wallet address

**Solution**:
1. Check `InfoFiMarketFactory` constructor grants role correctly
2. Verify `BACKEND_WALLET_ADDRESS` env var is correct
3. Check deployment logs for backend wallet address

## Production Deployment Checklist

- [ ] Set `BACKEND_WALLET_ADDRESS` in deployment environment
- [ ] Set `BACKEND_WALLET_PRIVATE_KEY` in backend service environment
- [ ] Run deployment script
- [ ] Verify "BACKEND_ROLE verification: SUCCESS" in logs
- [ ] Test backend can create markets (buy tickets to cross 1% threshold)
- [ ] Monitor backend logs for successful market creation

## Related Files

- `contracts/script/Deploy.s.sol` - Deployment script with role grant
- `contracts/src/infofi/InfoFiMarketFactory.sol` - Factory contract with BACKEND_ROLE
- `backend/src/lib/viemClient.js` - Backend wallet configuration
- `backend/src/services/infoFiMarketCreator.js` - Market creation service
- `scripts/grant-backend-role.js` - Manual role grant script (backup)
- `.env.example` - Environment variable documentation

## Summary

✅ **BACKEND_ROLE is now part of automated deployment**
✅ **No manual steps required for local development**
✅ **Production requires setting BACKEND_WALLET_ADDRESS env var**
✅ **Deployment fails fast if role not granted**
✅ **Verification script available for troubleshooting**
