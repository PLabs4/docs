# Compliance implementation

Privacy Core uses a **two-layer compliance model**:

1. **Entry / exit screening** — off-chain address checks at official infrastructure boundaries (shield, unshield, and issuer paths where applicable).
2. **Post-hoc fund freeze** — on-chain note freeze via `cmxFrozenRoot` after a `cmx` is identified.

Smart contracts remain **permissionless** (same pattern as Aave, Morpho, Uniswap): compliance is enforced on **official channels** (relayer, web app) plus **in-circuit spend rules** for frozen notes. There is **no on-chain sanctions oracle** in pool contracts.

```
                    Official channel (relayer + web app)
┌──────────────────────────────────────────────────────────────┐
│  L1  Entry / exit screening (off-chain)                      │
│      TRM + Chainalysis API → block sanctioned EVM addresses  │
└────────────────────────────┬─────────────────────────────────┘
                             │ pass
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  Pool contracts (permissionless on-chain)                    │
│  shield / unshield / transfer / mint / burn                  │
└────────────────────────────┬─────────────────────────────────┘
                             │ spend
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  L2  Post-hoc freeze (on-chain)                              │
│      frozenRoot — prove spent cm_old ∉ frozen IMT            │
└──────────────────────────────────────────────────────────────┘
```

Cryptographic details of L2 are in [Compliance freeze design](compliance-freeze.md). This page covers **operational implementation** of both layers.

---

## Layer 1 · Entry / exit screening (off-chain)

### Purpose

Block **sanctioned or otherwise prohibited EVM addresses** from using official infrastructure to move value **into or out of** a privacy pool, before a transaction is built or broadcast.

This is **not** enforced in `ERC20Shield` / `PERC20` bytecode. Users who bypass the official relayer or frontend can still call contracts directly — the same limitation accepted by major DeFi frontends (see [Industry precedent](#industry-precedent)).

### Scope — which address to check

| Boundary | On-chain actor | Address to screen | Who submits tx |
|----------|----------------|-------------------|----------------|
| **Shield ERC20 · `shield`** | `msg.sender` pays underlying | **Depositor** (connected wallet) | Depositor signs + sends |
| **Shield ERC20 · `unshield`** | `recipient` in calldata | **Recipient** (not relayer) | Relayer may broadcast; payout is proof-bound |
| **pERC20 · `mint`** (issuer) | Issuer workflow | **Beneficiary / depositor** per issuer policy | Varies |
| **pERC20 · `burn` / unshield-like exit** | Public amount, private holder | **Public recipient** if any; else attribution → L2 | Varies |
| **In-pool `transfer`** | No public address boundary | — | Optional relayer policy only |

For [Shield ERC20](erc20-shield/operations.md), the critical gates are **`shield` (depositor)** and **`unshield` (recipient)**. Recipient binding in the binding sighash prevents a relayer from redirecting unshield payouts; screening must target the **declared recipient**, not the relayer EOA.

### Data providers

Official infrastructure MAY integrate **both**:

| Provider | Role | Typical v1 rule |
|----------|------|-----------------|
| **Chainalysis** (Sanctions / KYT API) | Direct sanctions / SDN-aligned lists | Block if address is a sanctioned entity |
| **TRM Labs** (Wallet Screening API) | Sanctions + optional counterparty / indirect exposure | v1: **Ownership** (direct sanctions) only |

**Recommended v1 policy — union (strictest):** block if **either** provider flags the address for direct sanctions.

| Mode | When to use |
|------|-------------|
| **Union** | Either API → block (default for production official channel) |
| **Primary + fallback** | TRM primary; Chainalysis only when TRM is unavailable |
| **Layered** | Chainalysis hard-blocks sanctions; TRM adds counterparty / indirect rules in a later phase |

Counterparty and indirect TRM rules increase false positives (e.g. dust from sanctioned addresses). Enable only with volume / time thresholds after v1.

### Integration points

| Component | Enforcement | Required? |
|-----------|-------------|-----------|
| **privacy-relayer** | Reject `/wrapped/shield/calldata`, `/wrapped/unshield/submit`, and issuer-path submits when screening fails | **Yes** — hard gate |
| **Web app** (`apps/web`) | Pre-check on wallet connect, shield confirm, unshield recipient input | Recommended (UX) |
| **privacy-indexer** | No blocking; optional analytics for ops | Optional |

Relayer flow (wrapped pools):

```
shield:   POST /wrapped/shield/calldata  → screen(depositor) → return calldata or 403
unshield: POST /wrapped/unshield/submit  → screen(recipient) → broadcast or 403
```

Shield requests SHOULD include an explicit `depositor_evm` field so the relayer does not infer the payer from the bundle alone.

### Screening module behavior

```text
screenAddress(addr):
  1. Normalize checksummed lowercase EVM address
  2. Parallel: Chainalysis + TRM (with timeout)
  3. Merge per SCREENING_MODE (default: union on direct sanctions)
  4. fail-closed: if both providers error → reject (do not silently allow)
  5. Log: timestamp, address, provider hits, rule ids (audit trail)
  6. Cache results (e.g. 5–15 min) to limit API cost
```

**HTTP response on block:** `403` with a stable error code (e.g. `SANCTIONED_ADDRESS`) and no calldata / no broadcast.

**Environment variables (illustrative):**

| Variable | Purpose |
|----------|---------|
| `SCREENING_ENABLED` | Master switch |
| `SCREENING_MODE` | `union` \| `primary_fallback` \| `layered` |
| `CHAINALYSIS_API_KEY` | Chainalysis API |
| `TRM_API_KEY` | TRM Wallet Screening |
| `SCREENING_FAIL_CLOSED` | Default `true` |

### Operational / legal complement

Technical screening alone does not satisfy all regulatory expectations. Production deployments SHOULD also document:

- **Terms of use** — prohibited jurisdictions, sanctioned persons, no VPN circumvention
- **Geographic restrictions** on hosted interfaces where required (e.g. IP controls on web app)
- **Appeals process** for false positives
- **Retention policy** for screening logs

See [Off-chain services](../ecosystem/off-chain-services.md) for relayer placement in the stack.

### Limits of Layer 1

| Scenario | Layer 1 effect |
|----------|----------------|
| User uses official relayer + frontend | Blocked if address sanctioned |
| User calls `shield` / `unshield` directly on-chain | **Not blocked** by Layer 1 |
| Sanctioned underlying (e.g. frozen USDC) | **USDC `blacklist`** may still revert transfers independently |

Layer 1 is **reasonable efforts on official infrastructure**, aligned with industry practice — see [Industry precedent](#industry-precedent).

---

## Layer 2 · Post-hoc fund freeze (on-chain)

### Purpose

After investigation, **freeze specific notes** identified by commitment `cmx` so they **cannot be spent** inside the pool — including private `transfer`, swap legs, and unshield.

This is enforced **in the Groth16 circuit** and on-chain via `cmxFrozenRoot()`; it cannot be bypassed by choosing a different frontend.

### Mechanism (summary)

| Item | Detail |
|------|--------|
| Off-chain set | Sorted blacklist of `cmx` values |
| Structure | Depth-20 incremental Merkle tree (IMT) |
| On-chain | `cmxFrozenRoot()` per pool; admin `setFrozenRoot(newRoot)` |
| Circuit | `FrozenCmxNonMember` — spent **input** `cm_old` must not be in the frozen set |
| Public binding | `pubFields[7] == cmxFrozenRoot()` |

Full design: [Compliance freeze design](compliance-freeze.md).

### Admin workflow

```
1. Detect illicit activity (Chainalysis / TRM case, law enforcement, internal analytics)
2. Attribute flows to specific notes → recover cmx (indexer + trial decrypt where authorized)
3. Rebuild IMT off-chain with new cmx leaf(es)
4. Admin multisig: setFrozenRoot(newRoot) on affected pool(s)
5. Provers / wallets fetch new root for witnesses
6. Frozen notes: any spend attempt reverts (BadFrozenRoot / proof failure)
```

Grace window: implementations MAY accept the **previous** root briefly after an update so in-flight proofs are not stranded (see audit L-03 / L-04 in pool audits).

### What freeze does and does not do

| Does | Does not |
|------|----------|
| Prevent spending frozen **notes** in-pool | Freeze an EVM address at the token contract |
| Apply per pool (`PERC20`, each `ERC20Shield`) | Automatically screen new depositors |
| Hide blacklist contents (only root is public) | Recover underlying to treasury without a valid unshield path |

Unshield of a **non-frozen** note to a sanctioned address is a Layer 1 concern; spending a **frozen** note is blocked regardless of recipient.

### Trust trade-off

The compliance admin can expand the frozen set over time. Production SHOULD use **multisig + timelock** for `setFrozenRoot`, with documented governance.

---

## How the two layers work together

| Stage | Layer | Identifier | Bypass official channel? |
|-------|-------|------------|---------------------------|
| Deposit (shield) | L1 | Depositor EVM address | Yes — direct `shield` call |
| Withdraw (unshield) | L1 | Recipient EVM address | Yes — direct `unshield` call |
| In-pool activity | L2 only | Note `cmx` | No — circuit enforces freeze |
| After attribution | L2 | Note `cmx` | No |

Typical enforcement chain:

1. **L1** stops sanctioned addresses on the official relayer at pool boundaries.
2. If funds already entered (direct contract call or pre-sanction deposit), investigation maps value to **`cmx`**.
3. **L2** `setFrozenRoot` makes those notes unspendable on-chain.
4. Underlying asset issuers (e.g. USDC blacklist) may provide an **additional** freeze at the ERC-20 layer.

---

## Per-protocol checklist

### Shield ERC20 (`ERC20Shield`)

| Operation | L1 screen | L2 freeze |
|-----------|-----------|-----------|
| `shield` | Depositor | Output note `cmx` after attribution |
| `unshield` | Recipient | Spent input `cm_old` |
| `transfer` | — | Spent input `cm_old` |

### pERC20 (`PERC20`)

| Operation | L1 screen | L2 freeze |
|-----------|-----------|-----------|
| `mint` | Per issuer policy | Output `cmx` |
| `burn` | Public exit address if applicable | Spent `cm_old` |
| `transfer` | — | Spent `cm_old` |

### pSWAP / cross-chain apps

Each swap leg uses the same pool verifier — every action must satisfy **L2** non-membership for that leg’s pool. Entry screening applies at the **first shield / mint** into a leg’s asset.

---

## Implementation status

| Component | Layer 1 (screening) | Layer 2 (frozenRoot) |
|-----------|---------------------|----------------------|
| Pool contracts | Not in bytecode (by design) | ✅ Shipped (`cmxFrozenRoot`, circuit) |
| privacy-relayer | Planned / operator-configured | N/A |
| Web app | Planned / operator-configured | Prover uses on-chain root |
| Admin tooling | N/A | Off-chain IMT rebuild + `setFrozenRoot` |

---

## Industry precedent

Layer 1 follows the **dominant DeFi compliance pattern**: pool / router contracts stay **permissionless**; screening runs on **official frontends, indexers, or relayers** using third-party risk APIs (TRM, Chainalysis, or both). None of the protocols below embed sanctions checks in core lending or AMM bytecode.

| Protocol / product | What they screen | Provider(s) | Where enforced | Core contracts |
|--------------------|------------------|-------------|----------------|----------------|
| **[Aave](https://aave.com)** | Connected wallet before supply / borrow / etc. | [TRM Wallet Screening](https://www.trmlabs.com/resources/blog/how-defi-platforms-are-using-data-from-trm-labs-to-respond-to-tornado-cash-sanctions) | [Official interface](https://github.com/aave/interface/pull/1017) (2022+, post–Tornado Cash) | Permissionless — no `isSanctioned` in pool contracts |
| **[Uniswap](https://uniswap.org)** (Labs frontend) | Connected wallet | TRM Labs | [Uniswap Labs web app only](https://support.uniswap.org/hc/en-us/articles/8671777747597) | Uniswap v2/v3/v4 pools unchanged |
| **[Morpho](https://morpho.org)** | Wallet on official / curated interfaces | TRM (ecosystem standard) | [Morpho App](https://docs.morpho.org/get-started/resources/app-ecosystem); governance (e.g. MIP-96) requires OFAC screening on hosted frontends | Morpho core immutable; compliance at app layer |
| **[Balancer](https://balancer.fi)** | Connected wallet | TRM | Official frontend (2022, same wave as Aave) | Vault contracts permissionless |
| **[Oasis](https://oasis.app)** | Connected wallet | TRM | Official frontend (2022) | Maker / Oasis contracts permissionless |
| **[dYdX](https://dydx.exchange)** | Wallet + IP geolocation | TRM + WAF | [Official interface / indexer](https://help.dydx.exchange/en/articles/4798063-location-restrictions); v4 chain code neutral | Perps matching on-chain; blocks are at hosted layer |
| **[1inch](https://1inch.io)** | Wallet before swap routing | TRM (industry-standard integration) | 1inch frontend / API | Aggregation contracts permissionless |
| **[Railgun](https://railgun.org)** | Shield / unshield participants | Chainalysis and other [PPOI list providers](https://railgun.org) | Wallet SDK + relayer policy; **not** in Railgun smart contracts | Privacy pool contracts do not call sanctions oracles |
| **Tornado Cash** (historical, pre-shutdown) | Depositor / withdrawer | Chainalysis Sanctions Oracle | **Frontend only** — oracle was never enforced in the mixer contracts | Illustrates frontend-only limits of L1 |

**Common traits** these products share with Privacy Core L1:

| Trait | Industry | Privacy Core L1 |
|-------|----------|-----------------|
| Enforcement surface | Hosted UI, API, indexer, or relayer | `privacy-relayer` + `apps/web` |
| Contract layer | Permissionless pools / routers | Permissionless `ERC20Shield` / `PERC20` |
| Primary APIs | TRM and/or Chainalysis off-chain | TRM **and** Chainalysis (union on direct sanctions) |
| v1 rule focus | Direct sanctions (Ownership) | Same — Ownership / SDN-aligned hits |
| Bypass | Direct contract interaction | Direct `shield` / `unshield` call |
| Wider rules (optional) | TRM counterparty / indirect (Aave tuned post-2022) | Deferred to v2 with thresholds |

**Not the same as L1** (listed for contrast only):

| Approach | Example | Difference from L1 here |
|----------|---------|------------------------|
| Issuer ERC-20 blacklist | [USDC](https://www.circle.com/en/usdc) / USDT `blacklist()` | On-chain; freezes token transfers, not pool entry |
| Permissioned RWA pools | [Aave Horizon](https://aave.com/blog/horizon-launch) | KYC + issuer allowlist at token level |
| In-circuit withdrawal rules | [Privacy Pools](https://privacypools.com) ASP | Association sets bound in ZK withdrawal proof |
| On-chain sanctions oracle | Chainalysis `isSanctioned` in **business** contracts | Rare in lending/AMM cores; Privacy Core does **not** use this |

Privacy Core L1 is therefore **aligned with Aave / Uniswap / Morpho / dYdX** (off-chain API at official infrastructure), not with issuer blacklists or on-chain oracle hooks. Layer 2 (`frozenRoot`) adds an on-chain spend freeze that most of those protocols do **not** have — closer in spirit to Railgun post-hoc note policy + Privacy Pools association enforcement, but keyed on `cmx` in-circuit.

---

## Next

- [Compliance freeze design](compliance-freeze.md) — IMT, circuit, `pubFields[7]`
- [Shield ERC20 operations](erc20-shield/operations.md) — shield / unshield boundaries
- [Off-chain services](../ecosystem/off-chain-services.md) — relayer architecture
- [pERC20 overview](perc20/overview.md)
