# SeasonGating Design Document

**Date:** 2026-02-01
**Status:** Draft
**Feature:** Participation Requirements for Raffle Seasons

---

## Overview

Add gating capabilities to raffle seasons, requiring users to meet certain requirements before purchasing tickets. The system uses a modular, upgradeable architecture with AND-based composition for multiple requirements.

**Key Decisions:**
- Separate `SeasonGating.sol` contract (keeps Raffle.sol clean)
- PASSWORD gate type as highest priority
- Once-per-user verification model
- AND-only composition (all requirements must be met)
- Upgradeable proxy pattern for future gate types

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Raffle.sol                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  SeasonConfig                                         │  │
│  │  - bool gated                                         │  │
│  │  - ... existing fields                                │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                 │
│                   buyTickets() checks                        │
│                            ▼                                 │
└────────────────────────────┼────────────────────────────────┘
                             │
                    isUserVerified(seasonId, user)
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│               SeasonGating.sol (Upgradeable)                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Storage                                              │  │
│  │  - seasonGates[seasonId] → GateConfig[]               │  │
│  │  - userVerified[seasonId][user][gateIndex] → bool     │  │
│  │  - passwordHashes[seasonId][gateIndex] → bytes32      │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Gate Types:                                                │
│  - PASSWORD: User submits password, verified against hash   │
│  - ALLOWLIST: Merkle proof verification (future)            │
│  - TOKEN_GATE: Minimum token balance check (future)         │
│  - SIGNATURE: Off-chain signature verification (future)     │
└─────────────────────────────────────────────────────────────┘
```

---

## Contract: SeasonGating.sol

### Storage Structures

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/access/AccessControlUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/utils/ReentrancyGuardUpgradeable.sol";

enum GateType {
    NONE,           // 0 - No gate (placeholder)
    PASSWORD,       // 1 - Simple password verification
    ALLOWLIST,      // 2 - Merkle proof allowlist (future)
    TOKEN_GATE,     // 3 - Minimum token balance (future)
    SIGNATURE       // 4 - Off-chain signature (future)
}

struct GateConfig {
    GateType gateType;
    bool enabled;
    bytes32 configHash;  // Interpretation depends on gateType
                         // PASSWORD: keccak256(password)
                         // ALLOWLIST: merkleRoot
                         // TOKEN_GATE: keccak256(abi.encode(tokenAddress, minBalance))
                         // SIGNATURE: trusted signer address (as bytes32)
}
```

### Core Functions

```solidity
/// @notice Configure gates for a season (called by admin/Raffle)
/// @param seasonId The season to configure
/// @param gates Array of gate configurations (AND logic - all must pass)
function configureGates(uint256 seasonId, GateConfig[] calldata gates) external;

/// @notice Check if user has passed all gates for a season
/// @param seasonId The season to check
/// @param user The user address
/// @return verified True if user has passed all required gates
function isUserVerified(uint256 seasonId, address user) external view returns (bool);

/// @notice Submit password to verify for PASSWORD gate
/// @param seasonId The season
/// @param gateIndex Which gate in the array
/// @param password The plaintext password
function verifyPassword(uint256 seasonId, uint256 gateIndex, string calldata password) external;

/// @notice Get gate configuration for a season
/// @param seasonId The season
/// @return gates Array of gate configs
function getSeasonGates(uint256 seasonId) external view returns (GateConfig[] memory);

/// @notice Check verification status for specific gate
/// @param seasonId The season
/// @param gateIndex The gate index
/// @param user The user address
/// @return verified True if this specific gate is passed
function isGateVerified(uint256 seasonId, uint256 gateIndex, address user) external view returns (bool);
```

### Events

```solidity
event GatesConfigured(uint256 indexed seasonId, uint256 gateCount);
event UserVerified(uint256 indexed seasonId, uint256 indexed gateIndex, address indexed user, GateType gateType);
event GateAdded(uint256 indexed seasonId, uint256 gateIndex, GateType gateType);
event GateRemoved(uint256 indexed seasonId, uint256 gateIndex);
```

### Custom Errors

```solidity
error InvalidSeasonId();
error GateNotEnabled();
error InvalidGateIndex();
error InvalidPassword();
error AlreadyVerified();
error GateTypeMismatch();
error EmptyPassword();
```

---

## Integration with Raffle.sol

### Changes to SeasonConfig

```solidity
struct SeasonConfig {
    string name;
    uint256 startTime;
    uint256 endTime;
    uint8 winnerCount;
    uint16 grandPrizeBps;
    address treasuryAddress;
    address raffleToken;
    address bondingCurve;
    bool isActive;
    bool isCompleted;
    bool gated;  // NEW: If true, check SeasonGating contract
}
```

### Changes to Raffle State

```solidity
// Add to RaffleStorage.sol
address public gatingContract;

// Or per-season if different gating contracts needed
mapping(uint256 => address) public seasonGatingContracts;
```

### Changes to buyTickets()

```solidity
function buyTickets(uint256 seasonId, uint256 amount, uint256 maxCost) external nonReentrant {
    // ... existing validation ...

    // NEW: Check gating requirements
    SeasonConfig storage config = seasonConfigs[seasonId];
    if (config.gated && gatingContract != address(0)) {
        if (!ISeasonGating(gatingContract).isUserVerified(seasonId, msg.sender)) {
            revert UserNotVerified(seasonId, msg.sender);
        }
    }

    // ... rest of function ...
}
```

---

## PASSWORD Gate Implementation Details

### Password Flow

1. **Admin Setup (Create Season):**
   - Admin enters plaintext password in UI
   - Frontend hashes: `keccak256(abi.encodePacked(password))`
   - Hash stored on-chain in `GateConfig.configHash`

2. **User Verification:**
   - User enters password in UI
   - Frontend calls `verifyPassword(seasonId, gateIndex, password)`
   - Contract hashes input and compares to stored hash
   - If match: marks `userVerified[seasonId][user][gateIndex] = true`

3. **Ticket Purchase:**
   - `buyTickets()` calls `isUserVerified()`
   - Returns true only if ALL gates are verified

### Password Security Considerations

```solidity
function verifyPassword(
    uint256 seasonId,
    uint256 gateIndex,
    string calldata password
) external {
    if (bytes(password).length == 0) revert EmptyPassword();

    GateConfig storage gate = seasonGates[seasonId][gateIndex];
    if (gate.gateType != GateType.PASSWORD) revert GateTypeMismatch();
    if (!gate.enabled) revert GateNotEnabled();
    if (userVerified[seasonId][msg.sender][gateIndex]) revert AlreadyVerified();

    bytes32 inputHash = keccak256(abi.encodePacked(password));
    if (inputHash != gate.configHash) revert InvalidPassword();

    userVerified[seasonId][msg.sender][gateIndex] = true;
    emit UserVerified(seasonId, gateIndex, msg.sender, GateType.PASSWORD);
}
```

**Note:** Password is sent in plaintext to the contract. This is acceptable because:
- Passwords are per-season, not user credentials
- Anyone who knows the password can verify
- The hash prevents enumeration of passwords

---

## Frontend Integration

### Admin Panel: Create Season Form

Add "Participation Requirements" section:

```jsx
// New component: GatingConfig.jsx
const GatingConfig = ({ onChange }) => {
  const [gated, setGated] = useState(false);
  const [gates, setGates] = useState([]);

  return (
    <div className="space-y-4">
      <div className="flex items-center gap-2">
        <Switch checked={gated} onCheckedChange={setGated} />
        <label>Enable Participation Requirements</label>
      </div>

      {gated && (
        <div className="space-y-3 border rounded-lg p-4">
          {gates.map((gate, i) => (
            <GateEditor key={i} gate={gate} onChange={...} />
          ))}
          <AddGateButton onAdd={...} />
        </div>
      )}
    </div>
  );
};
```

### User Flow: Password Verification

```jsx
// New component: GatingVerification.jsx
const PasswordGate = ({ seasonId, gateIndex, onVerified }) => {
  const [password, setPassword] = useState('');
  const { mutate, isPending, isError } = useVerifyPassword();

  return (
    <div className="flex gap-2">
      <Input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Enter password"
      />
      <Button
        onClick={() => mutate({ seasonId, gateIndex, password })}
        disabled={isPending}
      >
        {isPending ? 'Verifying...' : 'Submit'}
      </Button>
    </div>
  );
};
```

---

## File Structure

```
contracts/src/
├── core/
│   ├── Raffle.sol           # Add gated check to buyTickets
│   └── RaffleStorage.sol    # Add gatingContract address
├── gating/
│   ├── ISeasonGating.sol    # Interface
│   ├── SeasonGating.sol     # Main implementation
│   └── SeasonGatingStorage.sol  # Upgradeable storage
```

---

## Implementation Steps

### Phase 1: Core Contract (Priority)

1. Create `ISeasonGating.sol` interface
2. Create `SeasonGatingStorage.sol` with storage layout
3. Create `SeasonGating.sol` with PASSWORD gate
4. Add unit tests for PASSWORD verification
5. Deploy as upgradeable proxy

### Phase 2: Raffle Integration

6. Add `bool gated` to SeasonConfig
7. Add `gatingContract` to RaffleStorage
8. Modify `buyTickets()` to check gating
9. Add custom error `UserNotVerified`
10. Update CreateSeasonForm to include gating config

### Phase 3: Frontend Components

11. Create `GatingConfig.jsx` for admin panel
12. Create `GatingVerification.jsx` for user verification
13. Add verification state to raffle card UI
14. Add hooks: `useVerifyPassword`, `useGatingStatus`

### Phase 4: Future Gate Types (Deferred)

- ALLOWLIST with Merkle proofs
- TOKEN_GATE for token holders
- SIGNATURE for off-chain verification

---

## Gas Estimates

| Operation | Estimated Gas |
|-----------|---------------|
| configureGates (1 gate) | ~50,000 |
| verifyPassword | ~45,000 |
| isUserVerified (1 gate) | ~5,000 |
| isUserVerified (3 gates) | ~12,000 |
| buyTickets gating check | +5,000-15,000 |

---

## Testing Strategy

```solidity
// test/SeasonGating.t.sol
contract SeasonGatingTest is Test {
    function testPasswordVerification() public { ... }
    function testInvalidPasswordReverts() public { ... }
    function testOncePerUserVerification() public { ... }
    function testMultipleGatesANDLogic() public { ... }
    function testGatedBuyTicketsBlocked() public { ... }
    function testGatedBuyTicketsAfterVerification() public { ... }
}
```

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Password sent in plaintext | Acceptable for per-season passwords; not user credentials |
| Upgradeable proxy risks | Use OpenZeppelin's battle-tested implementation |
| Gas cost on buyTickets | Single external call; cache result if needed |
| Front-running password | Not exploitable - verifier gets marked, attacker gains nothing |

---

## Open Questions (Resolved)

1. ~~Password mode?~~ → Once per user
2. ~~Composition logic?~~ → AND only
3. ~~Upgradeability?~~ → Upgradeable proxy
4. ~~Separate contract?~~ → Yes, modular approach

---

## Appendix: Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ISeasonGating {
    enum GateType { NONE, PASSWORD, ALLOWLIST, TOKEN_GATE, SIGNATURE }

    struct GateConfig {
        GateType gateType;
        bool enabled;
        bytes32 configHash;
    }

    function configureGates(uint256 seasonId, GateConfig[] calldata gates) external;
    function isUserVerified(uint256 seasonId, address user) external view returns (bool);
    function verifyPassword(uint256 seasonId, uint256 gateIndex, string calldata password) external;
    function getSeasonGates(uint256 seasonId) external view returns (GateConfig[] memory);
    function isGateVerified(uint256 seasonId, uint256 gateIndex, address user) external view returns (bool);
}
```
