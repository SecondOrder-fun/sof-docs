# Root .md Files Migration Strategy

**Date:** November 10, 2025  
**Total Files:** 54 root .md files to migrate  
**Target:** Organize into docs/ repository structure

---

## Executive Summary

The root directory contains 54 .md files documenting various phases, bug fixes, implementations, and audits. These files should be reorganized into the existing docs/ structure for better discoverability and maintainability.

**Key Benefits:**
- ✅ Centralized documentation in docs/ repository
- ✅ Logical organization by category
- ✅ Easier cross-referencing and linking
- ✅ Cleaner root directory
- ✅ Better GitBook integration

---

## Detailed File Categorization

### Category 1: PHASE DOCUMENTATION (14 files)

**Purpose:** Track project phases and completion milestones  
**Destination:** `docs/07-changelog/`  
**Naming Convention:** `phase-{N}-{description}.md`

**Files:**
1. `PHASE1-IMPLEMENTATION-COMPLETE.md` → `phase-1-complete.md`
2. `PHASE1_COMPLETION_SUMMARY.md` → `phase-1-summary.md`
3. `PHASE1_ORACLE_INTEGRATION_COMPLETE.md` → `phase-1-oracle-integration.md`
4. `PHASE2-ENVIRONMENT-SETUP-COMPLETE.md` → `phase-2-environment-setup.md`
5. `PHASE2_IMPLEMENTATION_COMPLETE.md` → `phase-2-complete.md`
6. `PHASE2_QUICK_START.md` → `phase-2-quick-start.md`
7. `PHASE2_RECOVERY_MECHANISMS.md` → `phase-2-recovery-mechanisms.md`
8. `PHASE3-4-INTEGRATION-COMPLETE.md` → `phase-3-4-integration.md`
9. `PHASE3_IMPLEMENTATION_COMPLETE.md` → `phase-3-complete.md`
10. `PHASE_THREE_POLISH_COMPLETE.md` → `phase-3-polish.md`
11. `PHASE_THREE_TEST_RESULTS.md` → `phase-3-test-results.md`
12. `PHASE4_IMPLEMENTATION_COMPLETE.md` → `phase-4-complete.md`
13. `PHASE4_TESTING_COMPLETE.md` → `phase-4-testing.md`
14. `PHASE5-SEPOLIA-DEPLOYMENT.md` → `phase-5-sepolia-deployment.md`

**Action:** Create index file `07-changelog/README.md` linking all phases in chronological order

---

### Category 2: BUG FIXES & SOLUTIONS (10 files)

**Purpose:** Document bugs found and their solutions  
**Destination:** `docs/08-bug-fixes/`  
**Naming Convention:** `{component}-{issue}.md`

**Files:**
1. `ABI_NAMING_FIX.md` → `abi-naming-fix.md`
2. `ADDRESS_CASE_SENSITIVITY_FIX.md` → `address-case-sensitivity-fix.md`
3. `BUY_TOKENS_NONCE_FIX.md` → `buy-tokens-nonce-fix.md`
4. `FAUCET_ADDRESS_STALE_FIX.md` → `faucet-address-stale-fix.md`
5. `INFOFI_MARKET_CREATION_FIX.md` → `infofi-market-creation-fix.md`
6. `INFOFI_MARKET_CREATION_ROLE_FIX.md` → `infofi-market-creation-role-fix.md`
7. `MARKET_CREATION_COMPLETE_FIX.md` → `market-creation-complete-fix.md`
8. `MARKET_CREATION_GAS_FIX.md` → `market-creation-gas-fix.md`
9. `STALE_CONTRACT_ADDRESSES_FIX.md` → `stale-contract-addresses-fix.md`
10. `TRADE_LISTENER_FIX.md` → `trade-listener-fix.md`

**Action:** Create index file `08-bug-fixes/README.md` with severity/impact ratings

---

### Category 3: AUDITS & ANALYSIS (3 files)

**Purpose:** Audit reports and technical analysis  
**Destination:** `docs/04-audits/` and `docs/09-investigations/`

**Audit Files (04-audits/):**
1. `HYBRID_PRICING_INVARIANT_AUDIT.md` → `hybrid-pricing-invariant-audit.md`
2. `INFOFI_PRICE_ORACLE_AUDIT.md` → `infofi-price-oracle-audit.md`

**Analysis Files (09-investigations/):**
1. `POSITION_UPDATE_ANALYSIS.md` → `position-update-analysis.md`

---

### Category 4: ARCHITECTURE & DESIGN (5 files)

**Purpose:** System architecture and design decisions  
**Destination:** `docs/02-architecture/`  
**Naming Convention:** `{component}-{aspect}.md`

**Files:**
1. `LISTENER_DESIGN_PLAN.md` → `listener-design-plan.md`
2. `FPMM_LISTENER_ARCHITECTURE.md` → `fpmm-listener-architecture.md`
3. `MARKET_TYPE_IMPLEMENTATION.md` → `market-type-implementation.md`
4. `MARKET_TYPE_REFACTOR_PROPOSAL.md` → `market-type-refactor-proposal.md`
5. `RAFFLE_TO_SEASON_REFACTOR_SUMMARY.md` → `raffle-to-season-refactor.md`

---

### Category 5: FEATURE IMPLEMENTATIONS (15 files)

**Purpose:** Feature development documentation  
**Destination:** `docs/05-features/`  
**Naming Convention:** `{feature}-{aspect}.md`

**InfoFi Market Creation (5 files):**
1. `INFOFI_MARKET_CREATION_READY.md` → `infofi-market-creation-ready.md`
2. `INFOFI_MARKET_CREATION_TEST_GUIDE.md` → `infofi-market-creation-test-guide.md`
3. `INFOFI_MARKET_CREATION_TRACE.md` → `infofi-market-creation-trace.md`

**InfoFi Robustness (2 files):**
1. `INFOFI_ROBUSTNESS_IMPLEMENTATION.md` → `infofi-robustness-implementation.md`
2. `INFOFI_ROBUSTNESS_SUMMARY.md` → `infofi-robustness-summary.md`

**InfoFi Visibility (1 file):**
1. `INFOFI_VISIBILITY_IMPROVEMENTS.md` → `infofi-visibility-improvements.md`

**Markets Page (2 files):**
1. `MARKETS_PAGE_IMPLEMENTATION.md` → `markets-page-implementation.md`
2. `MARKETS_PAGE_TESTING_GUIDE.md` → `markets-page-testing-guide.md`

**Market Created Listener (3 files):**
1. `MARKET_CREATED_LISTENER_IMPLEMENTATION_PLAN.md` → `market-created-listener-implementation.md`
2. `MARKET_CREATED_LISTENER_TEST_RESULTS.md` → `market-created-listener-test-results.md`
3. `MARKET_CREATED_LISTENER_UPDATE_SUMMARY.md` → `market-created-listener-update-summary.md`

**Position Update Listener (4 files):**
1. `POSITION_UPDATE_LISTENER_EXECUTIVE_SUMMARY.md` → `position-update-listener-executive-summary.md`
2. `POSITION_UPDATE_LISTENER_IMPLEMENTATION.md` → `position-update-listener-implementation.md`
3. `POSITION_UPDATE_LISTENER_PLAN.md` → `position-update-listener-plan.md`
4. `POSITION_UPDATE_LISTENER_PLAN_REVISED.md` → `position-update-listener-plan-revised.md`

---

### Category 6: ORACLE INTEGRATION (4 files)

**Purpose:** Oracle integration and updates  
**Destination:** `docs/05-features/`  
**Naming Convention:** `oracle-{aspect}.md`

**Files:**
1. `INFOFI_ORACLE_INTEGRATION_PLAN.md` → `oracle-integration-plan.md`
2. `ORACLE_INTEGRATION_COMPLETE.md` → `oracle-integration-complete.md`
3. `ORACLE_INTEGRATION_FINAL_SUMMARY.md` → `oracle-integration-final-summary.md`
4. `ORACLE_UPDATE_SUMMARY.md` → `oracle-update-summary.md`

---

### Category 7: PAYMASTER INTEGRATION (3 files)

**Purpose:** Base Paymaster implementation  
**Destination:** `docs/05-features/`  
**Naming Convention:** `paymaster-{aspect}.md`

**Files:**
1. `Base-Paymaster-Implementation-Plan.md` → `paymaster-implementation-plan.md`
2. `Base-Paymaster-Integration-Work-Plan.md` → `paymaster-integration-work-plan.md`
3. `BASE_PAYMASTER_INTEGRATION_COMPLETE.md` → `paymaster-integration-complete.md`

---

### Category 8: DEPLOYMENT & SETUP (5 files)

**Purpose:** Deployment guides and configuration  
**Destination:** `docs/03-development/`  
**Naming Convention:** `{tool}-{aspect}.md` or `{feature}-setup.md`

**Files:**
1. `ANVIL_BASE_CONFIGURATION.md` → `anvil-base-configuration.md`
2. `FRAME_DEPLOYMENT_QUICK_START.md` → `frame-deployment-quick-start.md`
3. `REGISTRY_DEPLOYMENT_CHECKLIST.md` → `registry-deployment-checklist.md`
4. `SEASON_DATA_STORAGE_STRATEGY.md` → `season-data-storage-strategy.md`
5. `SEASON_STARTED_DATA_RETRIEVAL_PLAN.md` → `season-started-data-retrieval-plan.md`

---

### Category 9: GENERAL SUMMARIES (2 files)

**Purpose:** Project and implementation summaries  
**Destination:** `docs/03-development/`

**Files:**
1. `IMPLEMENTATION_COMPLETE.md` → `implementation-complete.md`
2. `INFOFI_QUICK_START.md` → `infofi-quick-start.md`

---

## Migration Process

### Phase 1: Preparation
- ✅ Create migration index (ROOT_FILES_MIGRATION_INDEX.md)
- ✅ Create migration strategy (this file)
- ⏳ Verify all destination directories exist
- ⏳ Create README files for each category

### Phase 2: File Migration
- ⏳ Copy files to destination directories
- ⏳ Normalize filenames to lowercase with hyphens
- ⏳ Update internal cross-references
- ⏳ Verify all files copied successfully

### Phase 3: Integration
- ⏳ Create category index files (README.md in each subdirectory)
- ⏳ Update main docs/README.md with new structure
- ⏳ Create cross-reference index
- ⏳ Test all links

### Phase 4: Cleanup
- ⏳ Delete root .md files
- ⏳ Update .gitignore to prevent future root .md files
- ⏳ Create migration completion summary

---

## Cross-Reference Strategy

### Linking Between Files

**Old Style (root directory):**
```markdown
See [PHASE1_ORACLE_INTEGRATION_COMPLETE.md](../PHASE1_ORACLE_INTEGRATION_COMPLETE.md)
```

**New Style (docs structure):**
```markdown
See [Phase 1 Oracle Integration](../07-changelog/phase-1-oracle-integration.md)
```

### Creating Index Files

Each category will have a README.md that:
1. Lists all files in the category
2. Provides brief descriptions
3. Links to related categories
4. Includes a table of contents

---

## Consolidation Opportunities

**Files that might be consolidated:**
- Multiple PHASE completion files → Single phase timeline
- Multiple listener implementation files → Single listener documentation
- Multiple oracle update files → Single oracle documentation

**Decision:** Keep files separate initially, consolidate only if clearly redundant

---

## Success Criteria

✅ All 54 files migrated to docs/  
✅ All filenames normalized to lowercase with hyphens  
✅ All internal cross-references updated  
✅ All category index files created  
✅ Main docs/README.md updated  
✅ Root directory cleaned of .md files  
✅ All links verified and working  
✅ .gitignore updated to prevent root .md files  

---

## Timeline

- **Phase 1 (Prep):** 15 minutes
- **Phase 2 (Migration):** 30 minutes
- **Phase 3 (Integration):** 45 minutes
- **Phase 4 (Cleanup):** 15 minutes
- **Total:** ~2 hours

---

## Notes

- This is a non-destructive migration (files copied, not moved)
- Root files will be deleted only after verification
- All changes will be committed to git with clear commit messages
- Migration can be rolled back if needed

