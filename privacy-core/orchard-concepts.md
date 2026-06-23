# Orchard concepts

Privacy Core uses the **[Orchard shielded pool](https://zips.z.cash/protocol/names#orchard-shielded-pool)** data model ([Zcash Protocol Specification](https://zips.z.cash/protocol/protocol.pdf)), deployed on EVM with **[Groth16](groth16-action-circuit.md)** and a **[per-asset Merkle tree](#commitment-tree)**.

## Notes vs account balances

Public tokens use **account balances**:

```
balance[Alice] = 100
balance[Bob]   = 50
```

Privacy Core uses **shielded notes**:

```
Tree: { cmx₁, cmx₂, cmx₃, ... }   // commitments only on-chain
Alice holds: notes decryptable with her viewing key
Bob holds:   notes decryptable with his viewing key
```

There is **no** on-chain `mapping(address => uint256) balance`.

## Note lifecycle

```
Create (mint / transfer output)
    → cmx inserted into Merkle tree
    → encCiphertext in NoteAdded event
         │
Hold (off-chain scan + trial-decrypt)
         │
Spend (transfer / burn input)
    → reveal nfOld (nullifier)
    → Groth16 proves knowledge + non-freeze
    → create output note (transfer) or destroy (burn)
```

## Cryptographic objects

| Object | Role | On-chain visibility |
|--------|------|---------------------|
| **cmx** | Note commitment, Merkle leaf | ✅ commitment |
| **nf** | Nullifier, anti double-spend | ✅ on spend |
| **anchor** | Historical Merkle root for input | ✅ |
| **encCiphertext** | Recipient-encrypted note (580 B) | ✅ ciphertext |
| **outCiphertext** | Sender OVK recovery (80 B) | ✅ ciphertext |
| **v** | Note value | ❌ constrained in circuit only |

## Action

One **action** = optionally consume **one input note** + create **one output note**, with:

1. **Value conservation** (`valueBalance` + bindingSig)
2. **Input membership** (Merkle32 under `anchor`)
3. **Correct nullifier** (`nf_old` from `nk, rho, psi, cm_old`)
4. **Output commitment** (`NoteCommitNew`)
5. **Spend authorization** (`rk` + spendAuthSig)
6. **Compliance non-membership** (input `cm_old` not in freeze set) — [Compliance freeze](compliance-freeze.md)

Mint uses a **dummy input** (`v_old = 0`) on the same path as transfer.

## Commitment tree

| Parameter | Value |
|-----------|-------|
| Depth | 32 |
| Hash | Poseidon (incremental insert) |
| Views | `cmxRoot()`, `isValidAnchor(root)`, `treeSize()` |

Historical roots are permanently queryable (`_allRootsEver`) for spending against old anchors.

## Keys (summary)

| Key | Use |
|-----|-----|
| Spending key | Nullifiers, spendAuthSig |
| Viewing key (ivk) | Scan events, trial-decrypt |
| Address (pk_d / perc1) | Recipient identifier |

Wallet derivation is application-layer; the circuit binds `ivk` to `pk_d` via `CommitIvkPkD`.

## Privacy guarantees & limits

**Hidden**: sender, recipient, transfer amounts (mint/burn amounts are public at app layer).

**Visible**: `NoteAdded` blobs, `totalSupply` (pERC20), submitter EOA (unless relayer).

**Important**: `NoteAdded` alone does not prove payment — recipient must trial-decrypt.

## Next

- [Note state machine](state-machine.md)
- [Groth16 action circuit](groth16-action-circuit.md)
