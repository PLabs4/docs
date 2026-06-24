# Compliance freeze design

Compliance freeze is Privacy Core’s **explicit extension** over Zcash Orchard: the circuit proves spent notes are **not** on an admin-maintained blacklist.

## Goals

| Goal | Mechanism |
|------|-----------|
| Sanctions / blacklist | Off-chain set, on-chain root updates |
| Hide list contents | Only Merkle root public |
| No bypass | Prove **input** `cm_old` non-membership (not output `cmx`) |

## Indexed Merkle Tree (IMT)

| Parameter | Value |
|-----------|-------|
| Depth | 20 |
| Leaf format | Ordered `(val, next_val)` low-leaf chain |
| Empty root | `0` |
| On-chain | `cmxFrozenRoot()` |

Admin rebuilds IMT off-chain, then `setFrozenRoot(newRoot)`.

## Circuit: `FrozenCmxNonMember`

Witness includes low-leaf bracket + IMT path proving `cm_old` falls in a gap → not in frozen sorted set.

### Why bind cm_old not cmx (audit H-01)

Binding only output `cmx` lets attacker spend frozen note with fresh output randomness. **Input commitment** must be constrained.

## On-chain: pubFields[7]

```
pubFields[7] == cmxFrozenRoot()   // else BadFrozenRoot
```

Each action’s Groth16 public input commits to this root via `pub_hash`.

## Trust trade-off

Admin can freeze identified notes. Should use multisig / timelock in production.

## Shared by applications

- **pERC20**: per-pool `cmxFrozenRoot`
- **pSWAP**: each leg must pass non-membership in its pool

## Admin workflow

1. Identify `cmx` to freeze off-chain
2. Rebuild IMT → new root
3. `setFrozenRoot(newRoot)`
4. Wallets/provers use new root in witness

## Next

- [Universal call format](universal-call-format.md)
- [pERC20 overview](perc20/overview.md)
