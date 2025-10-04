# Terminal Event Resolution Strategy

## Decision: Markets Resolve at Season End Only

After analysis, we've decided **NOT to auto-resolve markets when players liquidate**. Instead, all InfoFi markets resolve together at the terminal event (season end via VRF).

## Rationale

### Why Terminal Event Resolution is Better

#### 1. **Richer Game Theory**

Players can liquidate and re-enter, creating speculation opportunities:

```
Timeline:
Day 1:  Player buys 1000 tickets → Market created at 10% odds
Day 5:  Player sells all tickets → Odds drop to 0%
Day 7:  Traders speculate: "Will they buy back in?"
Day 10: Player buys 2000 tickets → Odds jump to 15%
Day 14: Season ends → Market resolves based on VRF result
```

This creates a meta-game around player behavior, not just final outcomes.

#### 2. **More Trading Volume**

- Markets stay active even when position = 0
- Speculation on re-entry drives trading
- "Dead" markets can revive if player returns
- More volume = more platform fees

#### 3. **Simpler Smart Contract Logic**

- No complex auto-resolution on position changes
- One resolution event for all markets (season end)
- Single VRF call determines all outcomes
- Fewer edge cases to handle

#### 4. **Better Aligned with Product Vision**

SecondOrder.fun is about **second-order effects** - betting on player behavior, not just outcomes. Keeping markets open when position = 0 enables this.

## Implementation

### Smart Contract Behavior

**InfoFiMarketFactory.sol:**

```solidity
// When position changes to 0:
// 1. Update oracle to 0 BPS ✅
// 2. Emit ProbabilityUpdated event ✅
// 3. DO NOT resolve market ✅
// 4. Market stays active until season end ✅
```

**Resolution happens at season end:**

```solidity
// After VRF determines winner:
// 1. Lock all markets for the season
// 2. Resolve each market based on actual winner
// 3. Enable payout claims
```

### Frontend UI Strategy

#### Warning Display When Position = 0

```jsx
{Number(percent) === 0 && (
  <div className="bg-amber-50 border border-amber-300 rounded-lg p-3">
    <div className="flex items-center gap-2">
      <span>⚠️</span>
      <span className="font-medium">Player has no raffle position</span>
    </div>
    <p className="text-xs mt-1">
      Cannot win unless they buy back in before season ends
    </p>
  </div>
)}
```

#### Market Status Indicators

- **Active + Position > 0**: Green indicator, normal trading
- **Active + Position = 0**: Amber warning, trading still enabled
- **Locked**: Season ended, awaiting VRF
- **Resolved**: Winner determined, claims enabled

## User Experience Benefits

### For Traders

1. **More opportunities**: Can bet on player behavior changes
2. **Longer trading windows**: Markets don't close prematurely
3. **Strategic depth**: Predict re-entry, position changes
4. **Clear warnings**: UI shows when position = 0

### For Players

1. **Flexibility**: Can liquidate and re-enter same market
2. **No penalty**: Liquidation doesn't kill their market
3. **Strategic options**: Can bluff, change positions
4. **Fair resolution**: All markets resolve together at end

## Edge Cases Handled

### Case 1: Player Liquidates and Never Returns

- Market shows 0% odds (correct)
- Traders can still bet NO (will win)
- Market resolves to NO at season end
- NO bettors claim payouts

### Case 2: Player Liquidates Then Re-enters

- Odds drop to 0%, then rise when they buy back
- Traders who bet NO at 0% can profit if player doesn't win
- Traders who bet YES after re-entry get better odds
- Creates interesting arbitrage opportunities

### Case 3: Multiple Liquidations/Re-entries

- Oracle updates on every position change
- Market stays active throughout
- Final resolution based on actual VRF winner
- All intermediate positions are just speculation

### Case 4: Player Liquidates Right Before Season End

- Market shows 0% odds
- Too late to re-enter
- Effectively same as auto-resolve, but simpler
- Resolves to NO at season end

## Comparison with Auto-Resolve

| Aspect | Auto-Resolve | Terminal Event |
|--------|--------------|----------------|
| **Complexity** | High (position tracking) | Low (one event) |
| **Trading Volume** | Lower (markets close early) | Higher (stay open) |
| **Game Theory** | Simple (position = outcome) | Rich (behavior speculation) |
| **Re-entry** | Not possible (market closed) | Possible (market open) |
| **User Confusion** | Less (clear resolution) | More (needs UI warnings) |
| **Platform Fees** | Lower | Higher |
| **Product Vision** | Basic prediction | Second-order effects ✅ |

## Technical Implementation

### Contract Changes

**Removed auto-resolution logic:**

```solidity
// OLD (removed):
if (newTickets == 0 && oldTickets > 0) {
    resolveMarket(marketId, false);
}

// NEW (current):
// Just update oracle, market stays active
oracle.updateRaffleProbability(marketId, newBps);
```

### Frontend Changes

**Added warning component:**

```jsx
// InfoFiMarketCard.jsx
{Number(percent) === 0 && (
  <WarningBanner 
    message="Player has no position"
    detail="Cannot win unless they buy back in"
  />
)}
```

**Translation keys added:**

```json
{
  "playerHasNoPosition": "Player has no raffle position",
  "playerCannotWinUnlessReentry": "Cannot win unless they buy back in before season ends"
}
```

## Testing Scenarios

### Scenario 1: Normal Flow

1. Player crosses 1% → Market created
2. Player holds position → Odds stable
3. Season ends → Market resolves based on winner
4. Winners claim payouts

### Scenario 2: Liquidation Flow

1. Player crosses 1% → Market created at 10%
2. Player liquidates → Odds drop to 0%
3. Warning shows: "Player has no position"
4. Season ends → Market resolves to NO
5. NO bettors claim payouts

### Scenario 3: Re-entry Flow

1. Player crosses 1% → Market created at 10%
2. Player liquidates → Odds drop to 0%
3. Player re-enters → Odds rise to 15%
4. Season ends → Market resolves based on VRF
5. Correct side claims payouts

### Scenario 4: Multiple Changes

1. Market created at 10%
2. Player increases → 20%
3. Player liquidates → 0%
4. Player re-enters → 15%
5. Player increases → 25%
6. Season ends → Resolves based on final VRF result

## Monitoring & Analytics

Track these metrics to validate the strategy:

1. **Re-entry Rate**: % of liquidated players who buy back in
2. **Trading Volume**: Compare markets with position changes vs stable
3. **Speculation Accuracy**: Do traders correctly predict re-entry?
4. **User Confusion**: Support tickets about 0% odds markets

## Future Enhancements

1. **Re-entry Prediction Markets**: Meta-markets on whether player will return
2. **Position Change Alerts**: Notify traders when positions change
3. **Historical Position Chart**: Show position changes over time
4. **Liquidation Analytics**: Track patterns in player behavior

## Conclusion

**Terminal event resolution is the right choice** because:

- ✅ Aligns with "second-order" product vision
- ✅ Increases trading volume and platform fees
- ✅ Simpler smart contract implementation
- ✅ Enables richer game theory and strategy
- ✅ Allows player flexibility (liquidate + re-enter)

The trade-off is requiring clear UI warnings when position = 0, which we've implemented with the amber warning banner.
