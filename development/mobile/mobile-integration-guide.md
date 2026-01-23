# Mobile UI Integration Guide

## Quick Start Integration

This guide shows how to integrate the mobile UI components into your existing SecondOrder.fun application.

---

## Step 1: Update App.jsx

Wrap your app with the Shell component for automatic platform detection:

```jsx
// src/App.jsx
import { Shell } from "./components/shells";

function App() {
  return <Shell>{/* Your existing routes */}</Shell>;
}

export default App;
```

---

## Step 2: Update RaffleList Route

Add mobile conditional rendering:

```jsx
// src/routes/RaffleList.jsx
import { usePlatform } from "../hooks/usePlatform";
import MobileRafflesList from "../components/mobile/MobileRafflesList";
import { useState } from "react";
import BuySellSheet from "../components/mobile/BuySellSheet";

export default function RaffleList() {
  const { isMobile } = usePlatform();
  const [sheetOpen, setSheetOpen] = useState(false);
  const [sheetMode, setSheetMode] = useState("buy");
  const [selectedSeason, setSelectedSeason] = useState(null);

  // Your existing data fetching hooks
  const activeSeasons = []; // From your data source
  const allSeasons = []; // From your data source

  const handleBuy = (seasonId) => {
    setSelectedSeason(seasonId);
    setSheetMode("buy");
    setSheetOpen(true);
  };

  const handleSell = (seasonId) => {
    setSelectedSeason(seasonId);
    setSheetMode("sell");
    setSheetOpen(true);
  };

  if (isMobile) {
    return (
      <>
        <MobileRafflesList
          activeSeasons={activeSeasons}
          allSeasons={allSeasons}
          onBuy={handleBuy}
          onSell={handleSell}
        />
        <BuySellSheet
          open={sheetOpen}
          onOpenChange={setSheetOpen}
          mode={sheetMode}
          seasonId={selectedSeason}
          bondingCurveAddress={selectedSeason?.bondingCurve}
          maxSellable={selectedSeason?.userTickets ?? 0n}
          onSuccess={async (data) => {
            // Handle transaction
            console.log("Transaction:", data);
          }}
        />
      </>
    );
  }

  // Your existing desktop component
  return <YourDesktopRafflesList />;
}
```

---

## Step 3: Update RaffleDetails Route

Add mobile detail view:

```jsx
// src/routes/RaffleDetails.jsx
import { useParams, useNavigate } from "react-router-dom";
import { usePlatform } from "../hooks/usePlatform";
import MobileRaffleDetail from "../components/mobile/MobileRaffleDetail";
import { useState } from "react";
import BuySellSheet from "../components/mobile/BuySellSheet";

export default function RaffleDetails() {
  const { seasonId } = useParams();
  const navigate = useNavigate();
  const { isMobile } = usePlatform();
  const [sheetOpen, setSheetOpen] = useState(false);
  const [sheetMode, setSheetMode] = useState("buy");

  // Your existing data fetching hooks
  const seasonConfig = {}; // From useRaffleState
  const curveSupply = 0n; // From useCurveState
  const maxSupply = 0n; // From allBondSteps
  const curveStep = {}; // From useCurveState
  const localPosition = {}; // From your state
  const totalPrizePool = 0n; // From seasonDetails

  if (isMobile) {
    return (
      <>
        <MobileRaffleDetail
          seasonId={Number(seasonId)}
          seasonConfig={seasonConfig}
          curveSupply={curveSupply}
          maxSupply={maxSupply}
          curveStep={curveStep}
          localPosition={localPosition}
          totalPrizePool={totalPrizePool}
          onBuy={() => {
            setSheetMode("buy");
            setSheetOpen(true);
          }}
          onSell={() => {
            setSheetMode("sell");
            setSheetOpen(true);
          }}
          onBack={() => navigate("/raffles")}
        />
        <BuySellSheet
          open={sheetOpen}
          onOpenChange={setSheetOpen}
          mode={sheetMode}
          seasonId={Number(seasonId)}
          bondingCurveAddress={seasonConfig?.bondingCurve}
          maxSellable={localPosition?.tickets ?? 0n}
          onSuccess={async (data) => {
            // Handle transaction
            console.log("Transaction:", data);
          }}
        />
      </>
    );
  }

  // Your existing desktop component
  return <YourDesktopRaffleDetails />;
}
```

---

## Step 4: Update Tailwind Config

Add safe area utilities:

```js
// tailwind.config.js
export default {
  theme: {
    extend: {
      spacing: {
        "safe-top": "env(safe-area-inset-top)",
        "safe-bottom": "env(safe-area-inset-bottom)",
        "safe-left": "env(safe-area-inset-left)",
        "safe-right": "env(safe-area-inset-right)",
      },
    },
  },
};
```

---

## Step 5: Add CurveGraph Mini Prop

Update your CurveGraph component to support mini mode:

```jsx
// src/components/curve/CurveGraph.jsx
export const CurveGraph = ({ bondSteps, currentSupply, mini = false }) => {
  const height = mini ? 96 : 300; // 96px for mini, 300px for full
  const showLabels = !mini;
  const showGrid = !mini;

  // Your existing graph logic with conditional rendering
};
```

---

## Step 6: Create useCurveCalculations Hook

If it doesn't exist, create this hook:

```jsx
// src/hooks/useCurveCalculations.js
import { useReadContract } from "wagmi";
import SOFBondingCurveAbi from "../abis/SOFBondingCurveAbi";

export const useCurveCalculations = (bondingCurveAddress) => {
  const calculateBuyPrice = async (quantity) => {
    // Call contract to get buy price
    // Return estimated cost in wei
  };

  const calculateSellPrice = async (quantity) => {
    // Call contract to get sell price
    // Return estimated proceeds in wei
  };

  return {
    calculateBuyPrice,
    calculateSellPrice,
  };
};
```

---

## Testing Checklist

### Desktop Browser

- [ ] Shell renders WebShell
- [ ] Desktop components display correctly
- [ ] No mobile components visible

### Mobile Browser

- [ ] Shell renders WebShell (not mobile-specific)
- [ ] Responsive design works
- [ ] Touch interactions work

### Farcaster Mini App

- [ ] Shell renders MiniAppShell
- [ ] FID profile image displays in header
- [ ] Bottom navigation shows
- [ ] Safe area padding applies
- [ ] Carousel swipes smoothly
- [ ] Bottom sheet works

### Base App

- [ ] Shell renders DappBrowserShell
- [ ] Wallet connection works
- [ ] Bottom navigation shows
- [ ] Transactions execute

---

## Troubleshooting

### Profile Image Not Showing

- Check Farcaster SDK is initialized
- Verify `context.user.pfpUrl` is available
- Check fallback avatar displays

### Safe Area Not Working

- Ensure viewport meta tag includes `viewport-fit=cover`
- Check CSS env() variables are supported
- Verify useSafeArea hook is called

### Carousel Not Swiping

- Check CSS scroll-snap is supported
- Verify overflow-x is set correctly
- Test with touch events, not mouse

### Bottom Sheet Not Opening

- Check Sheet component from shadcn/ui is installed
- Verify open/onOpenChange props are wired correctly
- Check z-index stacking

---

## Performance Tips

1. **Lazy Load Mobile Components**

   ```jsx
   const MobileRafflesList = lazy(() => import("./mobile/MobileRafflesList"));
   ```

2. **Memoize Expensive Calculations**

   ```jsx
   const formattedPrice = useMemo(() => formatSOF(price), [price]);
   ```

3. **Debounce Quantity Changes**

   ```jsx
   const debouncedQuantity = useDebounce(quantity, 300);
   ```

4. **Virtualize Long Lists**
   - Use react-window for All Seasons list if >100 items

---

## Browser Support

### Minimum Requirements

- iOS Safari 14+
- Chrome Mobile 90+
- Firefox Mobile 88+
- Samsung Internet 14+

### Progressive Enhancement

- Safe area insets (iOS 11+)
- CSS scroll-snap (iOS 11+, Chrome 69+)
- CSS env() (iOS 11.2+)

---

## Deployment Checklist

- [ ] Test on real iOS device (iPhone)
- [ ] Test on real Android device
- [ ] Test in Farcaster mobile app
- [ ] Test in Base App
- [ ] Verify touch targets ≥44px
- [ ] Check color contrast ratios
- [ ] Test with slow 3G connection
- [ ] Verify safe area on iPhone with notch
- [ ] Test landscape orientation
- [ ] Check keyboard behavior with inputs

---

_Integration guide for SecondOrder.fun mobile UI v1.0_
