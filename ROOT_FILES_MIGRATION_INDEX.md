# Root .md Files Migration Index

**Date:** November 10, 2025  
**Status:** Migration Plan - 54 files to organize

This document maps all root-level .md files to their new locations in the docs/ repository structure.

---

## Migration Categories

### 1. PHASE DOCUMENTATION (9 files) → `07-changelog/`

**Phase Summaries & Completion Reports:**
- `PHASE1-IMPLEMENTATION-COMPLETE.md` → `07-changelog/phase-1-complete.md`
- `PHASE1_COMPLETION_SUMMARY.md` → `07-changelog/phase-1-summary.md`
- `PHASE1_ORACLE_INTEGRATION_COMPLETE.md` → `07-changelog/phase-1-oracle-integration.md`
- `PHASE2-ENVIRONMENT-SETUP-COMPLETE.md` → `07-changelog/phase-2-environment-setup.md`
- `PHASE2_IMPLEMENTATION_COMPLETE.md` → `07-changelog/phase-2-complete.md`
- `PHASE2_QUICK_START.md` → `07-changelog/phase-2-quick-start.md`
- `PHASE2_RECOVERY_MECHANISMS.md` → `07-changelog/phase-2-recovery-mechanisms.md`
- `PHASE3-4-INTEGRATION-COMPLETE.md` → `07-changelog/phase-3-4-integration.md`
- `PHASE3_IMPLEMENTATION_COMPLETE.md` → `07-changelog/phase-3-complete.md`
- `PHASE4_IMPLEMENTATION_COMPLETE.md` → `07-changelog/phase-4-complete.md`
- `PHASE4_TESTING_COMPLETE.md` → `07-changelog/phase-4-testing.md`
- `PHASE5-SEPOLIA-DEPLOYMENT.md` → `07-changelog/phase-5-sepolia-deployment.md`
- `PHASE_THREE_POLISH_COMPLETE.md` → `07-changelog/phase-3-polish.md`
- `PHASE_THREE_TEST_RESULTS.md` → `07-changelog/phase-3-test-results.md`

### 2. BUG FIXES & SOLUTIONS (11 files) → `08-bug-fixes/`

**All *_FIX.md files:**
- `ABI_NAMING_FIX.md` → `08-bug-fixes/abi-naming-fix.md`
- `ADDRESS_CASE_SENSITIVITY_FIX.md` → `08-bug-fixes/address-case-sensitivity-fix.md`
- `BUY_TOKENS_NONCE_FIX.md` → `08-bug-fixes/buy-tokens-nonce-fix.md`
- `FAUCET_ADDRESS_STALE_FIX.md` → `08-bug-fixes/faucet-address-stale-fix.md`
- `INFOFI_MARKET_CREATION_FIX.md` → `08-bug-fixes/infofi-market-creation-fix.md`
- `INFOFI_MARKET_CREATION_ROLE_FIX.md` → `08-bug-fixes/infofi-market-creation-role-fix.md`
- `MARKET_CREATION_COMPLETE_FIX.md` → `08-bug-fixes/market-creation-complete-fix.md`
- `MARKET_CREATION_GAS_FIX.md` → `08-bug-fixes/market-creation-gas-fix.md`
- `STALE_CONTRACT_ADDRESSES_FIX.md` → `08-bug-fixes/stale-contract-addresses-fix.md`
- `TRADE_LISTENER_FIX.md` → `08-bug-fixes/trade-listener-fix.md`

### 3. AUDITS & ANALYSIS (4 files) → `04-audits/`

**Audit Reports:**
- `HYBRID_PRICING_INVARIANT_AUDIT.md` → `04-audits/hybrid-pricing-invariant-audit.md`
- `INFOFI_PRICE_ORACLE_AUDIT.md` → `04-audits/infofi-price-oracle-audit.md`
- `POSITION_UPDATE_ANALYSIS.md` → `09-investigations/position-update-analysis.md`

### 4. ARCHITECTURE & DESIGN (6 files) → `02-architecture/`

**System Architecture & Design Plans:**
- `LISTENER_DESIGN_PLAN.md` → `02-architecture/listener-design-plan.md`
- `FPMM_LISTENER_ARCHITECTURE.md` → `02-architecture/fpmm-listener-architecture.md`
- `MARKET_TYPE_IMPLEMENTATION.md` → `02-architecture/market-type-implementation.md`
- `MARKET_TYPE_REFACTOR_PROPOSAL.md` → `02-architecture/market-type-refactor-proposal.md`
- `RAFFLE_TO_SEASON_REFACTOR_SUMMARY.md` → `02-architecture/raffle-to-season-refactor.md`

### 5. FEATURE IMPLEMENTATIONS (12 files) → `05-features/`

**Feature Development & Implementation:**
- `INFOFI_MARKET_CREATION_READY.md` → `05-features/infofi-market-creation-ready.md`
- `INFOFI_MARKET_CREATION_TEST_GUIDE.md` → `05-features/infofi-market-creation-test-guide.md`
- `INFOFI_MARKET_CREATION_TRACE.md` → `05-features/infofi-market-creation-trace.md`
- `INFOFI_ROBUSTNESS_IMPLEMENTATION.md` → `05-features/infofi-robustness-implementation.md`
- `INFOFI_ROBUSTNESS_SUMMARY.md` → `05-features/infofi-robustness-summary.md`
- `INFOFI_VISIBILITY_IMPROVEMENTS.md` → `05-features/infofi-visibility-improvements.md`
- `MARKETS_PAGE_IMPLEMENTATION.md` → `05-features/markets-page-implementation.md`
- `MARKETS_PAGE_TESTING_GUIDE.md` → `05-features/markets-page-testing-guide.md`
- `MARKET_CREATED_LISTENER_IMPLEMENTATION_PLAN.md` → `05-features/market-created-listener-implementation.md`
- `MARKET_CREATED_LISTENER_TEST_RESULTS.md` → `05-features/market-created-listener-test-results.md`
- `MARKET_CREATED_LISTENER_UPDATE_SUMMARY.md` → `05-features/market-created-listener-update-summary.md`
- `POSITION_UPDATE_LISTENER_EXECUTIVE_SUMMARY.md` → `05-features/position-update-listener-executive-summary.md`
- `POSITION_UPDATE_LISTENER_IMPLEMENTATION.md` → `05-features/position-update-listener-implementation.md`
- `POSITION_UPDATE_LISTENER_PLAN.md` → `05-features/position-update-listener-plan.md`
- `POSITION_UPDATE_LISTENER_PLAN_REVISED.md` → `05-features/position-update-listener-plan-revised.md`

### 6. ORACLE & INTEGRATION (4 files) → `05-features/`

**Oracle Integration:**
- `INFOFI_ORACLE_INTEGRATION_PLAN.md` → `05-features/infofi-oracle-integration-plan.md`
- `ORACLE_INTEGRATION_COMPLETE.md` → `05-features/oracle-integration-complete.md`
- `ORACLE_INTEGRATION_FINAL_SUMMARY.md` → `05-features/oracle-integration-final-summary.md`
- `ORACLE_UPDATE_SUMMARY.md` → `05-features/oracle-update-summary.md`

### 7. DEPLOYMENT & SETUP (5 files) → `03-development/`

**Deployment Guides & Configuration:**
- `ANVIL_BASE_CONFIGURATION.md` → `03-development/anvil-base-configuration.md`
- `FRAME_DEPLOYMENT_QUICK_START.md` → `03-development/frame-deployment-quick-start.md`
- `REGISTRY_DEPLOYMENT_CHECKLIST.md` → `03-development/registry-deployment-checklist.md`
- `SEASON_DATA_STORAGE_STRATEGY.md` → `03-development/season-data-storage-strategy.md`
- `SEASON_STARTED_DATA_RETRIEVAL_PLAN.md` → `03-development/season-started-data-retrieval-plan.md`

### 8. PAYMASTER INTEGRATION (2 files) → `05-features/`

**Base Paymaster Implementation:**
- `Base-Paymaster-Implementation-Plan.md` → `05-features/base-paymaster-implementation-plan.md`
- `Base-Paymaster-Integration-Work-Plan.md` → `05-features/base-paymaster-integration-work-plan.md`
- `BASE_PAYMASTER_INTEGRATION_COMPLETE.md` → `05-features/base-paymaster-integration-complete.md`

### 9. GENERAL SUMMARIES (3 files) → `03-development/`

**Implementation & Project Summaries:**
- `IMPLEMENTATION_COMPLETE.md` → `03-development/implementation-complete.md`
- `INFOFI_QUICK_START.md` → `03-development/infofi-quick-start.md`

---

## File Count Summary

| Category | Count | Destination |
|----------|-------|-------------|
| Phase Documentation | 14 | `07-changelog/` |
| Bug Fixes | 10 | `08-bug-fixes/` |
| Audits & Analysis | 3 | `04-audits/` & `09-investigations/` |
| Architecture & Design | 5 | `02-architecture/` |
| Feature Implementations | 15 | `05-features/` |
| Oracle Integration | 4 | `05-features/` |
| Deployment & Setup | 5 | `03-development/` |
| Paymaster Integration | 3 | `05-features/` |
| General Summaries | 2 | `03-development/` |
| **TOTAL** | **54** | **docs/** |

---

## Migration Steps

1. ✅ Create this index document
2. ⏳ Create subdirectory structure if needed
3. ⏳ Copy files to new locations with normalized names (lowercase, hyphens)
4. ⏳ Create cross-reference index in docs/README.md
5. ⏳ Delete root .md files after verification
6. ⏳ Update .gitignore to prevent root .md files

---

## Notes

- All filenames will be normalized to lowercase with hyphens (e.g., `PHASE1-IMPLEMENTATION-COMPLETE.md` → `phase-1-complete.md`)
- Existing docs/ files with similar content will be reviewed for consolidation
- Cross-references between files will be updated to reflect new paths
- This index will be kept as a reference for the migration history

