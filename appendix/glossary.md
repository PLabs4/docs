# Glossary

## Modules

| Term | Definition |
|------|------------|
| **Privacy Core** | Cryptographic layer + **asset protocols** (pERC20, Shield ERC20) |
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
| **perc1** | Privacy recipient address format |
| **Subaccount** | ZIP-32 per-spender account (approve) |

## Shield ERC20

| Term | Definition |
|------|------------|
| **Shield ERC20** | Public ERC-20 → private notes asset protocol (coming soon) |
