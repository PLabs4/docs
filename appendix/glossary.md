# Glossary

## Modules

| Term | Definition |
|------|------------|
| **Privacy Core** | Cryptographic layer + **asset protocols** (pERC20, Shield ERC20) |
| **Applications** | Operations on privacy assets: pSWAP, pDEX, pX402 |
| **OrchardVerifier** | On-chain Privacy Core state machine |

## Privacy Core

| Term | Definition |
|------|------------|
| **Note** | Shielded value unit (commitment + encrypted plaintext) |
| **cmx** | Note commitment (Merkle leaf) |
| **Nullifier (nf)** | Spend tag preventing double-spend |
| **Action** | One spend/create step + Groth16 proof |
| **PrivacyCall** | On-chain payload: `actions` + `bindingSig` |
| **cmxFrozenRoot** | Compliance freeze IMT root |
| **bindingSig** | Bundle-level value-conservation Schnorr |
| **spendAuthSig** | Per-action anti-tamper Schnorr |

## pERC20

| Term | Definition |
|------|------------|
| **pERC20** | Native private fungible token asset protocol (Privacy Core) |
| **PERC20** | Reference contract name |
| **perc1** | Privacy recipient address — bech32m string, prefix `perc1` (EVM assets), e.g. `perc1q…`; decodes to 43-byte raw (`d[11] ‖ pk_d`) for prover |
| **Subaccount** | ZIP-32 per-spender account (approve) |

## Shield ERC20

| Term | Definition |
|------|------------|
| **Shield ERC20** | Public ERC-20 → private notes asset protocol |

## pSWAP

| Term | Definition |
|------|------------|
| **pSWAP** | Private atomic swap: coordinator registers legs, `settle` runs dual `IPERC20.transfer` |
| **SwapCoordinator** | Application contract: `initiateSwap` / `joinSwap` / `settle` |
| **htlcHash** | One-time `keccak256(preimage)` per swap; only preimage holder can `settle` |
| **preimage** | Secret `S` revealed at settle; binds LP leg to user leg (HTLC hashlink) |
| **commitA / commitB** | `keccak256(abi.encode(PrivacyCall))` per leg; register vs settle integrity |

## pDEX

| Term | Definition |
|------|------------|
| **pDEX** | Private decentralized exchange & routing (coming soon) |

## pX402

| Term | Definition |
|------|------------|
| **pX402** | Private HTTP 402 micropayments (coming soon) |
