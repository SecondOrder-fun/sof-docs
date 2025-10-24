# Backend Wallet Setup Guide

## Overview

The backend wallet is a dedicated Ethereum address that the SecondOrder.fun backend uses to create InfoFi markets on behalf of users. This architecture reduces user gas costs by 70-80% by moving market creation from user transactions to backend operations.

## Why Backend Wallet?

**Before (On-Chain Market Creation):**
- User buys tickets: ~100k gas
- Market creation triggered automatically: ~400k gas
- **Total user cost: ~500k gas**

**After (Backend-Driven Market Creation):**
- User buys tickets: ~100k gas
- Backend creates market separately: ~300k gas (platform absorbs)
- **Total user cost: ~100k gas (80% savings)**

## Security Considerations

### Private Key Management

**CRITICAL:** The backend wallet private key must be kept secure:

1. **Never commit to Git**
   - Add to `.gitignore`
   - Use environment variables only

2. **Production Environment**
   - Use AWS Secrets Manager, HashiCorp Vault, or similar
   - Rotate keys periodically
   - Implement key access logging

3. **Local Development**
   - Use Anvil's default accounts (account[0])
   - Never use production keys locally

### Access Control

The backend wallet needs:
- **BACKEND_ROLE** on `InfoFiMarketFactory` contract
- Sufficient ETH for gas costs
- No other special permissions (principle of least privilege)

## Setup Instructions

### Local Development (Anvil)

1. **Start Anvil:**
   ```bash
   anvil --gas-limit 30000000
   ```

2. **Use Default Account:**
   ```bash
   # Anvil account[0] (default)
   BACKEND_WALLET_ADDRESS=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
   BACKEND_WALLET_PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
   ```

3. **Add to `.env`:**
   ```bash
   # Backend Wallet Configuration
   BACKEND_WALLET_ADDRESS=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
   BACKEND_WALLET_PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
   ```

4. **Grant Role During Deployment:**
   The `Deploy.s.sol` script automatically grants `BACKEND_ROLE` to the backend wallet address.

### Testnet (Sepolia)

1. **Create Dedicated Wallet:**
   ```bash
   # Generate new wallet
   cast wallet new
   
   # Save output:
   # Address: 0x...
   # Private key: 0x...
   ```

2. **Fund Wallet:**
   ```bash
   # Get testnet ETH from faucet
   # https://sepoliafaucet.com/
   
   # Verify balance
   cast balance 0xYourBackendWalletAddress --rpc-url https://sepolia.infura.io/v3/YOUR_KEY
   ```

3. **Add to `.env.sepolia`:**
   ```bash
   BACKEND_WALLET_ADDRESS=0xYourBackendWalletAddress
   BACKEND_WALLET_PRIVATE_KEY=0xYourPrivateKey
   ```

4. **Grant Role:**
   ```bash
   # After deploying contracts
   cast send $INFOFI_FACTORY_ADDRESS \
     "grantRole(bytes32,address)" \
     $(cast keccak "BACKEND_ROLE") \
     $BACKEND_WALLET_ADDRESS \
     --rpc-url https://sepolia.infura.io/v3/YOUR_KEY \
     --private-key $ADMIN_PRIVATE_KEY
   ```

### Production (Mainnet)

1. **Create Production Wallet:**
   ```bash
   # Use hardware wallet or secure key generation
   # NEVER generate production keys on development machine
   
   # Example with Ledger:
   cast wallet address --ledger
   ```

2. **Secure Key Storage:**
   ```bash
   # AWS Secrets Manager
   aws secretsmanager create-secret \
     --name prod/backend-wallet-private-key \
     --secret-string "0xYourPrivateKey"
   
   # Retrieve in production
   export BACKEND_WALLET_PRIVATE_KEY=$(aws secretsmanager get-secret-value \
     --secret-id prod/backend-wallet-private-key \
     --query SecretString \
     --output text)
   ```

3. **Fund Wallet:**
   ```bash
   # Calculate required funding:
   # - Average 300k gas per market creation
   # - Expected 100 markets per day
   # - Gas price average: 30 gwei
   # Daily cost: 100 * 300k * 30 gwei = 0.9 ETH
   # Fund for 30 days: ~30 ETH
   
   # Send ETH from treasury
   cast send $BACKEND_WALLET_ADDRESS \
     --value 30ether \
     --rpc-url $MAINNET_RPC \
     --private-key $TREASURY_PRIVATE_KEY
   ```

4. **Grant Role:**
   ```bash
   # Use multi-sig for production role grants
   # Example with Gnosis Safe:
   # 1. Create transaction in Safe UI
   # 2. Call grantRole(BACKEND_ROLE, backendWalletAddress)
   # 3. Get required signatures
   # 4. Execute transaction
   ```

## Monitoring & Maintenance

### Balance Monitoring

Set up alerts for low balance:

```javascript
// Example monitoring script
const ALERT_THRESHOLD = ethers.parseEther("5"); // Alert at 5 ETH
const CRITICAL_THRESHOLD = ethers.parseEther("2"); // Critical at 2 ETH

async function checkBalance() {
  const balance = await provider.getBalance(BACKEND_WALLET_ADDRESS);
  
  if (balance < CRITICAL_THRESHOLD) {
    sendCriticalAlert("Backend wallet critically low on ETH");
  } else if (balance < ALERT_THRESHOLD) {
    sendWarningAlert("Backend wallet needs refunding soon");
  }
}

// Run every hour
setInterval(checkBalance, 60 * 60 * 1000);
```

### Gas Cost Tracking

Monitor actual gas costs:

```javascript
// Log gas costs in backend
async function createMarket(seasonId, playerAddress) {
  const tx = await infoFiFactory.onPositionUpdate(/* ... */);
  const receipt = await tx.wait();
  
  const gasUsed = receipt.gasUsed;
  const gasPrice = receipt.gasPrice;
  const costEth = ethers.formatEther(gasUsed * gasPrice);
  
  logger.info({
    seasonId,
    playerAddress,
    gasUsed: gasUsed.toString(),
    gasPriceGwei: ethers.formatUnits(gasPrice, "gwei"),
    costEth,
  }, "Market created");
  
  // Store in database for analytics
  await db.logMarketCreationCost({
    seasonId,
    playerAddress,
    gasUsed,
    gasPrice,
    costEth,
    timestamp: new Date(),
  });
}
```

### Automatic Refunding

Set up automatic refunding from treasury:

```javascript
// Example auto-refund script
async function autoRefund() {
  const balance = await provider.getBalance(BACKEND_WALLET_ADDRESS);
  const targetBalance = ethers.parseEther("20"); // Keep 20 ETH
  
  if (balance < ethers.parseEther("10")) {
    const refundAmount = targetBalance - balance;
    
    // Send from treasury
    const tx = await treasuryWallet.sendTransaction({
      to: BACKEND_WALLET_ADDRESS,
      value: refundAmount,
    });
    
    await tx.wait();
    
    logger.info({
      refundAmount: ethers.formatEther(refundAmount),
      newBalance: ethers.formatEther(targetBalance),
    }, "Backend wallet refunded");
  }
}

// Run daily
setInterval(autoRefund, 24 * 60 * 60 * 1000);
```

## Troubleshooting

### "Insufficient funds for gas"

**Cause:** Backend wallet out of ETH

**Solution:**
```bash
# Check balance
cast balance $BACKEND_WALLET_ADDRESS --rpc-url $RPC_URL

# Fund wallet
cast send $BACKEND_WALLET_ADDRESS \
  --value 10ether \
  --rpc-url $RPC_URL \
  --private-key $TREASURY_PRIVATE_KEY
```

### "AccessControlUnauthorizedAccount"

**Cause:** Backend wallet doesn't have BACKEND_ROLE

**Solution:**
```bash
# Check if role granted
cast call $INFOFI_FACTORY_ADDRESS \
  "hasRole(bytes32,address)" \
  $(cast keccak "BACKEND_ROLE") \
  $BACKEND_WALLET_ADDRESS \
  --rpc-url $RPC_URL

# Grant role if needed
cast send $INFOFI_FACTORY_ADDRESS \
  "grantRole(bytes32,address)" \
  $(cast keccak "BACKEND_ROLE") \
  $BACKEND_WALLET_ADDRESS \
  --rpc-url $RPC_URL \
  --private-key $ADMIN_PRIVATE_KEY
```

### Gas Price Spikes

**Cause:** Network congestion causing high gas prices

**Solution:**
```javascript
// Implement gas price limits in backend
const MAX_GAS_PRICE = ethers.parseUnits("100", "gwei");

async function createMarketWithGasLimit(seasonId, playerAddress) {
  const currentGasPrice = await provider.getFeeData();
  
  if (currentGasPrice.gasPrice > MAX_GAS_PRICE) {
    logger.warn({
      currentGasPrice: ethers.formatUnits(currentGasPrice.gasPrice, "gwei"),
      maxGasPrice: ethers.formatUnits(MAX_GAS_PRICE, "gwei"),
    }, "Gas price too high, delaying market creation");
    
    // Retry later
    return { success: false, reason: "gas_too_high" };
  }
  
  // Proceed with creation
  return await createMarket(seasonId, playerAddress);
}
```

## Security Checklist

- [ ] Private key stored in secure secret manager (production)
- [ ] Private key never committed to Git
- [ ] Backend wallet has only BACKEND_ROLE (no admin roles)
- [ ] Balance monitoring alerts configured
- [ ] Automatic refunding system in place
- [ ] Gas cost tracking implemented
- [ ] Key rotation schedule defined
- [ ] Access logs enabled for key retrieval
- [ ] Backup key recovery process documented
- [ ] Multi-sig controls for role grants (production)

## Cost Estimates

### Per Market Creation

- **Gas used:** ~300,000 gas
- **Gas price (average):** 30 gwei
- **Cost per market:** ~0.009 ETH (~$18 at $2000/ETH)

### Daily Costs (100 markets/day)

- **Total gas:** 30,000,000 gas
- **Cost:** ~0.9 ETH/day (~$1,800/day)
- **Monthly:** ~27 ETH/month (~$54,000/month)

### Optimization Strategies

1. **Batch Creation:** Create multiple markets in single transaction
2. **Gas Price Timing:** Create markets during low-congestion periods
3. **Layer 2:** Consider L2 deployment for lower costs
4. **Threshold Adjustment:** Increase 1% threshold to reduce market count

## Admin Panel Integration

The backend wallet status is visible in the Admin Panel:

1. Navigate to **Admin Panel → Backend Wallet** tab
2. View current balance (ETH and SOF)
3. See market creation statistics
4. Monitor recent transactions
5. Get alerts for low balance

## References

- [OpenZeppelin AccessControl](https://docs.openzeppelin.com/contracts/4.x/access-control)
- [Ethers.js Wallet Management](https://docs.ethers.org/v6/api/wallet/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [Gnosis Safe Multi-Sig](https://safe.global/)
