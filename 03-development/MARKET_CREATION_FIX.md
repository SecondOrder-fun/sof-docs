# Market Creation Fix - Treasury Approval Issue

## Problem Summary

Market creation was failing with "Unknown error" even though the transaction succeeded on-chain. Investigation revealed that the `InfoFiMarketFactory` contract was emitting a `MarketCreationFailed` event instead of `MarketCreated`.

## Root Cause

The `InfoFiMarketFactory` contract attempts to pull SOF tokens from the treasury using `transferFrom()`:

```solidity
sofToken.transferFrom(treasury, address(this), INITIAL_LIQUIDITY)
```

**However, the treasury never approved the factory to spend its tokens.** This causes the `transferFrom()` to fail, which the contract's try-catch block catches and emits as "Unknown error".

## Evidence

Transaction `0xbc6526516b31fa7ecf0a46dcb35dd13f610b91d79d6b8aae876ca7ce9c62407f` shows:

- Status: Success (transaction didn't revert)
- Event emitted: `MarketCreationFailed` with reason "Unknown error"
- No `MarketCreated` event

The contract's defensive error handling prevented the transaction from reverting, but market creation still failed internally.

## Solution

### 1. Enhanced Error Messages (Implemented)

Added detailed checks in `InfoFiMarketFactory.sol`:

```solidity
// Check treasury allowance first
uint256 treasuryAllowance = sofToken.allowance(treasury, address(this));
require(treasuryAllowance >= INITIAL_LIQUIDITY, string(abi.encodePacked(
    "Treasury allowance insufficient: has ",
    _uint2str(treasuryAllowance),
    " needs ",
    _uint2str(INITIAL_LIQUIDITY)
)));

// Check treasury balance
uint256 treasuryBalance = sofToken.balanceOf(treasury);
require(treasuryBalance >= INITIAL_LIQUIDITY, string(abi.encodePacked(
    "Treasury balance insufficient: has ",
    _uint2str(treasuryBalance),
    " needs ",
    _uint2str(INITIAL_LIQUIDITY)
)));
```

Now the error message will clearly state:

- "Treasury allowance insufficient: has 0 needs 100000000000000000000"
- Or "Treasury balance insufficient: has X needs Y"

### 2. Treasury Approval Setup (Required)

The treasury must approve the factory before any markets can be created:

```bash
# Run the setup script
forge script script/admin/SetupTreasuryApproval.s.sol:SetupTreasuryApproval \
  --rpc-url https://sepolia.base.org \
  --private-key $TREASURY_PRIVATE_KEY \
  --broadcast
```

Or manually via cast:

```bash
# Approve factory to spend 1M SOF from treasury
cast send $SOF_TOKEN \
  "approve(address,uint256)" \
  $INFOFI_FACTORY \
  1000000000000000000000000 \
  --rpc-url https://sepolia.base.org \
  --private-key $TREASURY_PRIVATE_KEY
```

### 3. Verify Approval

```bash
# Check current allowance
cast call $SOF_TOKEN \
  "allowance(address,address)(uint256)" \
  $TREASURY_ADDRESS \
  $INFOFI_FACTORY \
  --rpc-url https://sepolia.base.org
```

## Deployment Steps

1. **Deploy updated InfoFiMarketFactory** (with enhanced error messages)
2. **Run treasury approval script** (as treasury account)
3. **Verify approval** is set correctly
4. **Test market creation** with a new purchase

## Testing

After deploying the fix and setting up approval:

1. Make a purchase that crosses 1% threshold
2. Check logs for detailed error messages (if any)
3. Verify `MarketCreated` event is emitted
4. Confirm market appears in `infofi_markets` table

## Prevention

### For Future Deployments

1. Always run treasury approval setup immediately after deploying InfoFiMarketFactory
2. Document the approval requirement in deployment guides
3. Add a view function to check if treasury approval is sufficient:

```solidity
function isTreasuryApprovalSufficient() external view returns (bool) {
    return sofToken.allowance(treasury, address(this)) >= INITIAL_LIQUIDITY * 10;
}
```

### Monitoring

Add alerts for:

- Treasury allowance falling below 10x INITIAL_LIQUIDITY
- Treasury balance falling below 10x INITIAL_LIQUIDITY
- Any `MarketCreationFailed` events

## Related Files

- Contract: `/contracts/src/infofi/InfoFiMarketFactory.sol`
- Setup Script: `/contracts/script/admin/SetupTreasuryApproval.s.sol`
- Backend Listener: `/backend/src/listeners/marketCreatedListener.js`

## Timeline

- **Issue Discovered**: 2025-12-08
- **Root Cause Identified**: Treasury approval missing
- **Fix Implemented**: Enhanced error messages + approval script
- **Status**: Ready for deployment and testing
