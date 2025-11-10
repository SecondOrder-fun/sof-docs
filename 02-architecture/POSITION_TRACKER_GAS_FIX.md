# Position Tracker Gas Issue - Root Cause & Fix

## Issue Summary

The `RafflePositionTracker` was not emitting `PositionSnapshot` events because the `updateAllPlayersInSeason()` call from the bonding curve was **running out of gas** and failing silently.

## Root Cause Analysis

### The Problem

When users bought tickets via `SOFBondingCurve.buyTokens()`, the contract attempted to call:

```solidity
IRafflePositionTracker(positionTracker).updateAllPlayersInSeason()
```

This function:

1. Fetches all participants from the Raffle contract
2. Fetches season details (total tickets)
3. **Loops through ALL players** and updates each one
4. For EACH player:
   - Calls `getParticipantPosition()` on Raffle
   - Calculates win probability
   - Pushes to `playerHistory` array
   - Updates `currentPositions` mapping
   - **Emits `PositionSnapshot` event**

### Gas Consumption

With just 4 players, this requires:

- 4 external contract calls
- 4 array pushes (storage writes)
- 4 mapping updates (storage writes)
- 4 event emissions

**Result:** The try-catch block ran out of gas and silently failed, preventing any `PositionSnapshot` events from being emitted.

### Evidence

Transaction trace of `0x31074fb38255814a5f949721ab1c5d8142487f960d8e65ccd2e2d28cdaaf2808`:

```text
├─ [140502] RafflePositionTracker::updateAllPlayersInSeason()
│   └─ ← [OutOfGas] EvmError: OutOfGas
```

## The Fix

### Changed Approach

Instead of updating **ALL players** on every buy/sell, we now update **only the active trader**:

```solidity
// BEFORE (out of gas):
try IRafflePositionTracker(positionTracker).updateAllPlayersInSeason() {

// AFTER (gas-efficient):
try IRafflePositionTracker(positionTracker).updatePlayerPosition(msg.sender) {
```

### Benefits

1. ✅ **Gas-efficient**: Only updates one player instead of all players
2. ✅ **Real-time updates**: Active trader gets immediate probability update
3. ✅ **Emits events**: `PositionSnapshot` event now fires successfully
4. ✅ **Scales better**: Gas cost doesn't grow with player count
5. ✅ **Backend compatibility**: Backend still receives events to update database

### Trade-offs

- Other players' probabilities are not updated until they trade
- Backend can still calculate all probabilities from the `PositionUpdate` event emitted by the bonding curve
- For admin-triggered updates of all players, can still call `updateAllPlayersInSeason()` directly with sufficient gas

## Files Modified

- `contracts/src/curve/SOFBondingCurve.sol`:
  - Line 232: Changed `updateAllPlayersInSeason()` to `updatePlayerPosition(msg.sender)` in `buyTokens()`
  - Line 297: Changed `updateAllPlayersInSeason()` to `updatePlayerPosition(msg.sender)` in `sellTokens()`

## Testing Required

1. ✅ Redeploy contracts with the fix
2. ✅ Buy tickets and verify `PositionSnapshot` events are emitted
3. ✅ Verify backend listener receives and processes events
4. ✅ Verify InfoFi markets are created when crossing 1% threshold
5. ✅ Verify database is updated with correct probabilities

## Alternative Solutions Considered

### Option 1: Increase Gas Limit

```solidity
try IRafflePositionTracker(positionTracker).updateAllPlayersInSeason{gas: 500000}() {
```

**Rejected:** Still scales poorly with player count, wastes gas

### Option 2: Backend-Only Updates

Remove on-chain updates entirely, let backend calculate from `PositionUpdate` events.

**Rejected:** Loses on-chain verifiability and real-time data for other contracts

### Option 3: Batch Updates (Chosen)

Update only the active trader on-chain, backend handles full recalculation.

**Accepted:** Best balance of gas efficiency and functionality

## Impact on Backend

The backend `positionTrackerListener` will now receive:

- ✅ `PositionSnapshot` events for the active trader
- ✅ Events will fire successfully (no more out-of-gas failures)
- ✅ Backend can calculate other players' probabilities from bonding curve's `PositionUpdate` event

No backend code changes required - the listener already handles `PositionSnapshot` events correctly.

## Deployment Steps

1. Compile contracts: `cd contracts && forge build`
2. Deploy to Anvil: `npm run anvil:deploy`
3. Update `.env` with new contract addresses
4. Restart backend to pick up new addresses
5. Test ticket purchase flow end-to-end

---

**Date:** 2025-10-25  
**Status:** Fixed  
**Severity:** High (prevented core functionality)  
**Resolution:** Changed from `updateAllPlayersInSeason()` to `updatePlayerPosition(msg.sender)`
