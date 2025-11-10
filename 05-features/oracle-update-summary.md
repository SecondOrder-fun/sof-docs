# InfoFiPriceOracle Update Summary

**Date**: Oct 26, 2025  
**Status**: ✅ COMPLETED - Oracle contract updated to use FPMM address as market ID

---

## What Changed

### Smart Contract: InfoFiPriceOracle.sol

**File**: `contracts/src/infofi/InfoFiPriceOracle.sol`

#### Key Changes

1. **Market ID Type Changed**
   - **Before**: `uint256 marketId` (abstract identifier)
   - **After**: `address fpmmAddress` (SimpleFPMM contract address)

2. **Storage Mapping Updated**
   ```solidity
   // Before
   mapping(uint256 => PriceData) public prices;
   
   // After
   mapping(address => PriceData) public prices;
   ```

3. **Function Signatures Updated**
   ```solidity
   // Before
   function updateRaffleProbability(uint256 marketId, uint256 raffleProbabilityBps)
   function updateMarketSentiment(uint256 marketId, uint256 marketSentimentBps)
   function getPrice(uint256 marketId)
   
   // After
   function updateRaffleProbability(address fpmmAddress, uint256 raffleProbabilityBps)
   function updateMarketSentiment(address fpmmAddress, uint256 marketSentimentBps)
   function getPrice(address fpmmAddress)
   ```

4. **Event Signature Updated**
   ```solidity
   // Before
   event PriceUpdated(uint256 indexed marketId, ...)
   
   // After
   event PriceUpdated(address indexed fpmmAddress, ...)
   ```

5. **Input Validation Added**
   ```solidity
   require(fpmmAddress != address(0), "Oracle: invalid FPMM address");
   ```

6. **Documentation Enhanced**
   - Updated NatSpec comments to reflect FPMM address usage
   - Added detailed comments explaining the hybrid price calculation

---

## Why This Change

| Aspect | Before | After |
|--------|--------|-------|
| **Market ID** | Abstract `uint256` | Concrete contract address |
| **Uniqueness** | Requires manual tracking | Automatic (contract address) |
| **Verification** | Indirect lookup | Direct on-chain reference |
| **Complexity** | Needs mapping layer | Direct key-value storage |
| **Clarity** | Abstract identifier | Clear contract reference |

---

## Integration Impact

### Backend Changes Required

**positionUpdateListener.js**:
```javascript
// Get fpmmAddress from database
const fpmmAddress = await db.getFpmmAddress(seasonId, player);

// Call oracle with fpmmAddress
await oracleCallService.updateRaffleProbability(
  fpmmAddress,  // Use FPMM address as market ID
  raffleProbabilityBps,
  logger
);
```

**tradeListener.js** (NEW):
```javascript
// Listen to Trade events from SimpleFPMM
publicClient.watchContractEvent({
  address: fpmmAddress,  // Each FPMM contract
  abi: fpmmAbi,
  eventName: 'Trade',
  onLogs: async (logs) => {
    await oracleCallService.updateMarketSentiment(
      fpmmAddress,  // Use FPMM address as market ID
      sentiment,
      logger
    );
  }
});
```

### Database Changes Required

**oracle_call_history table**:
```sql
- fpmm_address (VARCHAR, FK to infofi_markets.fpmm_address)
- call_type (VARCHAR)
- parameters (JSONB with fpmmAddress)
- status, attempt_count, transaction_hash, error_message
- created_at, updated_at
```

---

## Backward Compatibility

⚠️ **BREAKING CHANGE**: This is a breaking change for any code that references the oracle.

**Must Update**:
- ✅ All oracle function calls (use `address` instead of `uint256`)
- ✅ All oracle event listeners (expect `address` indexed parameter)
- ✅ All oracle data storage (use `address` as key)
- ✅ All oracle ABI references (regenerate from updated contract)

**No Changes Needed**:
- ✅ Hybrid price calculation formula (unchanged)
- ✅ Weight management (unchanged)
- ✅ Role-based access control (unchanged)

---

## Deployment Checklist

- [ ] Rebuild contracts: `cd contracts && forge build`
- [ ] Verify contract compiles without errors
- [ ] Update backend oracleCallService.js to use `address` parameters
- [ ] Update positionUpdateListener.js to extract and pass `fpmmAddress`
- [ ] Create tradeListener.js to listen to SimpleFPMM Trade events
- [ ] Update marketCreatedListener.js to store `fpmmAddress` in database
- [ ] Create database migration for oracle_call_history table
- [ ] Update all oracle ABI references
- [ ] Run integration tests
- [ ] Deploy to testnet
- [ ] Verify oracle calls work with new address-based market IDs

---

## Testing Strategy

### Unit Tests
- Test `updateRaffleProbability` with valid/invalid addresses
- Test `updateMarketSentiment` with valid/invalid addresses
- Test `getPrice` returns correct data for address key
- Test address validation (zero address rejection)

### Integration Tests
- Test backend calls oracle with correct fpmmAddress
- Test oracle stores data keyed by fpmmAddress
- Test PriceUpdated event emits with fpmmAddress
- Test hybrid price calculation with new data structure

### End-to-End Tests
- Test full flow: MarketCreated → extract fpmmAddress → call oracle → verify storage
- Test Trade event → call oracle → verify sentiment update
- Test PositionUpdate event → call oracle → verify probability update

---

## Related Files

**Smart Contracts**:
- `contracts/src/infofi/InfoFiPriceOracle.sol` ✅ UPDATED

**Backend** (To be updated):
- `backend/src/services/oracleCallService.js` (NEW)
- `backend/src/listeners/positionUpdateListener.js` (MODIFY)
- `backend/src/listeners/tradeListener.js` (NEW)
- `backend/src/listeners/marketCreatedListener.js` (MODIFY)

**Database**:
- `migrations/add_oracle_call_history_table.sql` (NEW)

**Documentation**:
- `INFOFI_ORACLE_INTEGRATION_PLAN.md` (UPDATED)

---

## Next Steps

1. ✅ Smart contract updated
2. ⏳ Rebuild and verify compilation
3. ⏳ Update backend services
4. ⏳ Create database migration
5. ⏳ Run tests
6. ⏳ Deploy to testnet
7. ⏳ Verify integration

