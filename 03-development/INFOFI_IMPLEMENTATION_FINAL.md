# InfoFi Batch Claims + Security & Gas Optimization - FINAL PLAN

**Date:** October 16, 2025  
**Status:** Ready for Implementation  
**Estimated Time:** 1-2 weeks

---

## **Key Updates from Original Plan**

✅ **Use Canonical ABIs** - All functions now import from `src/contracts/abis/` instead of inline minimal ABIs  
✅ **Season Completion Validation** - Claims only allowed when `SeasonStatus.Completed === 5`  
✅ **Deferred to Post-MVP** - ERC-1155 CTF and UMA oracle (our resolution depends on Raffle VRF)

---

## **Phase 1: Smart Contract Enhancements**

### **Changes to InfoFiMarket.sol**

1. **Add `calculatePayout()` view function** (after line 264)
2. **Add safety checks to both `claimPayout()` functions** (lines 222-264):
   - Zero-division protection
   - Solvency check
   - Sanity check (payout ≤ totalPool)
3. **Add pool invariant checks to `placeBet()`** (after line 152):
   - Assert `totalYesPool + totalNoPool == totalPool`
   - Overflow protection
4. **Add `ClaimRequest` struct** (after line 65)
5. **Add `batchClaimPayouts()` function** (after `calculatePayout`)
6. **Add `batchCalculatePayouts()` view** (after `batchClaimPayouts`)

### **Changes to InfoFiMarketFactory.sol**

1. **Add `MAX_PLAYERS_PER_UPDATE = 50` constant**
2. **Add `PartialOracleUpdate` event**
3. **Optimize `_updateAllPlayerProbabilities()`** with:
   - Gas limit (50 players max)
   - Unchecked loop optimization
   - Monitoring event emission

---

## **Phase 2: Frontend Service Updates**

### **File:** `src/services/onchainInfoFi.js`

**CRITICAL CHANGE:** Use canonical ABIs instead of inline minimal ABIs

**Add imports at top of file:**
```javascript
import InfoFiMarketABI from '@/contracts/abis/InfoFiMarket.json';
import RaffleABI from '@/contracts/abis/Raffle.json';
```

**Add 4 new functions (after line 62):**

1. **`calculatePayoutView()`** - Preview single payout
   - Uses `InfoFiMarketABI`
   - Calls `calculatePayout()` view function

2. **`batchCalculatePayoutsView()`** - Preview batch payouts
   - Uses `InfoFiMarketABI`
   - Calls `batchCalculatePayouts()` view function

3. **`batchClaimPayoutsTx()`** - Execute batch claim
   - Uses `InfoFiMarketABI`
   - Calls `batchClaimPayouts()` transaction

4. **`isSeasonCompleted()`** - Check season status
   - Uses `RaffleABI`
   - Calls `getSeasonDetails()` and checks `status === 5`

**Key Points:**
- ✅ All functions use canonical ABIs from `src/contracts/abis/`
- ✅ No inline ABI definitions
- ✅ Consistent with existing codebase patterns
- ✅ Prevents ABI update issues

---

## **Phase 3: Frontend Component Updates**

### **Step 3.1: Remove Claim Buttons from InfoFiMarketCard.jsx**

**DELETE:**
- Lines 245-264: Claim mutations (`claimYes`, `claimNo`)
- Lines 575-605: Claim UI section
- Line 12: Remove `claimPayoutTx` from imports

### **Step 3.2: Replace ClaimCenter.jsx**

**New features:**
- Uses `calculatePayoutView()` for payout preview
- Uses `isSeasonCompleted()` for season validation
- Uses `batchClaimPayoutsTx()` for batch claiming
- Separates "Ready to Claim" (completed seasons) from "Pending" (incomplete seasons)
- Shows gas savings estimate
- Groups claims by season

---

## **Implementation Checklist**

### **Smart Contracts** ✅

- [ ] Add `calculatePayout()` view function
- [ ] Add safety checks to both `claimPayout()` functions
- [ ] Add pool invariant checks to `placeBet()`
- [ ] Add `ClaimRequest` struct
- [ ] Add `batchClaimPayouts()` function
- [ ] Add `batchCalculatePayouts()` view function
- [ ] Add `MAX_PLAYERS_PER_UPDATE` constant to factory
- [ ] Add `PartialOracleUpdate` event to factory
- [ ] Optimize `_updateAllPlayerProbabilities()` with gas limit
- [ ] Write 7 unit tests
- [ ] Verify gas savings (target: 60-70%)

### **Frontend Services** ✅

- [ ] **Add canonical ABI imports** to `onchainInfoFi.js`
- [ ] Add `calculatePayoutView()` using `InfoFiMarketABI`
- [ ] Add `batchCalculatePayoutsView()` using `InfoFiMarketABI`
- [ ] Add `batchClaimPayoutsTx()` using `InfoFiMarketABI`
- [ ] Add `isSeasonCompleted()` using `RaffleABI`
- [ ] **Verify no inline ABIs remain**

### **Frontend Components** ✅

- [ ] Remove claim mutations from `InfoFiMarketCard.jsx`
- [ ] Remove claim UI from `InfoFiMarketCard.jsx`
- [ ] Remove `claimPayoutTx` import from `InfoFiMarketCard.jsx`
- [ ] Replace entire `ClaimCenter.jsx` component
- [ ] Add season completion validation
- [ ] Add batch claim with gas savings display
- [ ] Add "Ready to Claim" vs "Pending" sections

### **Testing** ✅

- [ ] Create `InfoFiMarketSafety.t.sol` with 7 test cases
- [ ] Create `infofi-batch-claim-safety.test.jsx` with 6 scenarios
- [ ] Manual E2E testing on Anvil
- [ ] Verify canonical ABIs work correctly

### **Documentation** ✅

- [ ] Update API docs with new functions
- [ ] Document canonical ABI usage pattern
- [ ] Document safety guarantees
- [ ] Document gas savings analysis

---

## **Canonical ABI Usage Pattern**

### **Before (Problematic):**
```javascript
// ❌ Inline minimal ABI - causes update issues
const abi = [{
  type: 'function',
  name: 'calculatePayout',
  // ... inline definition
}];

const result = await publicClient.readContract({
  address: market.address,
  abi, // Inline ABI
  functionName: 'calculatePayout',
  args: [...]
});
```

### **After (Correct):**
```javascript
// ✅ Import canonical ABI
import InfoFiMarketABI from '@/contracts/abis/InfoFiMarket.json';

const result = await publicClient.readContract({
  address: market.address,
  abi: InfoFiMarketABI, // Canonical ABI
  functionName: 'calculatePayout',
  args: [...]
});
```

### **Benefits:**
- ✅ Single source of truth for ABIs
- ✅ Automatic updates when contracts change
- ✅ Consistent with existing codebase
- ✅ Prevents version mismatch issues
- ✅ Easier to maintain and debug

---

## **Available Canonical ABIs**

Located in `src/contracts/abis/`:

- `InfoFiMarket.json` - Main prediction market contract
- `InfoFiMarketFactory.json` - Market creation factory
- `InfoFiPriceOracle.json` - Hybrid pricing oracle
- `Raffle.json` - Core raffle contract
- `RafflePrizeDistributor.json` - Prize distribution
- `SOFBondingCurve.json` - Bonding curve for tickets
- `SOFToken.json` - SOF ERC-20 token
- `ERC20.json` - Standard ERC-20 interface

**Usage in this implementation:**
- `InfoFiMarketABI` - For all InfoFi market interactions
- `RaffleABI` - For season status checks

---

## **Timeline**

### **Week 1: Smart Contract Implementation**
- **Day 1:** Add safety checks to existing functions
- **Day 2:** Implement batch claim functions
- **Day 3:** Optimize factory oracle updates
- **Day 4:** Write unit tests
- **Day 5:** Run tests, verify gas savings

### **Week 2: Frontend Integration & Testing**
- **Day 1:** Update `onchainInfoFi.js` with canonical ABIs
- **Day 2:** Update UI components
- **Day 3:** Write integration tests
- **Day 4:** Manual E2E testing on Anvil
- **Day 5:** Documentation and deployment prep

---

## **Success Metrics**

### **Safety** ✅
- Zero division errors: **0 occurrences**
- Pool invariant violations: **0 occurrences**
- Insolvency attempts: **100% blocked**
- All edge cases covered

### **Gas Efficiency** ✅
- Batch claim savings: **60-70% vs individual**
- Oracle update gas limit: **< 5M gas per update**
- Loop optimizations: **10-15% savings**

### **Code Quality** ✅
- Canonical ABI usage: **100%**
- No inline ABIs: **0 occurrences**
- ABI update issues: **0 occurrences**

---

## **Risk Mitigation**

1. **ABI version mismatch** → Use canonical ABIs from `copy-abis.js` script
2. **Gas limit exceeded** → 50-claim limit, UI estimation
3. **Solvency check false positives** → Comprehensive testing
4. **Oracle update gas spikes** → MAX_PLAYERS_PER_UPDATE cap
5. **User confusion** → Clear UI labels, gas savings display

---

## **Post-Implementation Verification**

### **Checklist:**
- [ ] All functions use canonical ABIs
- [ ] No inline ABI definitions in codebase
- [ ] `copy-abis.js` script runs successfully
- [ ] ABIs match deployed contract versions
- [ ] All tests pass with canonical ABIs
- [ ] Gas savings verified (60-70%)
- [ ] Season completion validation works
- [ ] Batch claims work end-to-end

---

## **Deferred to Post-MVP**

These features are acknowledged as important but not required for alpha/MVP:

1. **ERC-1155 Conditional Token Framework** - Enables DeFi composability
2. **Automated UMA Oracle Resolution** - Our resolution depends on Raffle VRF
3. **Off-chain Order Matching** - Gas optimization for high-volume trading
4. **IPFS Metadata Storage** - Gas savings for large data

---

**Total Estimated Changes:**
- **Smart Contracts:** ~400 lines
- **Frontend Services:** ~200 lines (with canonical ABIs)
- **Frontend Components:** ~300 lines
- **Tests:** ~500 lines
- **Total:** ~1,400 lines

---

**Ready to implement! 🚀**
