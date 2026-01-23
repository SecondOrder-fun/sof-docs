# Table of Contents

## Getting Started

- [Welcome](README.md)

## Product Documentation

- [Product Overview](product/README.md)
  - [Product Framework](product/framework.md)
  - [Tokenomics](product/tokenomics.md)
  - [Bonding Curve & Prize Pool](product/bonding-curve.md)
  - [Liquidity & Rewards](product/liquidity-rewards.md)
  - [Winner Selection](product/winner-selection.md)

## Architecture

- [Architecture Overview](architecture/README.md)
  - [Project Requirements](architecture/requirements.md)
  - [Project Structure](architecture/structure.md)
  - [Data Schema](architecture/data-schema.md)
  - [InfoFi Integration](architecture/infofi-integration/README.md)
    - [Part 1: Overview](architecture/infofi-integration/part-1-overview.md)
    - [Part 2: Smart Contracts](architecture/infofi-integration/part-2-contracts.md)
    - [Part 3: Backend Services](architecture/infofi-integration/part-3-backend.md)
    - [Part 4: Frontend Integration](architecture/infofi-integration/part-4-frontend.md)
    - [Part 5: Database Schema](architecture/infofi-integration/part-5-database.md)
    - [Part 6: Real-time Architecture](architecture/infofi-integration/part-6-realtime.md)
    - [Part 7: Arbitrage Detection](architecture/infofi-integration/part-7-arbitrage.md)
    - [Part 8: Security Considerations](architecture/infofi-integration/part-8-security.md)
    - [Part 9: Deployment](architecture/infofi-integration/part-9-deployment.md)
  - [InfoFi: Critical Bug (Factory Not Set)](architecture/infofi/critical-bug-factory-not-set.md)
  - [InfoFi: Position Updates](architecture/infofi/position-updates.md)

## Development

- [Development Overview](development/README.md)
  - [Frontend Guidelines](development/frontend-guidelines.md)
  - [Project Tasks](development/tasks.md)
  - [Prediction Market Dev Plan](development/prediction-market-plan.md)
  - [Quick Reference](development/quick-reference.md)

## API Reference

- [API Overview](api/README.md)
  - [Analytics API](api/analytics.md)
  - [Arbitrage API](api/arbitrage.md)
  - [InfoFi API](api/infofi.md)
  - [Pricing API](api/pricing.md)
  - [Settlement API](api/settlement.md)

## Reference

- [Project](reference/project/00-project/REPOSITORY_STRUCTURE.md)
- [Planning](reference/planning/sof-prediction-market-dev-plan.md)
- [Features Overview](reference/features/README.md)
  - [Internationalization (i18n)](reference/features/i18n/README.md)
    - [Integration Plan](reference/features/i18n/integration-plan.md)
    - [Localization System](reference/features/i18n/localization-system.md)
    - [Sample Translations](reference/features/i18n/sample-translations.md)
  - [Arbitrage Detection](reference/features/arbitrage/README.md)
    - [Detection System](reference/features/arbitrage/detection.md)
    - [Flow Diagram](reference/features/arbitrage/flow-diagram.md)
  - [Treasury System](reference/features/treasury/README.md)
    - [System Overview](reference/features/treasury/system.md)
  - [Consolation System](reference/features/consolation/README.md)
    - [Implementation](reference/features/consolation/implementation.md)

## Technical Analysis

- [Technical Analysis Overview](reference/technical-analysis/README.md)
  - [Fee Collection Gas Analysis](reference/technical-analysis/fee-collection-gas.md)

## History

- [History Overview](history/README.md)
  - [October 2025](history/2025-10/README.md)
    - [Oct 3: Implementation Summary](history/2025-10/03-implementation-summary.md)
  - [Feature Implementations](history/features/README.md)
    - [Account Routing Fix](history/features/account-routing-fix.md)
    - [Arbitrage Implementation](history/features/arbitrage-implementation.md)
    - [Consolation System](history/features/consolation-system.md)
    - [i18n Implementation](history/features/i18n-implementation.md)
    - [Name Validation](history/features/name-validation.md)
    - [Prize Sponsorship](history/features/prize-sponsorship.md)
    - [Transactions & Holders](history/features/transactions-holders.md)
    - [Treasury Implementation](history/features/treasury-implementation.md)

## Incidents

- [Bug Fixes Overview](incidents/bug-fixes/README.md)
  - [Display Issues](incidents/bug-fixes/display-issues/README.md)
    - [Consolation Display](incidents/bug-fixes/display-issues/consolation-display.md)
    - [Odds Display](incidents/bug-fixes/display-issues/odds-display.md)
    - [Position Display](incidents/bug-fixes/display-issues/position-display.md)
    - [Prize Pool Display](incidents/bug-fixes/display-issues/prize-pool-display.md)
    - [Tabs Display](incidents/bug-fixes/display-issues/tabs-display.md)
    - [Token Info Display](incidents/bug-fixes/display-issues/token-info-display.md)
  - [Trading Issues](incidents/bug-fixes/trading-issues/README.md)
    - [Max Sell Revert](incidents/bug-fixes/trading-issues/max-sell-revert.md)
    - [Sell All Tokens](incidents/bug-fixes/trading-issues/sell-all-tokens.md)
    - [Sell Max Fix](incidents/bug-fixes/trading-issues/sell-max-fix.md)
    - [Trading Lock Guard](incidents/bug-fixes/trading-issues/trading-lock-guard.md)
  - [Test Fixes](incidents/bug-fixes/test-fixes/README.md)
    - [E2E Test Results](incidents/bug-fixes/test-fixes/e2e-test-results.md)
    - [Failing Tests Analysis](incidents/bug-fixes/test-fixes/failing-tests-analysis.md)
    - [Frontend Timing Fixes](incidents/bug-fixes/test-fixes/frontend-timing-fixes.md)
    - [Raffle Holders Test](incidents/bug-fixes/test-fixes/raffle-holders-test.md)
    - [Test Fixes Final](incidents/bug-fixes/test-fixes/test-fixes-final.md)
    - [Test Fixes Progress](incidents/bug-fixes/test-fixes/test-fixes-progress.md)
    - [Test Fixes Session](incidents/bug-fixes/test-fixes/test-fixes-session.md)
    - [Test Suite Complete](incidents/bug-fixes/test-fixes/test-suite-complete.md)
  - [Other Fixes](incidents/bug-fixes/other/README.md)
    - [InfoFi Odds Bug](incidents/bug-fixes/other/infofi-odds-bug.md)
    - [Raffle Details Update](incidents/bug-fixes/other/raffle-details-update.md)
    - [Raffle Tabs Block Range](incidents/bug-fixes/other/raffle-tabs-block-range.md)
    - [Raffle Tabs Fix](incidents/bug-fixes/other/raffle-tabs-fix.md)
    - [Tabs Empty Data](incidents/bug-fixes/other/tabs-empty-data.md)
    - [Transaction Confirmation](incidents/bug-fixes/other/transaction-confirmation.md)

## Investigations

- [Investigations Overview](investigations/research/README.md)
  - [Position Display Investigation](investigations/research/position-display-investigation.md)
  - [Treasury Admin Debug](investigations/research/treasury-admin-debug.md)
