# Privacy Core · Overview

**Privacy Core** is the foundation of the pERC20 stack. It has two layers:

1. **Cryptographic layer** — how value exists as **Orchard notes**, how actions are proved and verified, compliance freeze
2. **Asset protocols** — how **private assets** are created (`pERC20` native mint, `Shield ERC20` public → private)

[Applications](../applications/overview.md) (pSWAP, pDEX, pX402) **consume** those assets; they do not define note creation.

## Cryptographic layer

```
Orchard concepts        notes / commitments / nullifiers / encryption
Note state machine      OrchardVerifier on-chain logic
Groth16 action circuit  131,787 constraints, BN254
Signatures              bindingSig + spendAuthSig
Compliance freeze       IMT blacklist + pubFields[7] binding
Universal call format   PrivacyCall / BundleAction (shared by all layers)
```

| Component | Role |
|-----------|------|
| `OrchardVerifier.sol` | Note state machine: proofs, tree, nullifiers, signatures |
| `ActionGroth16Verifier.sol` | `pubFields[8]` → `pub_hash` → Groth16 pairing |
| `action.circom` | Single Orchard action + freeze non-membership |

## Asset protocols

How privacy assets enter the system:

| Protocol | Status | Role |
|----------|--------|------|
| [pERC20](perc20/overview.md) | ✅ Shipped | Native private fungible token — `mint` / `transfer` / `burn` on `PERC20` pool |
| [Shield ERC20](erc20-shield/overview.md) | 🔜 Coming soon | Deposit public ERC-20 → private notes; unshield back |

`PERC20.sol` **inherits** `OrchardVerifier` and adds supply + `IPERC20` — see [pERC20 overview](perc20/overview.md).

pSWAP / pDEX / pX402 call **`IPERC20.transfer`** (and future app rules) on pools created by these asset protocols.

## Relation to Zcash Orchard

| Inherited | EVM adaptation |
|-----------|----------------|
| Note format, nullifiers, note encryption | Groth16 instead of Orchard native proofs |
| Action value conservation, binding sig | On-chain BabyJubJub Schnorr (Solidity) |
| Shielded-pool semantics | **Per-asset pool** (not one global shielded pool) |
| — | **Compliance freeze** in-circuit (`FrozenCmxNonMember`) |

## Trust boundaries

| Component | Trust |
|-----------|-------|
| Groth16 proof | Trustless on-chain verify |
| Trusted setup | Per-circuit ceremony; ≥1 honest participant |
| binding / spendAuth signatures | On-chain verify |
| Relayer | Minimal trust today (cannot tamper ciphertext; can censor). **Planned:** integrate [Kohaku](https://github.com/ethereum/kohaku) ERC-4337 relaying to remove reliance on self-hosted `privacy-relayer` |
| Compliance admin | Can update freeze root (explicit trade-off) |

## Next

- [Orchard concepts](orchard-concepts.md)
- [pERC20 overview](perc20/overview.md)
