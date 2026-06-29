# Shield ERC20 overview

**Shield ERC20** privatizes **existing public ERC-20** tokens. It is an **asset protocol** under Privacy Core (alongside [pERC20](../perc20/overview.md)), not a swap/payment application.

## One line

> Deposit a standard ERC-20 into custody → receive **private notes** backed 1:1; spend notes → **unshield** releases underlying to a bound recipient.

## Contract structure

```solidity
contract ERC20Shield is OrchardVerifier, IERC20Shield
```

| Inheritance | Provides |
|-------------|----------|
| `OrchardVerifier` | Privacy Core state machine |
| `IERC20Shield` | Custody metadata, `shield` / `unshield`, disabled `mint`/`burn` |
| `IPERC20` (via `IERC20Shield`) | Metadata, `totalSupply`, `transfer`, compliance freeze |

## pERC20 vs Shield ERC20

| | **pERC20 (`PERC20`)** | **Shield ERC20 (`ERC20Shield`)** |
|---|---|---|
| Value source | Issuer `mint` / holder `burn` | User `shield` / `unshield` |
| External custody | None (`totalSupply` is bookkeeping) | Yes — pool locks underlying ERC-20 |
| Permissionless deposit | No (`onlyIssuer` mint) | Yes — anyone may `shield` |
| Supply cap | Issuer policy | Pool's custodied balance |
| `mint` / `burn` | Enabled | **Disabled** (`MintDisabled` / `BurnDisabled`) |
| Typical use | Privacy-native issuance | Privatize USDC, WETH, etc. |

Cryptographically, **`mint` is shield without custody** and **`burn` is unshield without payout**. `ERC20Shield` adds the ERC-20 pull/transfer layer and enforces the custody invariant.

## Stack position

```
Shield ERC20  →  deposit / withdraw rules, ERC-20 custody, scale
       │
pERC20 / OrchardVerifier  →  shielded note state machine (shared Groth16 VK)
```

Built on the same [Privacy Core](../overview.md) primitives as [pERC20](../perc20/overview.md). Shield pools share the same circuit, verifier, and note format — they are **swap-compatible** with issuer-minted pERC20 assets via value-neutral `transfer`.

## Custody invariant

Every unit of shielded supply must be matched by custodied underlying:

```
underlying.balanceOf(pool) >= shieldedSupply * scale
```

- `shield` increases both sides by `amountUnits * scale`
- `unshield` decreases both sides by `amountUnits * scale`
- `transfer` is value-neutral — touches neither custody nor `shieldedSupply`

`>=` (not `==`) allows direct donations to the pool; excess underlying cannot be claimed by any note.

## Units

| Term | Meaning |
|------|---------|
| **Note unit** | Value field inside a shielded note (64-bit circuit range) |
| **`scale`** | Wei of underlying per note unit: `weiAmount = amountUnits * scale` |
| **`shieldedSupply`** | Total backed note units (also exposed as `totalSupply()`) |

Dust below one note unit (`< scale` wei) cannot be shielded.

## Implementation

| Contract | Path |
|----------|------|
| Pool | `contracts/ptoken/ERC20Shield.sol` |
| Factory | `contracts/ptoken/ERC20ShieldFactory.sol` |
| Interface | `contracts/interfaces/IERC20Shield.sol` |

## Next

- [Interface & pERC20 mapping](interface-and-perc20.md)
- [Operations: shield / unshield / transfer](operations.md)
- [Deployment & factory](deployment.md)
