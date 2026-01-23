# Broadcasting via Cast Commands

**Status:** Ready to execute  
**Method:** Direct cast commands through Frame.sh RPC

---

## Setup

```bash
export FRAME_RPC="http://127.0.0.1:1248"
export SENDER="0xeb23e28171099c5edddb1f96cfe38086af1b4852"
export SEPOLIA_RPC="https://sepolia.base.org"
```

---

## Transaction 1: Deploy SOF Token

```bash
cast create \
  --rpc-url $FRAME_RPC \
  --from $SENDER \
  --constructor-args \
    $(cast abi-encode "constructor(string,string,uint256,address)" \
      "SecondOrder Fun" "SOF" "100000000000000000000000000" "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266") \
  --bytecode-path contracts/out/SOFToken.sol/SOFToken.json
```

**Expected Output:** SOF Token address

---

## Transaction 2: Deploy Raffle

After getting SOF Token address, run:

```bash
SOF_ADDRESS="0x..." # Replace with address from Tx1

cast create \
  --rpc-url $FRAME_RPC \
  --from $SENDER \
  --constructor-args \
    $(cast abi-encode "constructor(address,address,uint64,bytes32)" \
      "$SOF_ADDRESS" \
      "0xd5d517ABe5Cf79B7e95eC98DB0f0277788aF8a45" \
      "0" \
      "0x9e1b49a3bfe90b5c1445b41f6ca7c0ad713d7c8496ada80757bdd57502ccf60b") \
  --bytecode-path contracts/out/Raffle.sol/Raffle.json
```

**Expected Output:** Raffle address

---

## Simpler Alternative: Use Deployment Addresses

Since we already have the simulated addresses from the script, you can:

1. **Copy the broadcast file** to a safe location
2. **Use Frame.sh to manually approve** each transaction
3. **Or use Lattice directly** to sign transactions

---

## Recommended: Use Frame.sh Web Interface

The easiest way is to:

1. **Open Frame.sh** in your browser
2. **Go to "Send Transaction"**
3. **Paste the contract bytecode** and constructor args
4. **Approve on Lattice device**

---

## Contract Bytecodes

Get the bytecodes from compiled contracts:

```bash
# SOF Token
cat contracts/out/SOFToken.sol/SOFToken.json | jq -r '.bytecode.object'

# Raffle
cat contracts/out/Raffle.sol/Raffle.json | jq -r '.bytecode.object'

# SeasonFactory
cat contracts/out/SeasonFactory.sol/SeasonFactory.json | jq -r '.bytecode.object'

# SOFBondingCurve
cat contracts/out/SOFBondingCurve.sol/SOFBondingCurve.json | jq -r '.bytecode.object'
```

---

## Status

The broadcast file is ready at:
```
contracts/broadcast/00_DeployToSepolia.s.sol/1/run-latest.json
```

You can use this file as reference for all transaction details.

