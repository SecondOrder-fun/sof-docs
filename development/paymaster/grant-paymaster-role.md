# Grant PAYMASTER_ROLE to Backend Wallet

## Issue

The backend wallet needs `PAYMASTER_ROLE` on InfoFiMarketFactory to call `onPositionUpdate()`.

## Current Status

```bash
# Check if backend wallet has the role (returns false)
cast call 0xB654A23D56B677C573d92BB69760A12ea3cDf9f9 \
  "hasRole(bytes32,address)(bool)" \
  0x3a43d4210774a09c5d26191d1150219028070cec811f3e6f8ab498268dd6140c \
  0x1eD4aC856D7a072C3a336C0971a47dB86A808Ff4 \
  --rpc-url https://sepolia.base.org

# Output: false ❌
```

## Solution

Grant the role using the admin account:

```bash
# Set environment variable for private key
export PRIVATE_KEY=99593f2b6808e237a23806fc06f8ad5f76987b01d69e31425a13afcefbfaa826

# Grant PAYMASTER_ROLE to backend wallet
cast send 0xB654A23D56B677C573d92BB69760A12ea3cDf9f9 \
  "setPaymasterAccount(address)" \
  0x1eD4aC856D7a072C3a336C0971a47dB86A808Ff4 \
  --rpc-url https://sepolia.base.org \
  --private-key $PRIVATE_KEY
```

## Verify

After granting, verify the role was granted:

```bash
cast call 0xB654A23D56B677C573d92BB69760A12ea3cDf9f9 \
  "hasRole(bytes32,address)(bool)" \
  0x3a43d4210774a09c5d26191d1150219028070cec811f3e6f8ab498268dd6140c \
  0x1eD4aC856D7a072C3a336C0971a47dB86A808Ff4 \
  --rpc-url https://sepolia.base.org

# Should output: true ✅
```

## Addresses

- **InfoFiMarketFactory**: `0xB654A23D56B677C573d92BB69760A12ea3cDf9f9`
- **Backend Wallet**: `0x1eD4aC856D7a072C3a336C0971a47dB86A808Ff4`
- **PAYMASTER_ROLE**: `0x3a43d4210774a09c5d26191d1150219028070cec811f3e6f8ab498268dd6140c`

## Why Backend Wallet?

In our simplified implementation:

- Backend wallet (EOA) signs transactions
- Paymaster RPC sponsors gas (if contract is in allowlist)
- No separate smart account needed

The backend wallet needs PAYMASTER_ROLE because it's the one calling `onPositionUpdate()`.
