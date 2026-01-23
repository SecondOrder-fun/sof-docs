# Sepolia Deployment Guide

**Date:** November 10, 2025  
**Network:** Base Sepolia Testnet  
**Deployment Method:** Frame.sh (Lattice Hardware Wallet)

---

## Prerequisites

### 1. Frame.sh Installation
- Download from https://frame.sh
- Install on your system (macOS, Windows, or Linux)
- Start Frame.sh application

### 2. Lattice Hardware Wallet
- Connect your Lattice hardware wallet to Frame.sh
- Ensure it's unlocked and ready

### 3. Base Sepolia ETH
- Get test ETH from faucet: https://www.alchemy.com/faucets/base-sepolia
- Need ETH for gas fees during deployment

### 4. Foundry
- Already installed in project
- Verify: `forge --version`

---

## Deployment Steps

### Step 1: Start Frame.sh

```bash
# Frame.sh runs on localhost:1248
# On macOS: Open Frame.app from Applications
# On Linux/Windows: Run Frame executable
```

### Step 2: Get Lattice Address

```bash
export FRAME_RPC="http://127.0.0.1:1248"
LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)
echo "Lattice Address: $LATTICE_ADDRESS"
```

### Step 3: Deploy Contracts

```bash
cd /Users/psd/Documents/PROJECTS/SOf/sof-alpha/contracts

export FRAME_RPC="http://127.0.0.1:1248"
LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)

forge script script/deploy/00_DeployToSepolia.s.sol \
  --rpc-url $FRAME_RPC \
  --sender $LATTICE_ADDRESS \
  --broadcast \
  -vvvv
```

### Step 4: Confirm Transactions

- Forge will prompt you to confirm transactions in Frame.sh
- Approve each transaction on your Lattice device
- Wait for confirmations

### Step 5: Update .env with Deployed Addresses

After deployment, update `.env` with the new Sepolia addresses:

```bash
# Add to .env:
VITE_RAFFLE_ADDRESS_TESTNET=0x...
VITE_SOF_ADDRESS_TESTNET=0x...
VITE_SEASON_FACTORY_ADDRESS_TESTNET=0x...
SOF_ADDRESS_TESTNET=0x...
RAFFLE_ADDRESS_TESTNET=0x...
SEASON_FACTORY_ADDRESS_TESTNET=0x...
```

### Step 6: Switch Frontend to Sepolia

Update `src/config/networks.js`:

```javascript
export const NETWORKS = {
  LOCAL: {
    chainId: 31337,
    name: 'Anvil',
    rpcUrl: 'http://127.0.0.1:8545',
  },
  TESTNET: {
    chainId: 84532,
    name: 'Base Sepolia',
    rpcUrl: 'https://sepolia.base.org',
  },
};

// Set default to TESTNET
export const DEFAULT_NETWORK = 'TESTNET';
```

### Step 7: Restart Backend and Frontend

```bash
# Terminal 1: Backend
npm run dev:backend

# Terminal 2: Frontend
npm run dev
```

---

## Deployment Output

The script will output:

```
============================================================
RAFFLE SYSTEM DEPLOYMENT COMPLETE
============================================================
SOF Token: 0x...
Raffle: 0x...
SeasonFactory: 0x...
SOFBondingCurve: 0x...
============================================================
```

---

## Verification

### 1. Check Deployment on Block Explorer

Visit: https://sepolia.basescan.org/

Search for your deployed contract addresses to verify.

### 2. Test Contract Interaction

```bash
# Check SOF balance
cast call 0x... "balanceOf(address)" 0x... --rpc-url https://sepolia.base.org

# Check Raffle state
cast call 0x... "owner()" --rpc-url https://sepolia.base.org
```

### 3. Frontend Verification

- Open frontend at http://localhost:5173
- Check that contract addresses are loaded
- Verify network is set to Base Sepolia

---

## Troubleshooting

### Frame.sh Not Connecting

```bash
# Verify Frame.sh is running on port 1248
curl http://127.0.0.1:1248

# If not working, restart Frame.sh
```

### Transaction Rejected

- Check Lattice device is unlocked
- Verify you have enough Base Sepolia ETH for gas
- Check transaction details in Frame.sh

### Contract Deployment Failed

- Check gas limit (set to 30M in Foundry config)
- Verify RPC endpoint is working
- Check contract code for compilation errors

---

## Environment Variables

### Current .env (Anvil/Local)

```
DEFAULT_NETWORK=LOCAL
RPC_URL_LOCAL=http://127.0.0.1:8545
```

### For Sepolia Testing

```
DEFAULT_NETWORK=TESTNET
RPC_URL_TESTNET=https://sepolia.base.org
```

---

## Next Steps After Deployment

1. ✅ Deploy contracts to Sepolia
2. ⏳ Update .env with Sepolia addresses
3. ⏳ Switch frontend to Sepolia network
4. ⏳ Test contract interactions
5. ⏳ Set up VRF subscription (if needed)
6. ⏳ Configure backend for Sepolia

---

## Resources

- Frame.sh: https://frame.sh
- Base Sepolia Faucet: https://www.alchemy.com/faucets/base-sepolia
- Base Sepolia Explorer: https://sepolia.basescan.org/
- Foundry Book: https://book.getfoundry.sh/

---

## Support

For issues:
1. Check Frame.sh logs
2. Verify RPC endpoint connectivity
3. Check Lattice device status
4. Review contract deployment output

