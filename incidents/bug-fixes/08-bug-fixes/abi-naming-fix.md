# ABI Naming Fix: SimpleFPMMAbi → InfoFiFPMMV2Abi

## Problem

The backend had a file named `SimpleFPMMAbi.js` which was misleading because:

- **No external contract named SimpleFPMM exists** - SimpleFPMM is defined inside `InfoFiFPMMV2.sol`
- **Naming mismatch** - The ABI file should match the actual contract file name in the project
- **Confusion for developers** - Suggests SimpleFPMM is a separate external contract

## Solution

### Files Changed

**Renamed**:

- ❌ `backend/src/abis/SimpleFPMMAbi.js` (old)
- ✅ `backend/src/abis/InfoFiFPMMV2Abi.js` (new)

**Updated header comment**:

```javascript
// Auto-generated from InfoFiFPMMV2.sol (SimpleFPMM contract)
// Do not edit manually - run 'npm run copy-abis' to regenerate
```

### Why This Matters

The ABI file now correctly reflects:

- **Source file**: `contracts/src/infofi/InfoFiFPMMV2.sol`
- **Contract name**: `SimpleFPMM` (the actual contract being used)
- **File organization**: Matches the actual project structure

### Contract Structure

In `InfoFiFPMMV2.sol`, there are actually TWO contracts:

- **SOLPToken** - LP token for FPMM markets
- **SimpleFPMM** - The fixed product market maker (this is what the ABI represents)
- **InfoFiFPMMV2** - Factory contract for creating FPMM markets

The ABI file represents the **SimpleFPMM** contract, which is deployed by **InfoFiFPMMV2**.

### Next Steps

When `marketCreatedListener.js` is updated to store FPMM addresses, it should import:

```javascript
import InfoFiFPMMV2Abi from "../abis/InfoFiFPMMV2Abi.js";
```

This makes it clear that:

- The ABI comes from the InfoFiFPMMV2 file
- It represents the SimpleFPMM contract
- It's used for FPMM market interactions

## Files to Update (When Needed)

Any backend files that reference the old ABI name should be updated:

```javascript
// OLD
import SimpleFPMMAbi from "../abis/SimpleFPMMAbi.js";

// NEW
import InfoFiFPMMV2Abi from "../abis/InfoFiFPMMV2Abi.js";
```

Currently, no files reference this ABI, so the rename is clean with no breaking changes.
