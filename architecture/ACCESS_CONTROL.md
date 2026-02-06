# SecondOrder.fun Access Control Architecture

**Last Updated:** February 6, 2026

This document provides an overview of access control mechanisms across SecondOrder.fun, linking related systems together.

---

## Overview

SecondOrder.fun uses a layered access control model:

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| **Smart Contract** | OpenZeppelin AccessControl | On-chain role-based permissions |
| **Automation** | SeasonLifecycleService | Automated state transitions |
| **Future** | Hats Protocol | Permissionless stake-based roles |

---

## 1. Smart Contract Roles (Current)

The `Raffle.sol` contract uses OpenZeppelin's AccessControl with these roles:

| Role | Purpose | Current Holder |
|------|---------|----------------|
| `DEFAULT_ADMIN_ROLE` | Grant/revoke roles, emergency functions | Team multisig |
| `SEASON_CREATOR_ROLE` | Create new seasons/raffles | Backend wallet + approved creators |
| `PRIZE_DISTRIBUTOR_ROLE` | Trigger prize distribution | VRF callback (internal) |

### Granting Roles

```solidity
// Admin grants SEASON_CREATOR_ROLE
raffle.grantRole(SEASON_CREATOR_ROLE, creatorAddress);
```

---

## 2. Season Lifecycle Automation

**Service:** `sof-backend/src/services/seasonLifecycleService.js`  
**Deployed:** February 6, 2026 (commit `dd7621b`)

Automates season state transitions that previously required manual admin action:

| Trigger | Condition | Action |
|---------|-----------|--------|
| Start Season | `startTime ≤ now < endTime` AND status=`NotStarted` | Calls `startSeason(seasonId)` |
| Request End | `now ≥ endTime` AND status=`Active` | Calls `requestSeasonEnd(seasonId)` |

### Configuration

| Env Var | Default | Description |
|---------|---------|-------------|
| `SEASON_LIFECYCLE_INTERVAL_MS` | `300000` (5 min) | Check frequency |

### Alerts

Sends Telegram notifications on:
- Successful season starts
- Successful season end requests
- Any failures (with error details)

---

## 3. Hats Protocol Integration (Future)

**Spec:** [HATS_PERMISSIONLESS_RAFFLE_CREATION.md](./HATS_PERMISSIONLESS_RAFFLE_CREATION.md)  
**Status:** Planned

Enables permissionless raffle creation through stake-based eligibility:

```
User stakes 50K $SOF → Gets "Raffle Creator" Hat → Can call createSeason()
```

### Key Features

- **Stake requirement:** 50,000 $SOF
- **Cooldown period:** 7 days before unstaking completes
- **Slashing:** Judges can slash misbehaving creators
- **Dual auth:** Supports BOTH hat ownership AND legacy `SEASON_CREATOR_ROLE`

### Hat Tree

```
Top Hat (DAO Safe)
├── Raffle Creator Hat (stake-gated)
├── Judge Hat (can slash)
└── Recipient Hat (receives slashed stakes → Treasury)
```

---

## Related Documents

- [Hats Permissionless Raffle Creation](./HATS_PERMISSIONLESS_RAFFLE_CREATION.md) - Full implementation spec
- Backend service: `sof-backend/src/services/seasonLifecycleService.js`

---

## Roadmap

1. ✅ **Phase 1:** Role-based access (current)
2. ✅ **Phase 2:** Automated lifecycle (deployed Feb 6)
3. 🔲 **Phase 3:** Hats stake-gated creation (in planning)
4. 🔲 **Phase 4:** Sponsorship system (TBD)

---

*Maintained by Briareos*
