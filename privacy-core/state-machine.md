# Note state machine (OrchardVerifier)

`OrchardVerifier` is the **on-chain state machine** for Privacy Core. All layers (pERC20 and future protocols on `IPERC20`) share this execution semantics.

## Scope

```
OrchardVerifier DOES:              DOES NOT:
✓ Groth16 verification             ✗ ERC-20 metadata
✓ Merkle append                    ✗ totalSupply accounting
✓ nullifier anti double-spend      ✗ external ERC-20/ETH custody
✓ binding + spendAuth sigs         ✗ swap / HTLC logic
✓ frozenRoot binding
```

Each pool instance has its **own** tree, nullifier set, verifier ref, and `cmxFrozenRoot`.  
**Asset identity = contract address** (same bytecode on different chains = different state).

## State

| State | Description |
|-------|-------------|
| `_tree` | depth-32 Poseidon incremental Merkle tree |
| `_isSpent[nf]` | Spent nullifiers |
| `cmxExists[cmx]` | Inserted commitments (no duplicate leaves) |
| `_allRootsEver[root]` | Permanent historical roots |
| `_frozenRoot` | Compliance IMT root |
| `groth16Verifier` | Groth16 verifier reference |
| `admin` | Compliance / config authority |

## Bundle execution order

```
1. For each BundleAction:
   a. pubFields[8] canonicality (< Fr)
   b. pubFields ↔ calldata equality
   c. Groth16 verifyAction
   d. spendAuthSig verify
2. bindingSig verify (before state writes)
3. For each action:
   a. mark nfOld spent
   b. insert cmx (reject zero / duplicate)
   c. emit NoteAdded / NoteConfirmed
```

**Invariant**: no public `bundle()` entrypoint — only app-layer `mint`/`burn`/`transfer` call `_executeBundle` (audit C-1).

## Views

| Method | Meaning |
|--------|---------|
| `cmxRoot()` | Latest tree root |
| `isValidAnchor(root)` | Root was ever valid |
| `isSpent(nf)` | Nullifier spent |
| `treeSize()` | Commitment count |

## Next

- [Groth16 action circuit](groth16-action-circuit.md)
- [Universal call format](universal-call-format.md)
