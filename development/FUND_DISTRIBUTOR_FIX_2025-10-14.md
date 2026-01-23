# Fund Distributor Fix - October 14, 2025

## Problem Summary

The fund distributor functionality in the Admin Panel was failing due to an **ABI mismatch** between the manually defined `RaffleMiniAbi` and the actual Raffle.sol contract interface.

## Root Cause Analysis

### The Issue

The `RaffleMiniAbi` defined in `src/routes/AdminPanel.jsx` had an **incorrect definition** for the `getSeasonDetails` function that didn't match the actual smart contract.

### Specific Mismatches

#### 1. Wrong Return Structure

**Incorrect ABI** (AdminPanel.jsx lines 44-69):
```javascript
outputs: [
  {
    name: "config",
    type: "tuple",
    components: [
      { name: "name", type: "string" },
      { name: "startTime", type: "uint256" },
      { name: "endTime", type: "uint256" },
      { name: "winnerCount", type: "uint8" },           // WRONG TYPE!
      { name: "prizePercentage", type: "uint8" },       // DOESN'T EXIST!
      { name: "consolationPercentage", type: "uint8" }, // DOESN'T EXIST!
      { name: "grandPrizeBps", type: "uint16" },
      { name: "raffleToken", type: "address" },
      { name: "bondingCurve", type: "address" },
      { name: "isActive", type: "bool" },
      { name: "isCompleted", type: "bool" },
    ],
  },
  { name: "status", type: "uint8" },
  { name: "totalTickets", type: "uint256" },
  { name: "winner", type: "address" },      // DOESN'T EXIST!
  { name: "merkleRoot", type: "bytes32" },  // DOESN'T EXIST!
]
```

**Actual Contract** (Raffle.sol lines 360-373):
```solidity
function getSeasonDetails(uint256 seasonId) external view returns (
    RaffleTypes.SeasonConfig memory config,
    SeasonStatus status,
    uint256 totalParticipants,  // MISSING FROM ABI!
    uint256 totalTickets,
    uint256 totalPrizePool      // MISSING FROM ABI!
)
```

**Actual SeasonConfig Struct** (RaffleTypes.sol):
```solidity
struct SeasonConfig {
    string name;
    uint256 startTime;
    uint256 endTime;
    uint16 winnerCount;      // NOT uint8!
    uint16 grandPrizeBps;
    address raffleToken;
    address bondingCurve;
    bool isActive;
    bool isCompleted;
    // NO prizePercentage or consolationPercentage!
}
```

#### 2. Impact on useFundDistributor

The `useFundDistributor.js` hook was trying to destructure values that didn't exist:

```javascript
// Line 54 - This was getting wrong/undefined values
const [, status, totalParticipants, totalTickets, totalPrizePool] = seasonDetails;

// Line 360 - This was accessing wrong index for bondingCurve
const bondingCurveAddr = seasonDetails[0][8]; // Index 8 doesn't exist!
```

## The Fix

### Changes Made

#### 1. Fixed AdminPanel.jsx RaffleMiniAbi

**File:** `src/routes/AdminPanel.jsx`

Updated the `getSeasonDetails` function definition to match the actual contract:

```javascript
{
  type: "function",
  name: "getSeasonDetails",
  stateMutability: "view",
  inputs: [{ name: "seasonId", type: "uint256" }],
  outputs: [
    {
      name: "config",
      type: "tuple",
      components: [
        { name: "name", type: "string" },
        { name: "startTime", type: "uint256" },
        { name: "endTime", type: "uint256" },
        { name: "winnerCount", type: "uint16" },      // Fixed type
        { name: "grandPrizeBps", type: "uint16" },
        { name: "raffleToken", type: "address" },
        { name: "bondingCurve", type: "address" },    // Now at index 6
        { name: "isActive", type: "bool" },
        { name: "isCompleted", type: "bool" },
        // Removed non-existent fields
      ],
    },
    { name: "status", type: "uint8" },
    { name: "totalParticipants", type: "uint256" },   // Added
    { name: "totalTickets", type: "uint256" },
    { name: "totalPrizePool", type: "uint256" },      // Added
    // Removed non-existent winner and merkleRoot
  ],
}
```

#### 2. Fixed useFundDistributor.js bondingCurve Index

**File:** `src/hooks/useFundDistributor.js`

Updated line 362 to access the correct index:

```javascript
// OLD (WRONG):
const bondingCurveAddr = seasonDetails[0][8]; // Index 8 was out of bounds

// NEW (CORRECT):
const bondingCurveAddr = seasonDetails[0][6]; // bondingCurve is at index 6
```

Added comment explaining the structure:
```javascript
// seasonDetails[0] is the config tuple
// bondingCurve is at index 6 in the config tuple 
// (after name, startTime, endTime, winnerCount, grandPrizeBps, raffleToken)
```

## Verification

### How to Verify the Fix

1. **Check ABI matches contract:**
   ```bash
   # View the actual contract function
   grep -A 20 "function getSeasonDetails" contracts/src/core/Raffle.sol
   
   # Compare with the generated ABI
   grep -A 100 '"name": "getSeasonDetails"' src/contracts/abis/Raffle.json
   ```

2. **Test the fund distributor:**
   - Start a local Anvil instance
   - Deploy contracts and create a season
   - Complete the season (buy tickets, request end, fulfill VRF)
   - Use the Admin Panel "Fund Distributor" button
   - Should now successfully extract SOF and fund the distributor

### Expected Behavior After Fix

1. `getSeasonDetails` returns correct data structure
2. `totalParticipants` and `totalPrizePool` are correctly extracted
3. `bondingCurve` address is correctly accessed at index 6
4. Fund distributor completes without errors
5. SOF balance updates correctly after claims

## Prevention

### Best Practices Going Forward

1. **Use generated ABIs:** Always import from `src/contracts/abis/*.json` instead of manually defining ABIs
2. **Verify ABI matches contract:** When updating contracts, regenerate ABIs with `npm run copy-abis`
3. **Add type checking:** Consider using TypeScript for better type safety with contract interactions
4. **Document ABI structure:** Add comments explaining tuple field indices when accessing nested data

### Recommended Improvement

Instead of manually maintaining `RaffleMiniAbi`, import from the generated ABI:

```javascript
import RaffleAbi from '@/contracts/abis/Raffle.json';

// Extract only needed functions
const RaffleMiniAbi = RaffleAbi.filter(item => 
  ['getSeasonDetails', 'getWinners', 'requestSeasonEndEarly', 'getVrfRequestForSeason'].includes(item.name)
);
```

## Related Files

- `src/routes/AdminPanel.jsx` - Contains RaffleMiniAbi definition
- `src/hooks/useFundDistributor.js` - Uses the ABI to interact with contract
- `contracts/src/core/Raffle.sol` - The actual smart contract
- `contracts/src/lib/RaffleTypes.sol` - Defines SeasonConfig struct
- `src/contracts/abis/Raffle.json` - Generated ABI (source of truth)

## Previous Related Fixes

This fix builds on previous work documented in system memory:
- Fixed Viem v2 `account` parameter requirement
- Added React Query cache invalidation for balance updates
- Cleaned up debug logs and lint errors

## Testing Checklist

- [ ] ABI matches actual contract interface
- [ ] Fund distributor completes without errors
- [ ] SOF is correctly extracted from bonding curve
- [ ] Prize distributor is correctly funded
- [ ] User balance updates after claim
- [ ] No console errors during execution
- [ ] Works on both local Anvil and testnet

## Notes

The console.log warnings in `useFundDistributor.js` are pre-existing debug statements. These can be removed in a future cleanup but are not blocking the fix.
