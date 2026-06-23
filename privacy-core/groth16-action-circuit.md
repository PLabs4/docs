# Groth16 action circuit

ZK core of Privacy Core: `circuits/action.circom`, Groth16 over BN254.

## Parameters

| Parameter | Value |
|-----------|-------|
| Constraints | **131,787** |
| Private inputs | 2,361 |
| Public inputs (circuit) | 1 (`pub_hash`) |
| On-chain public fields | 8 (hashed to `pub_hash`) |
| Powers of Tau | pot18 |
| Proof size | **256 bytes** |
| On-chain verify gas | ~188K (pairing only) |

## Sub-circuits

| Sub-circuit | Proves |
|-------------|--------|
| `NoteCommitOld` / `NoteCommitNew` | Commitments + 64-bit value range |
| `Merkle32` | Old note membership under `anchor` |
| `NullifierOld` | Correct `nf_old` |
| `ValueCommit` | Net value commitment `cv_net` |
| `SpendAuthRk` | Randomized auth key `rk` |
| `CommitIvkPkD` | `ivk` bound to address |
| `FrozenCmxNonMember` | Input not in freeze set (IMT depth 20) |
| 5× `BJJSubgroupCheck` | Witness curve points in prime subgroup |
| `PubHashAction` | 8 fields → single `pub_hash` |

## pubFields[8] pipeline

```
calldata pubFields[0..7]
    → ActionPubHash.hash()  (Poseidon sponge)
    → pub_hash
    → Groth16PairingVerifier.verifyProof
```

| Index | Field |
|-------|-------|
| 0 | anchor |
| 1–2 | cv_net_x/y |
| 3 | nf_old |
| 4–5 | rk_x/y |
| 6 | cmx |
| 7 | rt_frozen (**compliance root**) |

**H-1**: each `pubFields[i] < Fr` or `nf + Fr` replays with different `isSpent` key.

## Next

- [Signatures](signatures.md)
- [Compliance freeze](compliance-freeze.md)
