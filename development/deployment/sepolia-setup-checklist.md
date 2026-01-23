# Sepolia Deployment Setup Checklist

**Status:** Ready for deployment  
**Network:** Base Sepolia (Chain ID: 84532)  
**Deployment Method:** Frame.sh + Lattice Hardware Wallet

---

## Pre-Deployment Checklist

### System Requirements
- [x] Frame.sh installed and running
- [x] Lattice hardware wallet connected to Frame.sh
- [x] Base Sepolia ETH in Lattice wallet (for gas fees)
- [x] Foundry installed (`forge --version`)
- [x] Node.js and npm installed

### Network Configuration
- [x] Base Sepolia RPC: https://sepolia.base.org
- [x] Chain ID: 84532
- [ ] Block Explorer: https://sepolia.basescan.org/

### Get Test ETH
- [ ] Visit: https://www.alchemy.com/faucets/base-sepolia
- [x] Request Base Sepolia ETH to Lattice address
- [ ] Wait for confirmation (~1-2 minutes)

---

## Deployment Steps

### Step 1: Start Frame.sh
```bash
# Open Frame.app (macOS) or run Frame executable
# Verify it's running on localhost:1248
curl http://127.0.0.1:1248
```
- [x] Frame.sh running
- [x] Lattice device connected
- [x] Lattice device unlocked

### Step 2: Get Lattice Address
```bash
export FRAME_RPC="http://127.0.0.1:1248"
cast rpc eth_accounts --rpc-url http://127.0.0.1:1248
LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)
echo "Lattice Address: $LATTICE_ADDRESS"
```
- [ ] Lattice address obtained
- [ ] Address has Base Sepolia ETH

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
- [ ] Deployment script started
- [ ] Confirm transactions in Frame.sh
- [ ] All contracts deployed successfully
- [ ] Deployment addresses printed

### Step 4: Record Deployed Addresses
From deployment output, save:
- [ ] SOF Token address
- [ ] Raffle address
- [ ] SeasonFactory address
- [ ] SOFBondingCurve address

---

## Post-Deployment Configuration

### Step 5: Update .env File
Add to `./.env`:
```bash
# Sepolia Addresses
VITE_RAFFLE_ADDRESS_TESTNET=0x...
VITE_SOF_ADDRESS_TESTNET=0x...
VITE_SEASON_FACTORY_ADDRESS_TESTNET=0x...
VITE_BONDING_CURVE_ADDRESS_TESTNET=0x...

SOF_ADDRESS_TESTNET=0x...
RAFFLE_ADDRESS_TESTNET=0x...
SEASON_FACTORY_ADDRESS_TESTNET=0x...
BONDING_CURVE_ADDRESS_TESTNET=0x...

# Network Configuration
DEFAULT_NETWORK=TESTNET
RPC_URL_TESTNET=https://sepolia.base.org
TESTNET_CHAIN_ID=84532
```
- [ ] .env updated with Sepolia addresses
- [ ] DEFAULT_NETWORK set to TESTNET

### Step 6: Configure Frontend
Update `src/config/networks.js`:
```javascript
export const DEFAULT_NETWORK = 'TESTNET';
```
- [ ] Frontend network set to TESTNET
- [ ] Contract addresses configured

### Step 7: Restart Services
```bash
# Terminal 1: Backend
npm run dev:backend

# Terminal 2: Frontend
npm run dev
```
- [ ] Backend restarted
- [ ] Frontend restarted
- [ ] Both services running on Sepolia

---

## Verification

### Verify Deployment
- [ ] Visit https://sepolia.basescan.org/
- [ ] Search for deployed contract addresses
- [ ] Verify contracts are deployed and verified

### Test Frontend
- [ ] Open http://localhost:5173
- [ ] Check network is Base Sepolia
- [ ] Verify contract addresses are loaded
- [ ] Connect wallet to Base Sepolia

### Test Contract Interaction
```bash
# Check SOF balance
cast call 0x... "balanceOf(address)" 0x... \
  --rpc-url https://sepolia.base.org

# Check Raffle owner
cast call 0x... "owner()" \
  --rpc-url https://sepolia.base.org
```
- [ ] Contract calls working
- [ ] Data returned correctly

---

## Troubleshooting

### Frame.sh Connection Issues
```bash
# Verify Frame.sh is running
curl http://127.0.0.1:1248

# Check Frame.sh logs
# Restart Frame.sh if needed
```

### Deployment Failures
- [ ] Check gas limit (30M set in foundry.toml)
- [ ] Verify Lattice has enough ETH
- [ ] Check RPC endpoint connectivity
- [ ] Review contract compilation errors

### Frontend Issues
- [ ] Clear browser cache
- [ ] Hard refresh (Cmd+Shift+R)
- [ ] Check browser console for errors
- [ ] Verify .env variables loaded

---

## Environment Comparison

### Local (Anvil)
```
DEFAULT_NETWORK=LOCAL
RPC_URL_LOCAL=http://127.0.0.1:8545
Chain ID: 31337
```

### Testnet (Sepolia)
```
DEFAULT_NETWORK=TESTNET
RPC_URL_TESTNET=https://sepolia.base.org
Chain ID: 84532
```

---

## Quick Reference

| Item | Value |
|------|-------|
| Network | Base Sepolia |
| Chain ID | 84532 |
| RPC URL | https://sepolia.base.org |
| Explorer | https://sepolia.basescan.org/ |
| Faucet | https://www.alchemy.com/faucets/base-sepolia |
| Deployment Method | Frame.sh + Lattice |
| Frame.sh Port | 1248 |

---

## Next Steps After Deployment

1. ✅ Deploy contracts to Sepolia
2. ✅ Update .env with Sepolia addresses
3. ✅ Configure frontend for Sepolia
4. ⏳ Test contract interactions
5. ⏳ Set up VRF subscription (if needed)
6. ⏳ Configure backend for Sepolia
7. ⏳ Run end-to-end tests on Sepolia

---

## Support Resources

- **Frame.sh:** https://frame.sh
- **Base Sepolia Faucet:** https://www.alchemy.com/faucets/base-sepolia
- **Block Explorer:** https://sepolia.basescan.org/
- **Foundry Docs:** https://book.getfoundry.sh/
- **Deployment Guide:** See SEPOLIA_DEPLOYMENT_GUIDE.md

---

**Created:** November 10, 2025  
**Status:** Ready for execution

