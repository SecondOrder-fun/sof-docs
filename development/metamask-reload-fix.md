# MetaMask "Reload This Page" Issue - Fixed

## Problem

When loading the dapp in Chrome:
1. The MetaMask browser extension icon shows a white background
2. Clicking the icon shows: "Reload this page to apply your updated settings on this site."
3. Until reloading, the Connect Wallet button shows RainbowKit modal without MetaMask icon
4. This issue started after adding i18n support

## Root Cause

The `RainbowKitWrapper` component was using `useTranslation()` hook to dynamically set the locale:

```jsx
const RainbowKitWrapper = ({ children }) => {
  const { i18n } = useTranslation();
  const locale = localeMap[i18n.language] || 'en';
  
  return (
    <RainbowKitProvider locale={locale}>
      {children}
    </RainbowKitProvider>
  );
};
```

**Why this caused the issue:**
- When language changes, `useTranslation()` triggers a re-render
- This causes `RainbowKitProvider` to re-initialize with a new locale
- RainbowKit re-initialization causes the Ethereum provider to be re-injected
- MetaMask detects this provider change as a potential security issue
- MetaMask shows the "Reload this page" warning to ensure the page uses the correct provider

## Solution

Fixed by using a static locale instead of dynamic locale from i18n:

```jsx
const RainbowKitWrapper = ({ children }) => {
  return (
    <RainbowKitProvider locale="en">
      {children}
    </RainbowKitProvider>
  );
};
```

**Trade-off:**
- RainbowKit UI will always be in English
- Our app UI will still be fully translated via i18n
- This prevents MetaMask reload issues

## Alternative Solutions (Not Implemented)

If we need RainbowKit in multiple languages, we could:

1. **Use a separate locale state** that doesn't trigger provider re-initialization
2. **Implement custom wallet modal** instead of RainbowKit
3. **Use RainbowKit's custom theme** to hide text that needs translation

## Files Changed

### Core Fixes
- `src/context/WagmiConfigProvider.jsx` - Created singleton connectors to prevent re-initialization
- `src/main.jsx` - Removed dynamic locale from RainbowKitProvider
- `src/components/layout/Header.jsx` - Changed `chainStatus="icon"` to `chainStatus="none"` to hide white MetaMask icon

### Automatic Migration for Existing Users
- `src/main.jsx` - Added version-based cache clearing mechanism

```javascript
const CACHE_VERSION = '1.0.1';
const CURRENT_VERSION = localStorage.getItem('app_version');
if (CURRENT_VERSION !== CACHE_VERSION) {
  localStorage.setItem('app_version', CACHE_VERSION);
  if (CURRENT_VERSION) {
    window.location.reload(true);
  }
}
```

This ensures:
- New users get the fixed code immediately
- Existing users automatically get a one-time hard reload to clear old cached connectors
- No manual intervention required from users
- Version tracking prevents unnecessary reloads

## Testing

1. Load the dapp in Chrome (first-time users)
2. Check MetaMask extension icon - should have normal background
3. Click Connect Wallet - should show MetaMask option immediately
4. Change language - should not trigger MetaMask reload warning

For existing users:
1. Load the dapp - will automatically hard reload once
2. After reload, everything works normally
3. No more MetaMask reload warnings
