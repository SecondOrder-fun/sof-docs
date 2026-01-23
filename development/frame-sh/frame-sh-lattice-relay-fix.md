# Frame.sh + Lattice Relay: Base Sepolia Configuration Fix

**Date:** Nov 11, 2025  
**Issue:** Frame.sh returning Chain ID 1 (Mainnet) instead of 84532 (Base Sepolia)  
**Solution:** Configure Lattice Relay to use Base Sepolia RPC

---

## The Problem

Your Frame.sh is currently configured to use **Mainnet** (Chain ID 1) instead of **Base Sepolia** (Chain ID 84532).

**Evidence:**
```bash
$ cast chain-id --rpc-url $FRAME_RPC
1  # ❌ WRONG - This is Mainnet
```

Should be:
```bash
$ cast chain-id --rpc-url $FRAME_RPC
84532  # ✅ CORRECT - This is Base Sepolia
```

---

## The Solution: Configure Lattice Relay

According to official GridPlus documentation, you need to configure the **Lattice Relay** to use Base Sepolia.

### Step 1: Open Frame Settings

1. **Open Frame.sh application**
2. **Click the menu button icon** (top right)
3. **Scroll down to find "Lattice Relay"**

### Step 2: Switch to Custom Relay URL

**From GridPlus docs:**
> "open the Settings menu by clicking on the menu button icon and scroll down to Lattice Relay and switch it to your custom relay URL"

**Options:**

#### Option A: Use Public Base Sepolia RPC (Simplest)

In Frame Settings → Lattice Relay, enter:
```
https://sepolia.base.org
```

#### Option B: Use Alchemy Base Sepolia RPC (Recommended)

If you have an Alchemy account:
```
https://base-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_KEY
```

#### Option C: Run Your Own Relay (Advanced)

For production, you can run your own Lattice Relay:
- GitHub: https://github.com/GridPlus/lattice-connect-v2
- Requires: Node.js + running relay service
- Configuration: Update Lattice Manager with relay endpoint

### Step 3: Verify Configuration

After updating the Lattice Relay, restart Frame.sh and verify:

```bash
cast chain-id --rpc-url $FRAME_RPC
```

Should now return: `84532` ✅

---

## How Lattice Relay Works

**Architecture:**

```
Frame.sh (localhost:1248)
    ↓
Lattice Relay (messaging proxy)
    ↓
Base Sepolia RPC (https://sepolia.base.org)
    ↓
Lattice Device (USB)
```

**What Lattice Relay Does:**

1. **Intercepts Frame.sh requests** from `http://127.0.0.1:1248`
2. **Routes to configured RPC** (Base Sepolia in this case)
3. **Handles Lattice device communication** securely
4. **Signs transactions** with your hardware wallet

---

## Complete Configuration Steps

### Quick Fix (5 minutes)

1. **Open Frame.sh**
2. **Settings → Lattice Relay**
3. **Enter:** `https://sepolia.base.org`
4. **Restart Frame.sh**
5. **Verify:** `cast chain-id --rpc-url $FRAME_RPC` → should return `84532`

### Verify Everything Works

```bash
# 1. Check chain ID
cast chain-id --rpc-url $FRAME_RPC
# Expected: 84532

# 2. Get Lattice address
cast rpc eth_accounts --rpc-url $FRAME_RPC
# Expected: ["0xeb23e28171099c5edddb1f96cfe38086af1b4852"]

# 3. Check Lattice address has Base Sepolia ETH
cast balance 0xeb23e28171099c5edddb1f96cfe38086af1b4852 --rpc-url $FRAME_RPC
# Expected: > 0 (some ETH for gas)
```

---

## If You Need Your Own Relay (Advanced)

For production deployments, you can run your own Lattice Relay:

### Option 1: lattice-connect-v2 (Recommended)

```bash
# Clone the repo
git clone https://github.com/GridPlus/lattice-connect-v2.git
cd lattice-connect-v2

# Install dependencies
npm install

# Start relay on http://localhost:8080
npm start
```

Then in Frame Settings → Lattice Relay:
```
http://localhost:8080
```

### Option 2: Docker

```bash
docker run -p 8080:8080 gridplus/lattice-connect-v2
```

### Configuration in Lattice Manager

1. **Go to:** https://lattice.gridplus.io/
2. **Log in with your Lattice**
3. **Settings tab**
4. **Connection Endpoint field:** Enter your relay URL
   ```
   http://YOUR_RELAY_HOST:8080
   ```

---

## Troubleshooting

### ❌ Still showing Chain ID 1?

1. **Restart Frame.sh completely** (close and reopen)
2. **Verify Lattice Relay setting** was saved
3. **Check internet connection** to Base Sepolia RPC
4. **Try a different RPC URL:**
   - `https://sepolia.base.org` (official)
   - `https://base-sepolia.publicnode.com` (public node)
   - `https://base-sepolia.blockpi.network/v1/rpc/public` (BlockPI)

### ❌ "Connection refused" error?

1. **Verify Frame.sh is running**
2. **Check Lattice device is connected** via USB
3. **Unlock Lattice device**
4. **Restart Frame.sh**

### ❌ "Invalid RPC URL"?

1. **Verify URL is correct** (no typos)
2. **Test URL directly:**
   ```bash
   curl https://sepolia.base.org -X POST -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
   ```
3. **Should return:** `{"jsonrpc":"2.0","result":"0x14a94","id":1}` (0x14a94 = 84532 in hex)

---

## Official Documentation

- **Frame Advanced Setup:** https://docs.gridplus.io/apps-and-integrations/frame/advanced-frame-setup
- **Lattice Manager:** https://docs.gridplus.io/apps-and-integrations/lattice-manager
- **Private Endpoint Setup:** https://docs.gridplus.io/apps-and-integrations/lattice-manager/connecting-your-lattice-to-your-own-private-endpoint
- **lattice-connect-v2 GitHub:** https://github.com/GridPlus/lattice-connect-v2

---

## Next Steps

1. ✅ Configure Lattice Relay to use Base Sepolia RPC
2. ✅ Restart Frame.sh
3. ✅ Verify chain ID is 84532
4. ✅ Verify Lattice address has Base Sepolia ETH
5. ✅ Run deployment script:
   ```bash
   export FRAME_RPC="http://127.0.0.1:1248"
   export LATTICE_ADDRESS=$(cast rpc eth_accounts --rpc-url $FRAME_RPC | jq -r '.[0]')
   
   forge script script/deploy/00_DeployToSepolia.s.sol \
     --rpc-url $FRAME_RPC \
     --sender $LATTICE_ADDRESS \
     --broadcast \
     -vvvv
   ```

---

## Summary

The issue was that **Lattice Relay** (Frame.sh's messaging proxy) was configured to use the default Mainnet RPC instead of Base Sepolia.

**Fix:** Change Lattice Relay RPC endpoint to `https://sepolia.base.org` in Frame Settings.

**Result:** Frame.sh will then route all transactions through Base Sepolia, and you can deploy your contracts! 🚀

