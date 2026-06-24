# Universal call format: PrivacyCall / BundleAction

The **shared on-chain payload** exposed by Privacy Core to all applications.

## PrivacyCall

```solidity
struct PrivacyCall {
    bytes actions;           // abi.encode(BundleAction[])
    uint256[3] bindingSig;
}
```

## BundleAction

```solidity
struct BundleAction {
    bytes32    cmx;
    bytes      encCiphertext;    // 580 B
    bytes      outCiphertext;    // 80 B
    bytes32    epk;
    bytes32    nfOld;
    bytes32    anchor;
    bytes      proof;            // Groth16 256 B
    uint256[8] pubFields;
    uint256[3] spendAuthSig;
}
```

## Usage by applications

| Application | Call |
|-------------|------|
| pERC20 transfer | `transfer(PrivacyCall)` |
| pERC20 mint/burn | `mint` / `burn` + valueBalance |
| pERC20 approve (on-chain step) | `transfer(PrivacyCall)` |

## Calldata size

Single-action `transfer(PrivacyCall)` ≈ **1,900–1,960 B (~1.9 KB)** when ABI-encoded. Measured from real bundle encoding ([Performance benchmarks](../ecosystem/performance.md), 2026-06-23).

Groth16 proof is only **256 B raw**, so calldata is **not proof-dominated** — the largest field is **`encCiphertext`** (580 B raw).

### Per-field breakdown (`BundleAction` + `PrivacyCall`)

| Field | Raw | ABI-encoded | Notes |
|-------|-----|-------------|-------|
| `cmx` | 32 B | 32 B | |
| `encCiphertext` | 580 B | 640 B | 32 B length word + 580 B + 28 B padding |
| `outCiphertext` | 80 B | 128 B | 32 B length word + 80 B + 16 B padding |
| `epk` | 32 B | 32 B | |
| `nfOld` | 32 B | 32 B | |
| `anchor` | 32 B | 32 B | |
| `proof` | 256 B | 288 B | `abi.encode(pA[2], pB[2][2], pC[2])`; 32 B length + 256 B |
| `pubFields` | 256 B | 256 B | `uint256[8]` |
| `spendAuthSig` | 96 B | 96 B | `uint256[3]` |
| ABI dynamic offsets + array/tuple headers | — | ~300 B | `bytes actions` wrapper + 3 dynamic `bytes` offsets |
| `bindingSig` | 96 B | 96 B | `PrivacyCall` layer |
| Function selector | — | 4 B | `mint` / `burn` add +32 B for `amount` |

## Multi-action bundles

Default `maxActions = 10`, cap 16. One `bindingSig` per bundle.

## Encoding reference

- `contracts/interfaces/IEndpointCore.sol`
- `apps/web/src/issuance/bundleCodec.ts`

## Next

→ [Applications overview](../applications/overview.md)
