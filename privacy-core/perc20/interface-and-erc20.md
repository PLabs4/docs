# pERC20 · Interface & ERC-20 mapping

pERC20 is **capability-complete** with ERC-20 but **not byte-compatible**.

## Full mapping

| Interface | ERC-20? | Layer | Openness | pERC20 |
|-----------|---------|-------|----------|--------|
| `name/symbol/decimals` | yes | on-chain | public | Same |
| `totalSupply()` | yes | on-chain | public | mint − burn |
| `balanceOf(addr)` | yes | off-chain | **private** | Viewing-key note scan |
| `transfer(...)` | yes | on-chain | **private** | `transfer(PrivacyCall)` |
| `approve(spender, N)` | yes | off-chain | **private** | ZIP-32 subaccount |
| `allowance(o, s)` | yes | off-chain | **private** | Subaccount balance |
| `transferFrom(...)` | yes | off-chain | **private** | Subaccount spend |
| `mint(...)` | extension | on-chain | amount public | Issuer only |
| `burn(...)` | extension | on-chain | amount public | Any holder |
| `issuer()` | — | on-chain | public | Issuer address |
| `cmxFrozenRoot/setFrozenRoot` | — | on-chain | root public | [Compliance](../compliance-freeze.md) |
| Tree views | — | on-chain | public | `cmxRoot`, `isSpent`, … |

## Events

| ERC-20 | pERC20 |
|--------|--------|
| `Transfer` | ❌ not emitted |
| `Approval` | ❌ not emitted |
| — | `NoteAdded` / `NoteConfirmed` |
| — | `Mint` / `Burn` (public amount only) |

## On-chain indistinguishability

`transfer`, approve funding, `transferFrom`, and revoke all appear as `transfer(PrivacyCall)`.

## Not supported

- `approve(contract, amount)` — contracts cannot hold spending keys
- Generic ERC-20 indexers without privacy wallet

## Relation to Privacy Core

All value changes become `BundleAction[]` executed by `OrchardVerifier`. pERC20 updates `_totalSupply` on mint/burn.

## Next

- [Operations](operations.md)
