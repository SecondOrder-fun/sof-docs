# InfoFi Payout Calculation Examples

## How Payouts Work

The payout system calculates potential returns based on the current market odds (probability) for each outcome.

### Formula

```
Payout = Bet Amount / (Probability / 100)
Profit = Payout - Bet Amount
```

## Example Scenarios

### Scenario 1: Balanced Market (50/50 odds)

**Market State:**
- YES: 50%
- NO: 50%

**Betting 10 SOF on YES:**
- Payout: 10 / (50/100) = **20 SOF**
- Profit: 20 - 10 = **+10 SOF**

**Betting 10 SOF on NO:**
- Payout: 10 / (50/100) = **20 SOF**
- Profit: 20 - 10 = **+10 SOF**

### Scenario 2: Favorite Market (75/25 odds)

**Market State:**
- YES: 75% (favorite)
- NO: 25% (underdog)

**Betting 10 SOF on YES (favorite):**
- Payout: 10 / (75/100) = **13.33 SOF**
- Profit: 13.33 - 10 = **+3.33 SOF**

**Betting 10 SOF on NO (underdog):**
- Payout: 10 / (25/100) = **40 SOF**
- Profit: 40 - 10 = **+30 SOF**

### Scenario 3: Heavy Favorite (90/10 odds)

**Market State:**
- YES: 90% (heavy favorite)
- NO: 10% (long shot)

**Betting 100 SOF on YES:**
- Payout: 100 / (90/100) = **111.11 SOF**
- Profit: 111.11 - 100 = **+11.11 SOF** (11% return)

**Betting 100 SOF on NO:**
- Payout: 100 / (10/100) = **1,000 SOF**
- Profit: 1,000 - 100 = **+900 SOF** (900% return!)

## UI Display Behavior

### Default Display (No Bet Amount Entered)

Shows payout for a 1 SOF bet:

```
YES (65%)
65 SOF
per 1 SOF bet

NO (35%)
2.86 SOF
per 1 SOF bet
```

### With Bet Amount (User enters 50 SOF)

Shows actual payout and profit for entered amount:

```
YES (65%)
76.92 SOF
+26.92 profit

NO (35%)
142.86 SOF
+92.86 profit
```

## Real-Time Updates

The payout display updates automatically when:

1. **User changes bet amount** - Payout scales instantly
2. **Market odds change** - Recalculates based on new probability
3. **Player positions change** - Oracle price updates trigger recalculation

## Edge Cases

### Near-Certain Outcome (99% odds)

**Betting 1000 SOF on YES (99%):**
- Payout: 1000 / 0.99 = **1,010.10 SOF**
- Profit: **+10.10 SOF** (1% return)

**Betting 1000 SOF on NO (1%):**
- Payout: 1000 / 0.01 = **100,000 SOF**
- Profit: **+99,000 SOF** (9,900% return)

### Zero Probability Protection

If odds reach 0% or 100%, the calculation returns 0 to prevent division errors.

## Formatting Rules

The display automatically formats based on amount:

- **< 100 SOF**: Shows 2 decimals (e.g., "12.50 SOF")
- **100-999 SOF**: Shows whole number (e.g., "150 SOF")
- **≥ 1000 SOF**: Shows k notation (e.g., "1.5k SOF")

## User Benefits

1. **Transparency**: See exact returns before placing bet
2. **Risk Assessment**: Compare potential profits across outcomes
3. **Strategic Planning**: Calculate optimal bet sizes
4. **Live Feedback**: Instant updates as you type
