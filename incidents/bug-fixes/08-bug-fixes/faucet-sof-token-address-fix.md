# Faucet Revert Issue (Sepolia) - Wrong SOF Token Address

## Problem

Faucet claims reverted with an `ERC20InsufficientBalance`-style error, even though the faucet was funded.

## Root Cause

The faucet was configured with the **wrong `sofToken` contract address**.

- **Faucet `sofToken` address (wrong):** `0x1a4a7c6817982b63fa6eE0629f4112532bc03d85`
- **Correct SOF token address:** `0x452159a798d98981D5f964B0D93Aae7b79F45741`

Because the faucet pointed at the wrong ERC-20 contract, its balance in that token was `0`, so transfers reverted.

## Fix Implemented

### Contract Changes

The faucet contract was enhanced to support fixing configuration without redeploying:

- Added `setSofToken(address _sofToken)` so the owner can update the SOF token address
- Added debug events for troubleshooting
- Added more detailed logging inside `claim()` for future diagnosis

### Deployment Outcome

A new faucet was deployed and configured correctly:

- **SOF Token:** `0x452159a798d98981D5f964B0D93Aae7b79F45741`
- **Amount per request:** `1,000` SOF
- **Cooldown:** `6 hours`
- **Allowed chains:** `31337` (Anvil), `84532` (Base Sepolia)
- **Initial funding:** `100,000` SOF

## Environment Variables Updated

- Frontend:

  - `VITE_SOF_FAUCET_ADDRESS_TESTNET=0xA946df916A23819797d01Ba27b077Ef5Fbb1566e`

- Backend:

  - `SOF_FAUCET_ADDRESS_TESTNET=0xA946df916A23819797d01Ba27b077Ef5Fbb1566e`

## Verification

Successful claim verification checklist:

- Faucet is funded with the correct SOF token
- `claim()` succeeds
- User receives the configured amount
- Debug events (if enabled) show correct token address and balances
