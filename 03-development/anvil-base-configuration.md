# Anvil Configuration for Base Network Simulation

## Overview

Updated Anvil local development environment to closely match Base network parameters for more realistic testing.

## Base Network Parameters

| Parameter | Base Mainnet | Previous Anvil | New Anvil |
|-----------|--------------|----------------|-----------|
| Block Time | 24 seconds | 2 seconds | **24 seconds** |
| Gas Price | 0.084 Gwei | 0.001 Gwei | **0.084 Gwei** |
| Base Fee | 0.084 Gwei | 0 | **0.084 Gwei** |
| Gas Limit | ~30M | Default | **30M** |
| Simple TX Gas | 21,000 | 21,000 | 21,000 |

## Changes Made

### package.json Scripts Updated

**1. `npm run anvil`**

```bash
# Before
anvil -p 8545 --chain-id 31337 --block-base-fee-per-gas 0 --block-time 2

# After
anvil -p 8545 --chain-id 31337 --block-base-fee-per-gas 84000000 --gas-price 84000000 --block-time 24 --gas-limit 30000000
```

**2. `npm run anvil:deploy`**

Updated the embedded Anvil command to match the same parameters.

## Parameter Details

### Block Time: 24 seconds

- **Why**: Base produces blocks every ~24 seconds
- **Impact**:
  - Season timing more realistic
  - VRF callback timing matches production
  - User experience closer to mainnet

### Gas Price: 0.084 Gwei (84,000,000 wei)

- **Why**: Base's standard priority gas price
- **Impact**:
  - Transaction costs match production estimates
  - Gas optimization testing more accurate
  - Economic modeling realistic

### Gas Limit: 30,000,000

- **Why**: Base's block gas limit
- **Impact**:
  - Complex transactions (like market creation) can be tested
  - Batch operations realistic
  - No artificial gas constraints

## Testing Implications

### Positive Changes

✅ **Realistic timing**: 24-second blocks mean:

- Season durations feel real
- Event polling intervals can be tuned for production
- Race conditions more likely to surface

✅ **Accurate gas costs**: 0.084 Gwei means:

- Transaction cost estimates match mainnet
- Gas optimization has real impact
- Economic models accurate

✅ **Complex operations**: 30M gas limit means:

- InfoFi market creation works (needs ~500K gas)
- Batch operations possible
- No artificial constraints

### Considerations

⚠️ **Slower development**: 24-second blocks mean:

- Waiting longer between transactions
- Testing takes more time
- Can use `anvil_mine` RPC for fast-forward when needed

⚠️ **Higher gas costs**: 0.084 Gwei means:

- Accounts need more ETH for testing
- Gas optimization more important
- Transaction failures more expensive

## Quick Commands

### Fast-Forward Time (for testing)

```bash
# Mine 1 block immediately
cast rpc anvil_mine 1

# Mine 100 blocks (simulate ~40 minutes)
cast rpc anvil_mine 100

# Set next block timestamp (Unix timestamp)
cast rpc anvil_setNextBlockTimestamp 1730000000
```

### Check Current Block Time

```bash
# Get latest block
cast block latest --rpc-url http://127.0.0.1:8545

# Get block time
cast block latest --rpc-url http://127.0.0.1:8545 --json | jq .timestamp
```

### Check Gas Price

```bash
# Get current gas price
cast gas-price --rpc-url http://127.0.0.1:8545

# Should return: 84000000 (0.084 Gwei)
```

## Reverting to Fast Mode (if needed)

If you need faster testing, you can temporarily use:

```bash
# Fast mode (2 second blocks, low gas)
anvil -p 8545 --chain-id 31337 --block-base-fee-per-gas 0 --gas-price 1000000 --block-time 2 --gas-limit 30000000
```

## References

- [Base Network Stats](https://basescan.org/)
- [Anvil Documentation](https://book.getfoundry.sh/reference/anvil/)
- [Base Gas Tracker](https://basescan.org/gastracker)

## Status

✅ **Configuration Updated**
⏳ **Awaiting Testing**

## Next Steps

1. Restart Anvil with new configuration: `npm run kill:zombies && npm run anvil`
2. Deploy contracts: `npm run anvil:deploy` (in new terminal)
3. Test market creation with realistic gas limits
4. Verify 24-second block timing
5. Monitor gas costs match expectations
