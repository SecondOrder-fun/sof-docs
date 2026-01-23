# Frame.sh Signing Process Troubleshooting

**Date:** Nov 10, 2025  
**Issue:** Frame.sh deployment trying to use Mainnet instead of Sepolia  
**Status:** ✅ RESOLVED

---

## The Problem

When running the deployment script, Frame.sh was attempting to sign transactions for **Mainnet** instead of **Base Sepolia**, causing:

- Wrong chain ID in transactions
- Potential loss of funds if approved
- Deployment to wrong network

---

## Root Cause

The deployment command was using `--rpc-url baseSepolia`, which points to the **public Sepolia RPC endpoint** (`https://sepolia.base.org`).

However, Frame.sh runs a **local RPC proxy** on `http://127.0.0.1:1248` that:
1. Intercepts all transactions
2. Routes them to the Lattice device for signing
3. Submits signed transactions to the actual network

**Using the public RPC endpoint bypasses Frame.sh's signing mechanism entirely.**

---

## The Solution

### ✅ CORRECT: Use Frame.sh Local RPC

```bash
cd /Users/psd/Documents/PROJECTS/SOf/sof-alpha/contracts

# Set environment variables
export FRAME_RPC="http://127.0.0.1:1248"
export LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)

# Deploy using Frame.sh local RPC
forge script script/deploy/00_DeployToSepolia.s.sol \
  --rpc-url $FRAME_RPC \
  --sender $LATTICE_ADDRESS \
  --broadcast \
  -vvvv
```

### ❌ WRONG: Using Public Sepolia RPC

```bash
# This BYPASSES Frame.sh signing!
forge script script/deploy/00_DeployToSepolia.s.sol \
  --rpc-url baseSepolia \
  --sender 0xeb23e28171099c5edddb1f96cfe38086af1b4852 \
  --broadcast \
  --unlocked \
  -vvvv
```

---

## Step-by-Step: Correct Frame.sh Deployment

### 1. Start Frame.sh

```bash
# In a separate terminal, start Frame.sh
# (Assuming Frame.sh is installed and configured)
frame start
```

**Expected output:**

```text
Frame.sh listening on http://127.0.0.1:1248
Lattice device connected
```

### 2. Get Your Lattice Address

```bash
export FRAME_RPC="http://127.0.0.1:1248"
export LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)

echo "Your Lattice address: $LATTICE_ADDRESS"
```

**Expected output:**

```text
Your Lattice address: 0xeb23e28171099c5edddb1f96cfe38086af1b4852
```

### 3. Verify Frame.sh Connection

```bash
# Check that Frame.sh is responding
cast chain-id --rpc-url $FRAME_RPC
```

**Expected output:**

```text
84532  # Base Sepolia chain ID
```

If you see `1` (Mainnet) or an error, Frame.sh is not running or not configured correctly.

### 4. Run Deployment Script

```bash
cd /Users/psd/Documents/PROJECTS/SOf/sof-alpha/contracts

forge script script/deploy/00_DeployToSepolia.s.sol \
  --rpc-url $FRAME_RPC \
  --sender $LATTICE_ADDRESS \
  --broadcast \
  -vvvv
```

### 5. Approve Transactions on Lattice Device

When the script runs, it will:
1. Show pending transactions in Frame.sh UI
2. Wait for you to approve each transaction on your Lattice device
3. Sign transactions with your private key
4. Submit to Base Sepolia network

**You should see:**
```
 SOF Token deployed: 0x...
 Raffle deployed: 0x...
 SeasonFactory deployed: 0x...
 SOFBondingCurve deployed: 0x...
```

### 6. Verify Deployment

```bash
# Check contract on Block Explorer
# https://sepolia.basescan.org/address/0x...

# Or verify via cast
cast code 0x6f6BB04D19f02a96AF09f7Dc9c6E5165D564f622 --rpc-url https://sepolia.base.org
```

---

## Troubleshooting Checklist

###  "unknown account" Error

**Cause:** Frame.sh not running or not connected

**Fix:**

1. Verify Frame.sh is running: `ps aux | grep frame`
2. Check connection: `cast chain-id --rpc-url http://127.0.0.1:1248`
3. Restart Frame.sh if needed

###  "Chain ID is 1 (Mainnet)" in Logs

**Cause:** Using public RPC instead of Frame.sh local RPC

**Fix:**

```bash
#  WRONG
# ❌ WRONG
forge script ... --rpc-url baseSepolia ...

# ✅ CORRECT
forge script ... --rpc-url http://127.0.0.1:1248 ...
```

### ❌ "Transaction reverted" After Approval

**Cause:** Insufficient funds or contract error

**Fix:**

1. Verify Lattice address has Base Sepolia ETH
2. Check contract code for errors
3. Review transaction details in Frame.sh UI

### ❌ "Lattice device not found"

**Cause:** Lattice device not connected or Frame.sh not configured

**Fix:**

1. Connect Lattice device via USB
2. Unlock Lattice device
3. Restart Frame.sh
4. Check Frame.sh logs for connection status

### ❌ "Timeout waiting for transaction"

**Cause:** Transaction not approved on Lattice device

**Fix:**

1. Check Lattice device screen for pending approval
2. Press physical button to approve
3. Wait for transaction to be signed and submitted

---

## Key Differences: Frame.sh vs Direct Signing

| Aspect | Frame.sh | Direct Signing |
|--------|----------|----------------|
| **RPC URL** | `http://127.0.0.1:1248` (local proxy) | `https://sepolia.base.org` (public) |
| **Signing** | Lattice device (hardware) | Private key in environment (risky) |
| **Security** | ✅ High (hardware wallet) | ❌ Low (key in memory) |
| **Approval** | ✅ Physical button press | ❌ Automatic |
| **Chain ID** | ✅ Correct (84532) | ❌ May be wrong |

---

## Environment Variables

Add to `.env`:

```bash
# Frame.sh Configuration
FRAME_RPC=http://127.0.0.1:1248
LATTICE_ADDRESS=0xeb23e28171099c5edddb1f96cfe38086af1b4852
```

---

## Complete Working Example

```bash
#!/bin/bash

# Start Frame.sh (in background)
frame start &
sleep 2

# Set environment
export FRAME_RPC="http://127.0.0.1:1248"
export LATTICE_ADDRESS=$(cast wallet address --rpc-url $FRAME_RPC)

echo "🔗 Frame.sh RPC: $FRAME_RPC"
echo "👤 Lattice Address: $LATTICE_ADDRESS"

# Verify connection
CHAIN_ID=$(cast chain-id --rpc-url $FRAME_RPC)
echo "⛓️  Chain ID: $CHAIN_ID"

if [ "$CHAIN_ID" != "84532" ]; then
  echo "❌ ERROR: Wrong chain ID! Expected 84532 (Base Sepolia), got $CHAIN_ID"
  exit 1
fi

# Deploy
cd /Users/psd/Documents/PROJECTS/SOf/sof-alpha/contracts

forge script script/deploy/00_DeployToSepolia.s.sol \
  --rpc-url $FRAME_RPC \
  --sender $LATTICE_ADDRESS \
  --broadcast \
  -vvvv

echo "✅ Deployment complete!"
```

---

## Summary

**The key insight:** Frame.sh provides a **local RPC proxy** that intercepts transactions and routes them to your Lattice device for signing. Always use `http://127.0.0.1:1248` when deploying with Frame.sh, never the public RPC endpoint.
