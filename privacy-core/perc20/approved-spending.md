# pERC20 · Approved spending

Maps ERC-20 `approve` / `allowance` / `transferFrom` to **ZIP-32 subaccounts**. No on-chain `allowance` mapping.

## Why subaccounts?

Public `allowance(owner, spender, value)` permanently links owner ↔ spender. Each EOA spender gets an **isolated subaccount** with its own keys.

## Flow

| Wallet op | On-chain | Off-chain |
|-----------|----------|-----------|
| `approve(spender, N)` | one `transfer` (fund subaccount) | derive subaccount, deliver spending key |
| `allowance` | — | scan subaccount with viewing key |
| `transferFrom` | one `transfer` | spender spends subaccount |
| revoke | one `transfer` (sweep back) | — |

## EOA spenders only

No native `approve(contract)` — contracts cannot hold spending keys safely on-chain.

## Deep dive

Full ZIP-32 mapping: `docs/zip-32-approve-extend.md` (to be merged into this chapter).

Normative ERC for this feature: **[ERC-8302](eip-8302.md)** ([PR #1817](https://github.com/ethereum/ERCs/pull/1817)).

## Next

- [Issuance & deployment](issuance-and-deployment.md)
