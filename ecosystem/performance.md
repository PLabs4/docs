# Performance benchmarks

> **Measurement Environment**
> - **Test environment (proving / deployment)**: AWS EC2 `13.193.190.113`
>   - CPU: Intel(R) Xeon(R) Platinum 8124M @ 3.00 GHz (4 physical cores / 8 vCPU, hyper-threading)
>   - Memory: 15 GiB
>   - OS: Ubuntu 24.04, Docker Compose multi-service stack (web / store-service / per-chain indexer·relayer·prover-service / privacy-prover)
> - **Proof system**: Circom 2.x + **Groth16** (snarkjs 0.7.5 for artifact generation; **rapidsnark** for on-chain proving path, snarkjs as pure-JS fallback)
> - **Curve / field**: BN254 (alt_bn128), scalar field BN254 Fr (254 bit)
> - **Trusted setup**: Powers of Tau `pot18` (2¹⁸ = 262,144 field elements) + Groth16 phase-2 (one ceremony per circuit)
> - **Contract toolchain**: Foundry `forge 1.4.3`, Solidity `^0.8.20`; gas measured on a real EVM via `forge test` with **real Groth16 proofs + real BabyJubJub Schnorr signatures + real verifier contracts** (`test/PERC20E2E.t.sol`, no mocks)
> - Measurement date: 2026-06-23
> - **Price reference**: ETH = $2,400 USD (fixed reference value for cost estimates only)

> **Fundamental difference vs PrivacyBTC**
> pERC20 uses **Circom/Groth16**; PrivacyBTC uses **Halo2/KZG**. Direct consequences:
> - Proof size **256 bytes** (Groth16 fixed 8 field elements) vs PrivacyBTC **8,896 bytes** (≈ 35× smaller).
> - On-chain verification only **~188K gas** (2× `ecPairing` + 1× `ecMul`/`ecAdd`) vs PrivacyBTC **~4.26M gas** (40-step verifier) ≈ 23× cheaper.
> - Trade-off: each circuit requires a one-time **trusted setup** (Groth16 phase-2 ceremony); Halo2/KZG uses universal setup.
> - Proving time **~1 s (rapidsnark)** vs PrivacyBTC ~40 s.

---

## 1. On-Chain Gas Cost

### Conversion Baseline

```
ETH = $2,400 USD (fixed)
Ethereum mainnet gas price = 20 gwei (moderate congestion)
Typical L2 (Arbitrum / Base) gas price = 0.04 gwei

1 gas × 20 gwei   = 0.00000002 ETH    = $0.000048
1 gas × 0.04 gwei = 0.000000000004 ETH = $0.0000000096
```

> pERC20 is currently deployed on Ethereum, Base, Arbitrum, and Monad (see §3.2). Execution logic is identical; gas magnitudes match. The table below gives mainnet (20 gwei) and typical L2 (0.04 gwei) cost tiers.

### 1.1 Business Operations (`PERC20.sol` + `OrchardVerifier`, single action)

> **Design note (per action)**
> - `mint()` (issuer only): output-only, `valueBalance = amount | (1<<255)`; inserts a new note.
> - `transfer()`: value-neutral (`valueBalance = 0`); consumes an old note + creates a new note.
> - `burn()`: `valueBalance = amount`; public supply decreases.
> - Each on-chain action runs: **Groth16 proof verification** + **bindingSig** (BabyJubJub Schnorr, value conservation) + **spendAuthSig** (BabyJubJub Schnorr, anti-relayer ciphertext tampering) + **depth-32 Poseidon Merkle insert** + nullifier / cmx dedup writes + events.
> - One bundle may contain multiple actions (`maxActions` default 10, cap 16).

| Operation | Gas (measured, single action) | Ethereum mainnet cost | L2 cost | Notes |
|-----------|------------------------------|----------------------|---------|-------|
| `mint()` | **5,683,559** | **0.1137 ETH ≈ $273** | **≈ $0.055** | First insert (more cold SSTORE) |
| `transfer()` | **5,472,352** | **0.1094 ETH ≈ $263** | **≈ $0.053** | Value-neutral, 1 note consumed + 1 produced |
| `burn()` | **4,961,594** | **0.0992 ETH ≈ $238** | **≈ $0.048** | Subsequent insert (partially warm SSTORE) |
| `createPerc20()` (EIP-1167 clone, new asset pool) | **321,587** | **0.0064 ETH ≈ $15.4** | **≈ $0.0031** | Factory minimal proxy clone + `initialize` |
| `setFrozenRoot()` (compliance blacklist root update) | 23,539 (warm) / 71,654 (cold) | $1.1 / $3.4 | < $0.001 | admin-only |
| `isSpent()` / `isValidAnchor()` (read-only view) | ~2,600 (single SLOAD scale) | $0.12 | < $0.001 | nullifier / anchor query |

> The gap between `mint` and `burn` (~720K gas) is almost entirely **cold vs warm storage**: first writes touch more uninitialized Merkle padding subtree slots (cold SSTORE 20,000 gas/slot). Steady-state single action ≈ **5.0–5.5M gas**.

### 1.2 Gas Cost Breakdown (Key Insight)

| Component | Gas | Share | Measured / estimated |
|-----------|-----|-------|----------------------|
| **Groth16 proof verification** (`verifyProof`, pairing only) | 188,387 | ~3.3% | ✅ measured |
| `verifyAction` (= Groth16 + `ActionPubHash` Poseidon(8) + ABI decode) | 355,590 | ~6.3% | ✅ measured |
| 2× BabyJubJub Schnorr verify (bindingSig + spendAuthSig) + depth-32 Poseidon tree insert + nullifier/cmx storage + events | ~5.0–5.3M | ~90% | ⚙️ estimated (= total op gas − `verifyAction`) |

> **Conclusion**: Unlike PrivacyBTC, pERC20 on-chain cost is **not dominated by ZK verification** (Groth16 is only ~3%), but by **two pure-Solidity BabyJubJub Schnorr scalar muls + 32-layer Poseidon Merkle insert**.
> - Groth16 compresses ZK verification from PrivacyBTC's 4.26M to 0.19M, but Schnorr + Poseidon-tree costs are similar on both sides, so per-tx total gas (~5.5M) is comparable to PrivacyBTC (~4.9M).
> - The largest further optimization is **offloading BabyJubJub point ops / Poseidon hashing to precompiles or highly optimized libraries**.

### 1.3 Full Flow Cost Summary (mint → transfer → burn)

| Step | Gas | Ethereum mainnet cost | L2 cost |
|------|-----|----------------------|---------|
| `mint()` | 5,683,559 | ~$273 | ~$0.055 |
| `transfer()` | 5,472,352 | ~$263 | ~$0.053 |
| `burn()` | 4,961,594 | ~$238 | ~$0.048 |
| **Full flow total** | **16,117,505** | **~$774** | **~$0.155** |

> On L2 (Arbitrum / Base, 0.04 gwei), a full mint→transfer→burn flow ≈ **$0.16**; Ethereum mainnet ≈ **$774**.

---

## 2. Calldata Size

> Groth16 proofs are only 256 bytes, so pERC20 calldata is **no longer proof-dominated**; the largest single field is the 580-byte `encCiphertext` (recipient-encrypted note).

### 2.1 `BundleAction` Fields & Calldata Breakdown

`transfer(PrivacyCall)` / `mint(uint256, PrivacyCall)` / `burn(uint256, PrivacyCall)` contain `BundleAction[]` (per-action struct per `IEndpointCore.BundleAction`):

| Field | Raw size | ABI-encoded size | Notes |
|-------|----------|------------------|-------|
| `cmx` | 32 B | 32 B | New note commitment |
| `encCiphertext` | 580 B | 640 B | 32 B length + 580 B + 28 B padding (**largest field**) |
| `outCiphertext` | 80 B | 128 B | 32 B length + 80 B + 16 B padding |
| `epk` | 32 B | 32 B | Ephemeral public key |
| `nfOld` | 32 B | 32 B | Nullifier |
| `anchor` | 32 B | 32 B | Historical Merkle root |
| `proof` | 256 B | 288 B | **Groth16 = `abi.encode(pA[2], pB[2][2], pC[2])`**; 32 B length + 256 B |
| `pubFields` | 256 B | 256 B | `uint256[8]` (anchor/cv/nf/rk/cmx/rt_frozen) |
| `spendAuthSig` | 96 B | 96 B | `uint256[3]` (BJJ Schnorr R.x, R.y, s) |
| ABI dynamic offsets + array/struct headers | — | ~300 B | 3 dynamic bytes offsets + array/tuple wrapper |
| `bindingSig` (`PrivacyCall` layer) | 96 B | 96 B | `uint256[3]` |
| Function selector (+ `amount`) | — | 4 B (mint/burn +32 B) | |
| **Single action total** | | **≈ 1,900–1,960 B (≈ 1.9 KB)** | |

> Compared to PrivacyBTC `transfer` ≈ **10,404 B**: pERC20 ≈ **1.9 KB**, about **5.4× smaller**, mainly because Groth16 proof (256 B) is far smaller than Halo2 proof (8,896 B).

### 2.2 Compression Feasibility

| Data segment | Size | Compressible? | Notes |
|--------------|------|---------------|-------|
| Groth16 proof | 256 B | ❌ No | BN254 curve points, near-maximum entropy |
| `encCiphertext` | 580 B | ❌ No | Encrypted note ciphertext |
| `outCiphertext` | 80 B | ❌ No | OVK ciphertext |
| Signatures (2 × 96 B) | 192 B | ❌ No | Random Schnorr scalars |
| Zero-padded high bits in `pubFields` | ~100 B | ✅ Yes | L2 zlib batch compression can eliminate |

**Potential optimization paths** (by ROI):

| Approach | Expected savings | Implementation difficulty |
|----------|-----------------|---------------------------|
| BN254 G1 point compression (proof 64B→32B per point) | ~96 B | Medium: verifier needs decompression |
| Multi-action batched submit (amortize ABI header + shared anchor) | ~300 B header per extra action | Low |
| `encCiphertext` via events instead of calldata (partially via `NoteAdded` already) | ~640 B | Medium |
| EIP-4844 blobs (ciphertext / proof in blob) | Large L1 fee reduction | High: architectural refactor |

---

## 3. Contract Deployment Cost

### 3.1 Main Contracts (runtime size + deployment gas)

| Contract | Runtime size (B) | Init size (B) | Deployment gas | Ethereum mainnet cost | L2 cost |
|----------|-----------------|--------------|----------------|----------------------|---------|
| `PERC20` (standalone) | **24,131** | 25,859 | 5,547,484 | **0.111 ETH ≈ $266** | **≈ $0.053** |
| `ActionGroth16Verifier` (includes internal `Groth16PairingVerifier`) | 12,798 | 14,054 | 3,093,049 | **0.062 ETH ≈ $148** | **≈ $0.030** |
| `Groth16PairingVerifier` (snarkjs-generated, embedded VK) | 1,106 | 1,132 | (deployed with verifier) | — | — |
| `PERC20Factory` (embeds `PERC20` impl for cloning) | 1,617 | 27,566 | 5,938,720 | **0.119 ETH ≈ $285** | **≈ $0.057** |
| **Full stack deploy (verifier + factory)** | | | **~9,031,769** | **≈ 0.181 ETH ≈ $433** | **≈ $0.087** |

> After deployment, each new private asset pool requires only one **EIP-1167 clone** (`createPerc20`, ~321K gas ≈ $15 mainnet / $0.003 L2); no need to redeploy verifier or implementation contracts.

### 3.2 Current Test Environment Deployment Addresses

> Test environment (`13.193.190.113`) runs the same pERC20 stack on 4 chains. Token = **PTEST ("PrivacyTest", decimals = 6)**, per-pool asset cap `assetCap = 10,000,000 PTEST`.

| Chain (chainId) | `PERC20Factory` | Official Pool (= token address) | deployBlock |
|-----------------|-----------------|--------------------------------|-------------|
| Ethereum (1) | `0x40EA5172872d678A1cBa7de054b72FFaCAAda66B` | `0x1F0899de2d0a4dcBbd8542B58d5F51d0448F2CC9` | 25,334,294 |
| Base (8453) | `0xfcC286D09B4821402442eD06C63904b6D49594D3` | `0xbbBD0c3e6A42a2EC40c6da6bb17C808440795AAF` | 47,437,899 |
| Arbitrum (42161) | `0xfcC286D09B4821402442eD06C63904b6D49594D3` | `0xbbBD0c3e6A42a2EC40c6da6bb17C808440795AAF` | 474,298,423 |
| Monad (143) | `0xb09CA46E01b8aBB63078165411ee7bF51C15A67E` | `0xf0Fbb8955a33d689414437d7099EAd6bf05c8172` | 81,790,318 |

### 3.3 EIP-170 / EIP-3860 Compliance

| Contract | Runtime (B) | EIP-170 limit | Headroom | Status |
|----------|------------|---------------|----------|--------|
| `PERC20` | 24,131 | 24,576 | **445 B** | ⚠️ Near limit (split libraries if logic grows) |
| `ActionGroth16Verifier` | 12,798 | 24,576 | 11,778 B | ✅ |
| Other contracts | < 1,700 | 24,576 | — | ✅ |

> A monolithic Groth16 verifier is only ~12.8 KB (no PrivacyBTC-style 40-step split verifier) — another engineering advantage of Groth16 over Halo2: **one constant-size contract** suffices for verification.

---

## 4. Circuit Specification

### 4.1 Circuit Parameters

| Parameter | Value |
|-----------|-------|
| Proof system | **Groth16** (Circom 2.x → snarkjs / rapidsnark) |
| Commitment / polynomial commitment | Groth16 (QAP + BN254 pairing) |
| Curve | BN254 (alt_bn128) |
| Scalar field | BN254 Fr (254 bit) |
| Powers of Tau | `pot18` (2¹⁸ = 262,144 elements; 131,787 constraints < 2¹⁸) |
| Public inputs (circuit layer) | **1** (`pub_hash`) |
| On-chain public fields | 8 (compressed to 1 `pub_hash` on-chain via Poseidon, see §4.3) |

### 4.2 Constraint System Statistics (`snarkjs r1cs info`, `circuits/action.circom`)

| Metric | Value |
|--------|-------|
| Constraints (# of Constraints) | **131,787** |
| Wires | 131,812 |
| Private inputs (# of Private Inputs) | 2,361 |
| Public inputs (# of Public Inputs) | 1 |
| Labels | 251,204 |
| `action_final.zkey` size | ~65 MB (68,141,043 B) |
| `action.r1cs` size | ~20 MB (21,292,440 B) |
| `pot18_final.ptau` size | ~288 MB (302,072,984 B) |

**Sub-circuit composition (`Action()` template)**:

| Sub-circuit | Role |
|-------------|------|
| `NoteCommitOld` / `NoteCommitNew` | Old / new note Pedersen-style commitment (incl. 64-bit value range) |
| `Merkle32` (depth 32) | Merkle membership proof for old note commitment (binds `anchor`) |
| `NullifierOld` | Nullifier from `nk, rho, psi, cm_old` (double-spend prevention) |
| `ValueCommit` | Net value commitment `cv_net` (binds binding signature conservation) |
| `SpendAuthRk` | Randomized auth public key `rk = (ask+alpha)·G_SPEND_AUTH` |
| `CommitIvkPkD` | `ivk` commitment, binds `ak, nk, pk_d` |
| `FrozenCmxNonMember` (IMT depth 20) | Compliance blacklist **non-membership** proof (consumed note `cm_old` not in freeze set) |
| 5 × `BJJSubgroupCheck` | Prime-order subgroup check on all external witness curve points (`g_d`, `pk_d`, `ak`), anti key-malleability |
| `PubHashAction` | Poseidon-compress 8 public fields into single `pub_hash` |

### 4.3 Public Field Definition (on-chain `pubFields[8]` → 1 `pub_hash`)

| Index | Name | Description |
|-------|------|-------------|
| 0 | `anchor` | Merkle tree root (historical validity) |
| 1 | `cv_net_x` | Net value commitment X |
| 2 | `cv_net_y` | Net value commitment Y |
| 3 | `nf_old` | Old note nullifier (dummy on mint) |
| 4 | `rk_x` | Randomized auth public key X (for spendAuthSig verify) |
| 5 | `rk_y` | Randomized auth public key Y |
| 6 | `cmx` | New note commitment (Merkle insert) |
| 7 | `rt_frozen` | Compliance freeze-set SMT root (must == `IPERC20.cmxFrozenRoot()`) |

> On-chain, `ActionGroth16Verifier.verifyAction` recomputes the 8 fields into `pub_hash` via `ActionPubHash.hash(...)`, then runs Groth16 pairing verification. The contract checks each field `< FIELD_MODULUS` to prevent double-spend attacks with same `pub_hash` but different `isSpent` keys (e.g. `nf + p`).

### 4.4 Proof Size

| Item | Size |
|------|------|
| Groth16 proof (`pA[2]` + `pB[2][2]` + `pC[2]`) | **256 bytes** (8 × 32 B field elements) |
| On-chain public input | 1 × 32 B (`pub_hash`; 8 raw fields passed via calldata then compressed) |
| `proof.json` (snarkjs text format, with brackets/commas) | 709 bytes |

---

## 5. Proof Generation Time

### 5.1 Measurement Results (test environment `13.193.190.113`, 8 vCPU Xeon 8124M @ 3.0 GHz)

> **Method**: Inside the `privacy-prover` container, generate witness from real circuit input `action_shield.json` (2,361 private inputs), then prove with **rapidsnark** (production path) and **snarkjs** (JS fallback), millisecond timing.

| Phase | Time (controlled, single-process exclusive) | Notes |
|-------|----------------------------------------------|-------|
| Witness generation (`generate_witness.js`, wasm) | **~0.40 s** (379 / 400 / 406 ms) | snarkjs WASM witness calculator |
| **rapidsnark proving (production path)** | **cold start 1.36 s; steady ~1.02 s** (1007 / 1020 / 1022 / 1035 ms) | C++ multi-thread MSM + NTT, saturates 8 vCPU |
| snarkjs proving (pure JS fallback) | **7.34 s** | ~7× slower than rapidsnark |
| On-chain Groth16 verification | ~188K gas (see §1.2) | 2× `ecPairing` + `ecMul` |
| **Single action total (witness + rapidsnark)** | **≈ 1.4 s** | |

### 5.2 Production Load Observations

> `privacy-prover` service logs (real airdrop mint, 8 vCPU with concurrent web / store-service / per-chain indexer·relayer·prover-service, CPU contention):

| Phase | Observed |
|-------|----------|
| Prelim witness (nf_old extraction) | 0.45 s |
| Final witness build | 0.42 s |
| **Groth16 proof (rapidsnark, incl. subprocess spawn + IO)** | **2.13 s / 2.20 s (256 bytes)** |

> Controlled exclusive run: rapidsnark ~1.0 s; under production concurrency, CPU contention + subprocess overhead raises this to ~2.1 s. One `transfer` / airdrop claim = **2 actions = 2 proofs**: controlled ≈ 2.8 s, production concurrent ≈ 4.3–4.5 s.

### 5.3 Optimization Directions

| Optimization | Expected rapidsnark time | Notes |
|--------------|------------------------|-------|
| Current (8 vCPU CPU, rapidsnark) | ~1.0 s (exclusive) / ~2.1 s (concurrent) | Baseline |
| Dedicated prover machine / isolated prover node | ~1.0 s (eliminate CPU contention) | Detach prover from app nodes |
| Higher clock / more cores | 0.5–0.8 s | MSM/NTT is strongly CPU-bound, near-linear gains |
| GPU proving (rapidsnark-cuda, etc.) | 0.2–0.5 s | Significant MSM acceleration on GPU |
| Native witness generation (C++ witnessgen) | ~0.1 s (vs WASM 0.4 s) | Remove WASM interpreter overhead |

---

## 6. Security Model

### 6.1 Trust Assumptions

| Component | Trust level | Notes |
|-----------|-------------|-------|
| Groth16 proof | Trustless (full on-chain verification) | Depends on **per-circuit trusted setup** (below) |
| Groth16 trusted setup (`pot18` + phase-2) | **Setup must be trusted** | Secure if ≥1 honest ceremony participant; toxic waste destroyed → proofs unforgeable. Main trust cost of Groth16 vs Halo2/KZG (universal / no circuit-specific setup) |
| `bindingSig` (BabyJubJub Schnorr) | Trustless | On-chain verify, prevents value-balance forgery |
| `spendAuthSig` (BabyJubJub Schnorr) | Trustless | On-chain verify, prevents relayer tampering with `encCiphertext` / `outCiphertext` |
| Relayer (tx submission) | **Minimal trust** | Cannot tamper encrypted content (spendAuthSig protects); can censor / reorder |
| Issuer / compliance officer (= `admin`) | Trusted | Holds `mint`, `setFrozenRoot` (blacklist), `setGroth16Verifier`; two-step `transferAdmin`/`acceptAdmin` |
| Public field canonicality | Trustless | Contract checks each field `< FIELD_MODULUS`, prevents same-`pub_hash` double-spend |

### 6.2 spendAuthSig Design

```
sighash = keccak256(
  "SpendAuth..." domain separator
  ‖ chainId       (uint256)
  ‖ contractAddr  (address)
  ‖ nfOld         (bytes32)
  ‖ cmx           (bytes32)
  ‖ epk           (bytes32)
  ‖ encCiphertext (580 B)
  ‖ outCiphertext (80 B)
)

Signer public key: rk = (ask + alpha) · G_SPEND_AUTH  (= pubFields[4,5], constrained by Groth16 circuit)
Scheme:            BabyJubJub Schnorr   s·G == R + e·rk
```

### 6.3 Compliance (Freeze) Model

- Each asset pool has an independent `cmxFrozenRoot` (compliance blacklist Indexed Merkle Tree, depth 20).
- Each action's `pubFields[7]` must equal on-chain `cmxFrozenRoot()`, else revert `BadFrozenRoot`.
- Circuit proves consumed note `cm_old` is **non-member** of freeze set (binds `c1.cm_old_x`, not output `cmx`, preventing blacklist bypass via fresh randomness — see `action.circom` audit note H-01).
- L-04 grace period: `setFrozenRootGracePeriod` (≤ 1 day) allows in-flight proofs to use the previous root, avoiding instant invalidation on root update.

> ⚠️ **Test environment warning**: `13.193.190.113` `generated.env` uses the same plaintext dev private key across chains for airdrop minting — **testing only, never for production**.

---

## 7. Build / Deploy Workflow

```bash
# 1. Compile circuit + one-time trusted setup (outputs action_final.zkey / wasm / verifier)
cd circuits
npm install
# circom compile → r1cs + wasm; snarkjs groth16 setup (pot18) → zkey; export Groth16PairingVerifier.sol

# 2. Inspect constraint size / artifacts
snarkjs r1cs info build/action/action.r1cs

# 3. Compile contracts + size check (PERC20 runtime should be < 24,576 B)
forge build --sizes | grep -E "PERC20|Groth16|OrchardVerifier"

# 4. Run end-to-end real-proof tests (with gas report)
forge test --gas-report --match-contract PERC20E2ETest

# 5. Deploy (verifier + factory), then clone asset pools per chain
#    See docs/DEPLOY_AWS_EC2.md and deploy-all.sh (contract deploy + asset mint + service stack, idempotent)
bash scripts/deploy-all.sh

# 6. Test-environment proving benchmark (rapidsnark)
docker exec perc20-privacy-prover-1 sh -c '
  cd /circuits/build/action
  node action_js/generate_witness.js action_js/action.wasm input.json w.wtns
  /opt/rapidsnark/bin/prover action_final.zkey w.wtns proof.json public.json'
```

---

## 8. Revision History

| Date | Version | Changes |
|------|---------|---------|
| 2026-06-23 | **v1.0** | Initial pERC20 performance benchmark: Circom/Groth16 (131,787 constraints, BN254, pot18); on-chain gas (mint 5.68M / transfer 5.47M / burn 4.96M, Groth16 verify only 188K), calldata (~1.9 KB), deployment costs and test-environment addresses (ETH/Base/Arbitrum/Monad, PTEST), test-environment (8 vCPU Xeon) proving measurements (rapidsnark exclusive ~1.0 s / concurrent ~2.1 s, snarkjs ~7.3 s). |
