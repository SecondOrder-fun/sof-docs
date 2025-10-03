# Arbitrage Detection Flow Diagram

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INTERFACE                               │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │         ArbitrageOpportunityDisplay Component                  │  │
│  │  • Live indicator badge                                        │  │
│  │  • Opportunity cards with profitability                        │  │
│  │  • Strategy explanations                                       │  │
│  │  • Manual refresh button                                       │  │
│  └───────────────────────┬───────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      REACT HOOKS LAYER                               │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │         useArbitrageDetectionLive Hook                         │  │
│  │  • Subscribes to oracle PriceUpdated events                   │  │
│  │  • Triggers recalculation on price changes                     │  │
│  │  • Sets isLive indicator                                       │  │
│  └───────────────────────┬───────────────────────────────────────┘  │
│                          │                                           │
│                          ▼                                           │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │         useArbitrageDetection Hook                             │  │
│  │  • Fetches active markets                                      │  │
│  │  • Reads oracle prices                                         │  │
│  │  • Calculates raffle costs                                     │  │
│  │  • Compares prices                                             │  │
│  │  • Filters by profitability                                    │  │
│  │  • Sorts and limits results                                    │  │
│  └───────────────────────┬───────────────────────────────────────┘  │
└────────────────────────────┼────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     ON-CHAIN DATA SOURCES                            │
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ InfoFiMarket     │  │ InfoFiPriceOracle│  │ SOFBondingCurve  │  │
│  │ Factory          │  │                  │  │                  │  │
│  │                  │  │                  │  │                  │  │
│  │ • getSeasonPlayers│  │ • getPrice()    │  │ • curveConfig   │  │
│  │ • hasWinnerMarket│  │ • weights()      │  │ • getCurrentStep│  │
│  │ • MarketCreated  │  │ • PriceUpdated   │  │ • getBondSteps  │  │
│  │   event          │  │   event          │  │                  │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow Sequence

```
1. INITIALIZATION
   ┌─────────────────────────────────────────────────────────────┐
   │ Component mounts with seasonId + bondingCurveAddress        │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ useArbitrageDetectionLive subscribes to oracle events       │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ useArbitrageDetection fetches initial data:                 │
   │ • Markets from InfoFiMarketFactory                           │
   │ • Curve state from SOFBondingCurve                           │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼

2. CALCULATION LOOP (for each market)
   ┌─────────────────────────────────────────────────────────────┐
   │ Get oracle price for market                                  │
   │ ↓                                                             │
   │ readOraclePrice(marketId) → PriceData                        │
   │   • hybridPriceBps (70% raffle + 30% sentiment)             │
   │   • raffleProbabilityBps                                     │
   │   • marketSentimentBps                                       │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Get player position                                          │
   │ ↓                                                             │
   │ getPlayerPosition(player) → Position                         │
   │   • probability (basis points)                               │
   │   • tickets                                                  │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Calculate raffle entry cost                                  │
   │ ↓                                                             │
   │ ticketsNeeded = (probability / 10000) * totalSupply          │
   │ raffleCost = ticketsNeeded * currentTicketPrice              │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Convert oracle price to SOF                                  │
   │ ↓                                                             │
   │ marketPriceSOF = hybridPriceBps / 10000                      │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Calculate arbitrage metrics                                  │
   │ ↓                                                             │
   │ priceDiff = |raffleCost - marketPriceSOF|                    │
   │ avgPrice = (raffleCost + marketPriceSOF) / 2                 │
   │ profitability = (priceDiff / avgPrice) * 10000 (bps)         │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Filter by threshold                                          │
   │ ↓                                                             │
   │ if (profitability >= minProfitabilityBps) {                  │
   │   add to opportunities[]                                     │
   │ }                                                             │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼

3. RESULT PROCESSING
   ┌─────────────────────────────────────────────────────────────┐
   │ Sort opportunities by profitability (descending)             │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Limit to top N results (default: 10)                         │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Return to component                                          │
   │ • opportunities[]                                            │
   │ • isLoading                                                  │
   │ • error                                                      │
   │ • isLive                                                     │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼

4. REAL-TIME UPDATES
   ┌─────────────────────────────────────────────────────────────┐
   │ Oracle emits PriceUpdated event                              │
   │ ↓                                                             │
   │ event PriceUpdated(                                          │
   │   bytes32 indexed marketId,                                  │
   │   uint256 raffleBps,                                         │
   │   uint256 marketBps,                                         │
   │   uint256 hybridBps,                                         │
   │   uint256 timestamp                                          │
   │ )                                                             │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ useArbitrageDetectionLive receives event                     │
   │ ↓                                                             │
   │ setIsLive(true)                                              │
   │ refetch()  // Trigger recalculation                          │
   └────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Component updates with new opportunities                     │
   │ • Live badge shows green pulsing indicator                   │
   │ • Opportunity list refreshes                                 │
   │ • Last update timestamp updates                              │
   └─────────────────────────────────────────────────────────────┘
```

## Arbitrage Strategy Decision Tree

```
                    ┌─────────────────────┐
                    │  Compare Prices     │
                    │  raffleCost vs      │
                    │  marketPriceSOF     │
                    └──────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
    ┌───────────────────────┐   ┌───────────────────────┐
    │ raffleCost <          │   │ marketPriceSOF <      │
    │ marketPriceSOF        │   │ raffleCost            │
    └──────────┬────────────┘   └──────────┬────────────┘
               │                           │
               ▼                           ▼
    ┌───────────────────────┐   ┌───────────────────────┐
    │ STRATEGY A:           │   │ STRATEGY B:           │
    │ "Buy Raffle"          │   │ "Buy Market"          │
    │                       │   │                       │
    │ 1. Buy raffle tickets │   │ 1. Buy InfoFi position│
    │    at lower cost      │   │    at lower cost      │
    │                       │   │                       │
    │ 2. Sell InfoFi        │   │ 2. Exit raffle        │
    │    position at higher │   │    position at higher │
    │    price              │   │    price              │
    │                       │   │                       │
    │ Profit:               │   │ Profit:               │
    │ marketPrice -         │   │ raffleCost -          │
    │ raffleCost            │   │ marketPrice           │
    └───────────────────────┘   └───────────────────────┘
```

## Profitability Color Coding

```
┌─────────────────────────────────────────────────────────────┐
│                  Profitability Ranges                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  🟢 GREEN (High Profit)                                     │
│  ≥ 10%                                                       │
│  • Excellent arbitrage opportunity                          │
│  • High confidence in profit                                │
│  • Recommended for execution                                │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  🟡 YELLOW (Medium Profit)                                  │
│  5% - 10%                                                    │
│  • Good arbitrage opportunity                               │
│  • Consider gas costs                                       │
│  • Moderate confidence                                      │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  🟠 ORANGE (Low Profit)                                     │
│  3% - 5%                                                     │
│  • Marginal opportunity                                     │
│  • Gas costs may eat into profit                            │
│  • Proceed with caution                                     │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ⚪ GRAY (Minimal Profit)                                   │
│  2% - 3%                                                     │
│  • Barely profitable                                        │
│  • High risk of loss after gas                              │
│  • Not recommended                                          │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ⛔ FILTERED OUT                                            │
│  < 2%                                                        │
│  • Below minimum threshold                                  │
│  • Not displayed to users                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Component State Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    COMPONENT STATES                          │
└─────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │  MOUNTING    │
    │  isLoading:  │
    │    true      │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  LOADING     │◄─────────────┐
    │  • Spinner   │              │
    │  • "Scanning"│              │ refetch()
    └──────┬───────┘              │
           │                      │
           ▼                      │
    ┌──────────────┐              │
    │  LOADED      │──────────────┘
    │  isLoading:  │
    │    false     │
    └──────┬───────┘
           │
           ├──────────────┬──────────────┬──────────────┐
           │              │              │              │
           ▼              ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  ERROR   │   │  EMPTY   │  │ SUCCESS  │  │  LIVE    │
    │          │   │          │  │          │  │          │
    │ • Error  │   │ • No     │  │ • Show   │  │ • Green  │
    │   message│   │   opps   │  │   cards  │  │   badge  │
    │ • Retry  │   │ • Edu    │  │ • Sorted │  │ • Auto   │
    │   button │   │   text   │  │   list   │  │   update │
    └──────────┘   └──────────┘  └──────────┘  └──────────┘
```

## Event Subscription Flow

```
┌─────────────────────────────────────────────────────────────┐
│              ORACLE EVENT SUBSCRIPTION                       │
└─────────────────────────────────────────────────────────────┘

1. Component Mount
   ↓
   useEffect(() => {
     setupSubscription()
   }, [seasonId])
   ↓
2. Setup Subscription
   ↓
   const { subscribeOraclePriceUpdated } = await import(...)
   ↓
3. Subscribe to Events
   ↓
   unsubscribe = subscribeOraclePriceUpdated({
     networkKey,
     onEvent: (log) => {
       setIsLive(true)
       refetch()
     }
   })
   ↓
4. Listen for Events
   ↓
   ┌─────────────────────────────────────┐
   │ Oracle Contract                     │
   │                                     │
   │ emit PriceUpdated(                  │
   │   marketId,                         │
   │   raffleBps,                        │
   │   marketBps,                        │
   │   hybridBps,                        │
   │   timestamp                         │
   │ )                                   │
   └──────────────┬──────────────────────┘
                  │
                  ▼
   ┌─────────────────────────────────────┐
   │ WebSocket Transport                 │
   │ (viem watchContractEvent)           │
   └──────────────┬──────────────────────┘
                  │
                  ▼
   ┌─────────────────────────────────────┐
   │ onEvent Callback                    │
   │ • setIsLive(true)                   │
   │ • refetch()                         │
   └──────────────┬──────────────────────┘
                  │
                  ▼
   ┌─────────────────────────────────────┐
   │ Recalculate Opportunities           │
   │ • Fetch latest prices               │
   │ • Recompute profitability           │
   │ • Update UI                         │
   └─────────────────────────────────────┘
   ↓
5. Component Unmount
   ↓
   return () => {
     setIsLive(false)
     unsubscribe?.()
   }
```

## Performance Optimization

```
┌─────────────────────────────────────────────────────────────┐
│              OPTIMIZATION STRATEGIES                         │
└─────────────────────────────────────────────────────────────┘

1. CACHING
   ┌────────────────────────────────────┐
   │ Oracle prices cached in-memory     │
   │ • Reduces redundant RPC calls      │
   │ • Faster recalculations            │
   └────────────────────────────────────┘

2. EARLY EXIT
   ┌────────────────────────────────────┐
   │ Skip inactive markets              │
   │ • Check oracle.active flag         │
   │ • Skip if below threshold          │
   └────────────────────────────────────┘

3. BATCHING
   ┌────────────────────────────────────┐
   │ Batch RPC calls when possible      │
   │ • Use multicall for reads          │
   │ • Parallel async operations        │
   └────────────────────────────────────┘

4. DEBOUNCING
   ┌────────────────────────────────────┐
   │ Debounce rapid updates             │
   │ • Prevent UI thrashing             │
   │ • Smooth user experience           │
   └────────────────────────────────────┘

5. LAZY LOADING
   ┌────────────────────────────────────┐
   │ Load markets on-demand             │
   │ • Only fetch when needed           │
   │ • Reduce initial load time         │
   └────────────────────────────────────┘
```

---

**Note:** This diagram provides a visual overview of the arbitrage detection system architecture and data flow. For detailed implementation, see `doc/arbitrage-detection.md`.
