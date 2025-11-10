# Frame.sh Deployment Quick Start

**TL;DR** - Deploy to Sepolia in 5 commands

## Prerequisites

- Frame.sh running on localhost:1248
- Lattice hardware wallet connected to Frame
- Base Sepolia ETH for gas fees

## Quick Deploy

```bash
# 1. Get your Lattice address
export FRAME_RPC="http://127.0.0.1:1248"
LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)
echo "Deploying from: $LATTICE_ADDRESS"

# 2. Go to contracts directory
cd /Users/psd/Documents/PROJECTS/SOf/sof-alpha/contracts

# 3. Deploy contracts (Lattice will prompt for approval)
forge script script/deploy/00_DeployToSepolia.s.sol \
  --rpc-url $FRAME_RPC \
  --sender $LATTICE_ADDRESS \
  --broadcast \
  -vvvv

# 4. Copy deployed addresses from output
# (You'll see them printed at the end)

# 5. Update .env with addresses and start backend/frontend
```

## Verify Deployment

```bash
# Check contract on Sepolia Basescan
https://sepolia.basescan.org/address/0x...
```

## Environment Setup

```bash
# .env
DEFAULT_NETWORK=TESTNET
RPC_URL_TESTNET=https://sepolia.base.org
RAFFLE_ADDRESS_TESTNET=0x...  # From deployment
SOF_ADDRESS_TESTNET=0x...
SEASON_FACTORY_ADDRESS_TESTNET=0x...
INFOFI_FACTORY_ADDRESS_TESTNET=0x...
INFOFI_ORACLE_ADDRESS_TESTNET=0x...
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Frame not responding | Restart Frame.sh, check Lattice connected |
| Address lookup fails | Ensure Frame is running on localhost:1248 |
| Transaction rejected | Check ETH balance, approve on Lattice device |
| Deployment fails | Run with `-vvvv` flag to see detailed logs |

## Full Guide

See `PHASE5-SEPOLIA-DEPLOYMENT.md` for complete documentation.
