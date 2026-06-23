# Issuer guide

How to launch a private fungible token on pERC20.

## Overview

1. Deploy shared infrastructure (verifier + factory) — once per chain
2. Clone a new asset pool via factory
3. Configure issuer wallet + compliance policy
4. Mint initial supply to shielded notes
5. Distribute to holders (transfer / airdrop)

## Step 1 — Deploy factory stack

```bash
# See script/DeployOfficial.s.sol, script/deploy-all-chains.sh
forge script script/DeployOfficial.s.sol --broadcast
```

Deploys:

- `ActionGroth16Verifier` (Groth16 pairing verifier)
- `PERC20Factory` (EIP-1167 implementation + factory)

Cost: ~9M gas full stack (~$433 mainnet @ 20 gwei / ~$0.09 L2).

## Step 2 — Create asset

```solidity
factory.createPerc20("MyToken", "MTK", 6, issuerAddress);
```

Gas: ~321K per pool.

Emits `Perc20Created` with pool address (= token contract).

## Step 3 — Issuer responsibilities

| Role | Capability |
|------|------------|
| **Issuer** | `mint`, holds `issuer` address |
| **Admin** (= issuer by default) | `setFrozenRoot`, `setGroth16Verifier`, admin transfer |

## Step 4 — Mint

Issuer wallet:

1. Builds mint action witness (dummy input, `v = 0`)
2. Generates Groth16 proof
3. Calls `mint(amount, PrivacyCall)`

**Public**: mint amount, `totalSupply` increase  
**Private**: recipient

## Step 5 — Distribution

- **Direct transfer**: holder receives encrypted note at `perc1` address
- **Airdrop**: store-service + relayer pattern (see `docs/multichain.md`)
- **Approved spending**: holders use ZIP-32 subaccounts for EOA delegation

## Compliance

- Maintain off-chain blacklist SMT
- Call `setFrozenRoot(newRoot)` when list updates

## Web issuance UI

Reference: `apps/web/src/issuance/` — LaunchPanel, IssuancePanel, asset registry.

## Next

- [pERC20 issuance](../privacy-core/perc20/issuance-and-deployment.md)
- [Multi-chain deployment](../ecosystem/multichain.md)
- [Compliance freeze](../privacy-core/compliance-freeze.md)
