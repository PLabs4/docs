# Signatures: bindingSig & spendAuthSig

Besides Groth16, Privacy Core uses **BabyJubJub Schnorr** signatures verified on-chain in Solidity.

## Why two signatures?

| Signature | Level | Prevents |
|-----------|-------|----------|
| **bindingSig** | Bundle | Value not conserved |
| **spendAuthSig** | Per action | Relayer tampering with ciphertext |

Groth16 does not prove `encCiphertext` matches `cmx` (same as Zcash); spendAuth binds ciphertext to authorized key.

## bindingSig (bundle)

```
PrivacyCall.bindingSig = [Rx, Ry, s]
```

- Verified **before** Merkle writes
- Covers nullifiers, commitments, `valueBalance`

### valueBalance encoding (set by application layer)

| Case | valueBalance |
|------|--------------|
| Neutral transfer | `0` |
| Burn | bit255=0, low bits=amount |
| Mint | bit255=1, low bits=amount |

## spendAuthSig (per action)

```
sighash = keccak256(
  "SpendAuth..." ‖ chainId ‖ contractAddr
  ‖ nfOld ‖ cmx ‖ epk
  ‖ encCiphertext(580B) ‖ outCiphertext(80B)
)

Verify: s·G == R + e·rk
rk = (ask + alpha)·G_SPEND_AUTH  (= pubFields[4,5])
```

- `R` must be on-curve (audit L-1)
- `chainId` binding prevents cross-chain replay

## Gas share

Groth16 ~3% of per-action gas; **two Schnorrs + Poseidon insert + storage** ~90%.

## Next

- [Compliance freeze](compliance-freeze.md)
