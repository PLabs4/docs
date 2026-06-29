# Shield ERC20 · Interface & pERC20 mapping

`ERC20Shield` implements `IERC20Shield`, which extends `IPERC20`. Wallets and SDKs reuse the same `PrivacyCall` / `BundleAction` encoding as [pERC20](../perc20/interface-and-erc20.md).

## Full mapping

| Interface | pERC20 | Shield ERC20 |
|-----------|--------|--------------|
| `name/symbol/decimals` | On-chain, public | Same (pool metadata; independent of underlying) |
| `totalSupply()` | `mint − burn` (note units) | `shieldedSupply` (note units) |
| `issuer()` | Token issuer | **`admin`** (compliance officer; no mint power) |
| `underlying()` | — | Custodied ERC-20 address |
| `scale()` | — | Wei per note unit |
| `shieldedSupply()` | — | Backed note units (alias of `totalSupply`) |
| `balanceOf(addr)` | Off-chain viewing key | Same |
| `transfer(PrivacyCall)` | Private note→note | Same |
| `transfer(executor, call)` | Executor-gated (swap) | Same |
| `mint(...)` | Issuer only | **Disabled** — use `shield` |
| `burn(...)` | Any holder | **Disabled** — use `unshield` |
| `shield(...)` | — | Deposit underlying → shielded note |
| `unshield(...)` | — | Spend note → release underlying |
| `cmxFrozenRoot/setFrozenRoot` | Admin | Same — [Compliance](../compliance-freeze.md) |

## Events

| pERC20 | Shield ERC20 |
|--------|--------------|
| `Perc20Created` | `ShieldPoolCreated` (+ `underlying`, `scale`) |
| `Mint` / `Burn` | ❌ not emitted (mint/burn disabled) |
| — | `Shielded(depositor, amountUnits, weiAmount)` |
| — | `Unshielded(recipient, amountUnits, weiAmount)` |
| `NoteAdded` / `NoteConfirmed` | Same (all value changes) |
| `FrozenRootUpdated` | Same |

Shield/unshield emit **public amounts** (note units + wei); depositor/recipient identity for shield is not indexed on-chain beyond `msg.sender` on `Shielded`.

## IPERC20 compatibility notes

- **`mint` / `burn` revert** with `MintDisabled` / `BurnDisabled`. Integrators must route deposit/withdraw through `shield` / `unshield`.
- **`issuer()`** returns `admin`, not a mint authority. For shield pools, treat `admin` as the compliance officer only.
- **Asset identity is the shield pool address**, not `(underlying, name)`. Permissionless factory deployment means look-alike pools are possible — verify `ERC20ShieldFactory.isShieldPool(pool)` before interacting.

## ERC-20 layer (public)

| Step | On-chain (public) | Private |
|------|-------------------|---------|
| Shield | `approve(pool, amountUnits * scale)` then `shield` pulls ERC-20 | Output note recipient |
| Unshield | `unshield(..., recipient, call)` sends ERC-20 to `recipient` | Spent notes, change note |
| Transfer | — | Parties and amount |

Only **standard ERC-20** tokens are supported. Fee-on-transfer and rebasing tokens are rejected at `shield` via balance-delta check (`NonStandardToken`).

## Relation to Privacy Core

All note state transitions go through `_executeBundle` on `OrchardVerifier`. `ERC20Shield` adds custody accounting (`shieldedSupply`) and ERC-20 pull/transfer around shield/unshield.

## Next

- [Operations](operations.md)
- [Deployment](deployment.md)
