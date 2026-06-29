# Shield ERC20 · Deployment

## One-time infra (per chain)

Reuse the same Groth16 verifier as pERC20 — **no new circuit or trusted setup**.

| Contract | Role |
|----------|------|
| `ActionGroth16Verifier` | Shared with pERC20 (reuse existing deploy) |
| `ERC20ShieldFactory` | EIP-1167 clones + `(underlying, scale)` registry |

Deploy `ERC20ShieldFactory(groth16VerifierAddress)` once per chain. The factory deploys a locked `ERC20Shield` implementation internally.

## Create a shield pool

A **shield pool** is one `ERC20Shield` instance — custody for a single `(underlying, scale)` pair. Value enters only via `shield` and exits via `unshield`.

```solidity
address pool = factory.createShieldPool(
    "Shield USDC",    // name  — metadata only
    "sUSDC",          // symbol
    6,                // decimals (display; suggest matching underlying)
    usdcAddress,      // underlying ERC-20
    0                 // scale: 0 → use suggestedScale(underlying.decimals())
);
```

- **Permissionless** — anyone may deploy for any ERC-20
- **One pool per `(underlying, scale)`** — duplicate reverts `PoolExists(existing)`
- Deployer becomes **`admin`** (compliance: `setFrozenRoot`, `setGroth16Verifier`)
- Emits `ShieldPoolDeployed` and pool-local `ShieldPoolCreated`

### Deterministic address (CREATE2)

```solidity
address pool = factory.createShieldPoolDeterministic(
    name, symbol, decimals, underlying, scale, salt
);
address predicted = factory.predictShieldPoolAddress(creator, salt);
```

Salt is mixed with `msg.sender`: `keccak256(abi.encode(creator, salt))`.

## Scale selection

Note values are constrained to **64 bits** in the action circuit. `scale` maps note units to underlying wei:

```
weiAmount = amountUnits * scale
```

### `suggestedScale(tokenDecimals)`

| Underlying decimals | Default `scale` | Note unit granularity |
|--------------------|-----------------|------------------------|
| ≤ 8 | `1` | 1 wei |
| > 8 | `10^(decimals − 8)` | `10⁻⁸` of one whole token |

Example: 18-decimal token → `scale = 1e10` → one note unit = `1e-8` token.

Pass `scale_ = 0` to `createShieldPool` to auto-resolve via `suggestedScale`. Custom scales are allowed but **`scale` is fixed at init** — choose carefully before deployment.

Constraints enforced on-chain:

- `amountUnits < SUBGROUP_ORDER` (same as pERC20 mint/burn)
- Dust `< scale` wei cannot be represented as a note unit

## Registry

| Mapping | Purpose |
|---------|---------|
| `isShieldPool[pool]` | `true` for factory-deployed shield pools |
| `poolFor[keccak256(underlying, scale)]` | First shield pool for each pair |

Wallets and coordinators should verify `isShieldPool(pool)` before treating an address as a legitimate shield pool.

## Admin roles

| Role | Powers | Notes |
|------|--------|-------|
| `admin` (deployer by default) | `setFrozenRoot`, `setGroth16Verifier`, admin transfer | **No mint** — cannot create unbacked supply |
| `issuer()` view | Returns `admin` | IPERC20 compatibility only |

For shield pools, **`setGroth16Verifier` is a high-trust action** — a malicious verifier could authorize forged `unshield` proofs and drain custody. Same trust model as pERC20, with real ERC-20 at stake.

## Clone initialization

EIP-1167 clones call `initialize(...)` atomically in the factory transaction — no init front-running window. The locked implementation is fully initialized at factory deploy and cannot be hijacked.

## Multi-chain

Same bytecode per chain; binding signatures include `chainId` and `address(this)`. Each chain needs its own factory + per-asset pool clones.

→ [Multi-chain deployment](../../ecosystem/multichain.md)

## Tools

| Tool | Path |
|------|------|
| Deploy scripts | `script/DeployOfficial.s.sol`, `script/deploy-all-chains.sh` |
| Tests | `test/ERC20ShieldTest.t.sol` |
| Design notes | `docs/shield-unshield-and-swap.md` |

## Next

- [Shield ERC20 overview](overview.md)
- [pERC20 issuance](../perc20/issuance-and-deployment.md) (shared verifier infra)
- [Shield pool 命名变更说明](../../docs/shield-pool-naming-migration.md)
