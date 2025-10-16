# InfoFi Final Specifications - APPROVED

**Date:** October 16, 2025  
**Status:** Ready for Implementation

---

## Confirmed Specifications

1. **Invariant:** YES + NO = 10 SOF (complete set value)
2. **Fees:** 2% on winnings only (not on trades)
3. **Token Standard:** Basic position tracking (ERC-1155 deferred)
4. **Resolution:** Automatic from Raffle VRF callback
5. **Deployment:** Base L2

---

## Architecture: Pseudo-Categorical CSMM

**One SeasonCSMM per season** managing multiple player markets:
- **CSMM formula:** x + y = 10 SOF (constant sum, not constant product!)
- **Price:** P(YES) = y / 10 (linear pricing)
- **Complete set:** 1 YES + 1 NO = 10 SOF (redeemable)
- **Shared liquidity pool:** 10 SOF per player market
- **Cross-market effects** through shared pool

---

## Key Implementation Points

### 1. Liquidity Allocation (Scalable)

```
Base: 50 players
- Total: 500 SOF (50 × 10)
- Per player: 10 SOF
- Initial split: 5 YES + 5 NO (50% price)
- Invariant: YES + NO = 10 SOF

10x Scale: 500 players
- Total: 5,000 SOF
- Linear scaling

100x Scale: 5,000 players
- Total: 50,000 SOF
- Linear scaling

1000x Scale: 50,000 players
- Total: 500,000 SOF
- Linear scaling
```

### 2. Fee Structure (2% on Winnings)

```solidity
uint256 grossPayout = userShares; // 1 share = 1 SOF
uint256 fee = (grossPayout * 200) / 10000; // 2%
uint256 netPayout = grossPayout - fee;

// Fee to treasury, net to user
```

### 3. Resolution Flow

```
Raffle VRF → Winner Selected → Raffle._distributeWinnings()
  ↓
InfoFiFactory.resolveSeasonMarkets(seasonId, winner)
  ↓
For each player:
  - SeasonFPMM.resolveMarket(playerId)
  - InfoFiMarket.resolveMarket(outcome = player == winner)
  ↓
Users claim payouts with 2% fee
```

### 4. Base L2 Deployment

- Gas per trade: ~$0.01-0.05
- Market creation: ~$0.10-0.20
- Resolution: ~$0.05-0.10

---

## Implementation Timeline

**Phase 1 (Weeks 1-3):** Core FPMM + Resolution  
**Phase 2 (Weeks 4-5):** Gas Optimization  
**Phase 3 (Weeks 6-9):** ERC-1155 (deferred)

**Total: 5 weeks to production MVP**

---

## Next Steps

1. Implement SeasonFPMM.sol
2. Integrate with InfoFiMarketFactory
3. Add resolution callback to Raffle
4. Write comprehensive tests
5. Deploy to Base testnet
6. Frontend integration

---

**APPROVED FOR IMPLEMENTATION** ✅
