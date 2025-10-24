# ABI Management System

## Overview

The SecondOrder.fun project uses a centralized ABI management system that automatically copies contract ABIs from Foundry output to both frontend and backend directories in the appropriate formats.

## Architecture

### Source of Truth
- **Foundry Compilation**: `forge build` → `contracts/out/`
- **Single Script**: `scripts/copy-abis.js` handles all ABI distribution

### Distribution Targets

#### Frontend ABIs
- **Location**: `src/contracts/abis/`
- **Format**: JSON files (e.g., `SOFBondingCurve.json`)
- **Usage**: Direct JSON imports in React components
- **Example**:
  ```javascript
  import SOFBondingCurveAbi from '@/contracts/abis/SOFBondingCurve.json';
  ```

#### Backend ABIs
- **Location**: `backend/src/abis/`
- **Format**: ES Module .js files (e.g., `SOFBondingCurveAbi.js`)
- **Usage**: ES module imports in Node.js services
- **Example**:
  ```javascript
  import SOFBondingCurveAbi from '../abis/SOFBondingCurveAbi.js';
  ```

## How It Works

### 1. Contract Compilation
```bash
cd contracts
forge build
# Output: contracts/out/ContractName.sol/ContractName.json
```

### 2. ABI Extraction & Distribution
```bash
npm run copy-abis
```

This script:
1. Reads compiled contracts from `contracts/out/`
2. Extracts ABI arrays from Foundry JSON output
3. Copies to frontend as JSON files
4. Converts to ES modules for backend

### 3. Automatic Integration
The `copy-abis` script runs automatically in deployment flows:
- `npm run anvil:deploy` - Local Anvil deployment
- `npm run deploy:testnet` - Testnet deployment
- Manual: `npm run copy-abis` - Standalone execution

## Backend ABI Requirements

Backend services need these ABIs (defined in `scripts/copy-abis.js`):

```javascript
const backendAbiNeeds = [
  'SOFBondingCurve.json',      // For bondingCurveListener
  'InfoFiMarketFactory.json',  // For infoFiMarketCreator
  'InfoFiPriceOracle.json',    // For oracleListener
  'Raffle.json',               // For raffleListener
  'RafflePositionTracker.json', // For positionTrackerListener
];
```

### Backend Services Using ABIs

1. **bondingCurveListener.js**
   - Watches `PositionUpdate` events
   - Triggers InfoFi market creation
   - Uses: `SOFBondingCurveAbi.js`

2. **infoFiMarketCreator.js**
   - Calls `InfoFiMarketFactory.onPositionUpdate()`
   - Backend wallet transactions
   - Uses: `InfoFiMarketFactoryAbi.js`

3. **oracleListener.js**
   - Monitors price oracle updates
   - Real-time pricing coordination
   - Uses: `InfoFiPriceOracleAbi.js`

4. **raffleListener.js**
   - Watches raffle events
   - Season lifecycle tracking
   - Uses: `RaffleAbi.js`

5. **positionTrackerListener.js**
   - Monitors position snapshots
   - Player position tracking
   - Uses: `RafflePositionTrackerAbi.js`

## File Format Examples

### Frontend ABI (JSON)
```json
[
  {
    "type": "event",
    "name": "PositionUpdate",
    "inputs": [
      {
        "name": "seasonId",
        "type": "uint256",
        "indexed": true
      }
    ]
  }
]
```

### Backend ABI (ES Module)
```javascript
// Auto-generated from SOFBondingCurve.json
// Do not edit manually - run 'npm run copy-abis' to regenerate

export default [
  {
    "type": "event",
    "name": "PositionUpdate",
    "inputs": [
      {
        "name": "seasonId",
        "type": "uint256",
        "indexed": true
      }
    ]
  }
];
```

## Adding New ABIs

### For Frontend Only

1. Add to `contractsToCopy` in `scripts/copy-abis.js`:
   ```javascript
   { sourceFile: 'NewContract.sol/NewContract.json', destFile: 'NewContract.json' },
   ```

2. Run `npm run copy-abis`

3. Import in frontend:
   ```javascript
   import NewContractAbi from '@/contracts/abis/NewContract.json';
   ```

### For Backend

1. Add to `contractsToCopy` (if not already there)

2. Add to `backendAbiNeeds`:
   ```javascript
   const backendAbiNeeds = [
     // ... existing
     'NewContract.json',
   ];
   ```

3. Run `npm run copy-abis`

4. Import in backend service:
   ```javascript
   import NewContractAbi from '../abis/NewContractAbi.js';
   ```

## Troubleshooting

### "Cannot find module" Error

**Symptom:**
```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '.../backend/src/abis/ContractAbi.js'
```

**Solution:**
```bash
# Regenerate ABIs
npm run copy-abis

# Verify file exists
ls backend/src/abis/
```

### ABI Out of Date

**Symptom:**
- Contract function not found
- Event signature mismatch
- Type errors in viem calls

**Solution:**
```bash
# Recompile contracts
cd contracts
forge build

# Copy updated ABIs
cd ..
npm run copy-abis
```

### Backend ABI Not Generated

**Symptom:**
- Frontend ABI exists but backend ABI missing

**Cause:**
- Contract not in `backendAbiNeeds` list

**Solution:**
1. Add contract to `backendAbiNeeds` in `scripts/copy-abis.js`
2. Run `npm run copy-abis`

## Best Practices

### 1. Always Regenerate After Contract Changes
```bash
# After modifying contracts
cd contracts
forge build
cd ..
npm run copy-abis
```

### 2. Don't Edit Generated Files
Backend ABI files have this warning:
```javascript
// Auto-generated from ContractName.json
// Do not edit manually - run 'npm run copy-abis' to regenerate
```

### 3. Commit Generated ABIs
- Frontend ABIs: ✅ Commit to Git
- Backend ABIs: ✅ Commit to Git
- Reason: Ensures consistent ABIs across team

### 4. Verify After Deployment
```bash
# After deploying contracts
npm run copy-abis
git status  # Check for ABI changes
git diff    # Review ABI changes
```

## Integration with Deployment

### Local Anvil
```bash
npm run anvil:deploy
# Automatically runs: forge script → copy-abis → update-env
```

### Testnet/Mainnet
```bash
npm run deploy:testnet
# Automatically runs: forge script → copy-abis → update-env
```

### Manual Workflow
```bash
# 1. Compile contracts
cd contracts && forge build && cd ..

# 2. Deploy contracts
cd contracts && forge script script/Deploy.s.sol --broadcast && cd ..

# 3. Copy ABIs
npm run copy-abis

# 4. Update environment
npm run update-env
```

## Directory Structure

```
project-root/
├── contracts/
│   └── out/                          # Foundry compilation output
│       └── ContractName.sol/
│           └── ContractName.json     # Full Foundry output
├── src/
│   └── contracts/
│       └── abis/                     # Frontend ABIs (JSON)
│           ├── SOFBondingCurve.json
│           ├── InfoFiMarketFactory.json
│           └── ...
├── backend/
│   └── src/
│       └── abis/                     # Backend ABIs (ES Modules)
│           ├── SOFBondingCurveAbi.js
│           ├── InfoFiMarketFactoryAbi.js
│           └── ...
└── scripts/
    └── copy-abis.js                  # ABI distribution script
```

## Version Control

### What to Commit
- ✅ `scripts/copy-abis.js` - ABI distribution logic
- ✅ `src/contracts/abis/*.json` - Frontend ABIs
- ✅ `backend/src/abis/*.js` - Backend ABIs

### What to Ignore
- ❌ `contracts/out/` - Foundry build artifacts
- ❌ `contracts/cache/` - Foundry cache

### .gitignore
```gitignore
# Foundry
contracts/out/
contracts/cache/
contracts/broadcast/

# Note: ABIs in src/ and backend/src/abis/ are committed
```

## Performance Considerations

### Build Time
- **Foundry compilation**: ~5-10 seconds
- **ABI copying**: <1 second
- **Total overhead**: Minimal

### File Sizes
- **Frontend JSON**: ~5-50 KB per ABI
- **Backend JS**: ~5-50 KB per ABI (same data, different format)
- **Total**: ~500 KB for all ABIs

### Optimization
- Only backend-needed ABIs are converted to JS
- Frontend gets all ABIs (for flexibility)
- No runtime performance impact

## Future Enhancements

### Potential Improvements
1. **Selective Frontend ABIs**: Only copy ABIs used by frontend
2. **TypeScript Types**: Generate TypeScript types from ABIs
3. **ABI Versioning**: Track ABI changes across deployments
4. **Validation**: Verify ABI compatibility before copying

### Not Recommended
- ❌ Symlinks (platform compatibility issues)
- ❌ Runtime ABI fetching (adds latency)
- ❌ Separate backend ABI script (adds complexity)

## References

- [Foundry Book - Artifacts](https://book.getfoundry.sh/reference/forge/forge-build#artifacts)
- [Viem - Contract ABIs](https://viem.sh/docs/contract/getContract.html)
- [Node.js ES Modules](https://nodejs.org/api/esm.html)
