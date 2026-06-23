# Performance benchmarks

Summary of measured pERC20 performance. Full tables: **`docs/perc20-performance-benchmarks-en.md`**.

## Highlights (2026-06-23, 8 vCPU Xeon)

### On-chain gas (single action)

| Operation | Gas | L2 cost (~0.04 gwei) |
|-----------|-----|----------------------|
| `mint()` | 5,683,559 | ~$0.055 |
| `transfer()` | 5,472,352 | ~$0.053 |
| `burn()` | 4,961,594 | ~$0.048 |
| `createPerc20()` | 321,587 | ~$0.003 |

Groth16 verify alone: **188,387 gas (~3%)**. Dominant cost: Schnorr + Poseidon tree.

### Calldata

Single action ≈ **1.9 KB** (Groth16 proof = 256 bytes).

### Proving time

| Path | Time |
|------|------|
| rapidsnark (exclusive) | ~1.0 s |
| rapidsnark (production load) | ~2.1 s |
| snarkjs (JS fallback) | ~7.3 s |

### vs PrivacyBTC (Halo2/KZG)

| Metric | pERC20 | PrivacyBTC |
|--------|--------|------------|
| Proof size | 256 B | 8,896 B |
| Verify gas | ~188K | ~4.26M |
| Prove time | ~1 s | ~40 s |

## Next

- [Groth16 action circuit](../privacy-core/groth16-action-circuit.md)
- [pERC20 operations](../privacy-core/perc20/operations.md)
