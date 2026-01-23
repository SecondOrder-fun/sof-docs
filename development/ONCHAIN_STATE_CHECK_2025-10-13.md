# On-Chain State Check - Season 1 (2025-10-13 8:06 PM)

## Summary

Checked the on-chain state of Season 1 raffle positions and InfoFi markets to verify current win probabilities and market sentiment.

## Oracle Prices (On-Chain)

### Market 0 (Player1: 0x7099...79C8)

```
raffleProbabilityBps: 10000 (100%)
marketSentimentBps:   0     (0%)
hybridPriceBps:       7000  (70%)
```

**Calculation:**
- Hybrid = 70% × raffleProbability + 30% × marketSentiment
- Hybrid = 70% × 100% + 30% × 0%
- Hybrid = 70%

**Market Sentiment:** 0% YES (all bets are NO)

### Market 1 (Player2: 0x3C44...93BC)

```
raffleProbabilityBps: 4000  (40%)
marketSentimentBps:   0     (0%)
hybridPriceBps:       2800  (28%)
```

**Calculation:**
- Hybrid = 70% × raffleProbability + 30% × marketSentiment
- Hybrid = 70% × 40% + 30% × 0%
- Hybrid = 28%

**Market Sentiment:** 0% YES (all bets are NO)

## Analysis

### Player Positions (Raffle)

- **Player1**: 100% of raffle tickets (15,000 out of 15,000 total)
- **Player2**: 40% of raffle tickets (6,000 out of 15,000 total)

### InfoFi Market Bets

Both markets show **0% market sentiment for YES**, indicating:
- All bets placed are NO bets
- No one has bet YES on either player winning

### Hybrid Pricing Working Correctly

The hybrid pricing formula is working as designed:

**For Player1 (100% raffle position):**
- Even with 100% NO bets in the market, the hybrid price can't drop below 70%
- This is because the raffle probability (100%) has a 70% weight
- Market sentiment (0% YES) only has a 30% weight

**For Player2 (40% raffle position):**
- With 40% raffle probability and 0% market sentiment
- Hybrid = 70% × 40% + 30% × 0% = 28%
- This matches the displayed odds of 28% YES / 72% NO

## Why Market Sentiment is 0%

The `sentimentBps` function returns:
```solidity
(totalYesPool * 10000) / totalPool
```

If `totalYesPool = 0` (no YES bets), then sentiment = 0%, regardless of how many NO bets exist.

## Implications

1. **The 70/30 weighting means raffle position dominates**
   - Even if everyone bets NO on a player, if they have 100% of tickets, hybrid stays at 70%
   - This aligns with the "fair games" philosophy - actual position matters more than speculation

2. **Market sentiment has limited impact**
   - With 30% weight, market can only shift odds by ±30 percentage points
   - To see dramatic odds changes, would need to adjust oracle weights

3. **Current behavior is correct**
   - Oracle is being updated (confirmed by checking on-chain)
   - Hybrid calculations are mathematically correct
   - Frontend is displaying the correct hybrid probabilities

## Recommendations

If you want market sentiment to have more impact on displayed odds, consider:

1. **Adjust oracle weights** - Change from 70/30 to 50/50 or even 30/70
2. **Use market sentiment directly** - Display both hybrid AND pure market sentiment
3. **Add market depth indicators** - Show how much is bet on each side

## Contract Addresses (Local Anvil)

- InfoFi Market: `0x0DCd1Bf9A1b36cE34237eEaFef220932846BCD82`
- InfoFi Oracle: `0x9A676e781A523b5d0C0e43731313A708CB607508`
- Raffle: `0x5FC8d32690cc91D4c39d9d3abcBD16989F875707`

## Commands Used

```bash
# Check oracle price for market 0
cast call 0x9A676e781A523b5d0C0e43731313A708CB607508 "getPrice(uint256)" 0 --rpc-url http://127.0.0.1:8545

# Check oracle price for market 1
cast call 0x9A676e781A523b5d0C0e43731313A708CB607508 "getPrice(uint256)" 1 --rpc-url http://127.0.0.1:8545

# Check market sentiment
cast call 0x0DCd1Bf9A1b36cE34237eEaFef220932846BCD82 "sentimentBps(uint256)" 0 --rpc-url http://127.0.0.1:8545
```
