# Shield ERC20 · Operations

Official infrastructure SHOULD screen the **depositor** before building shield calldata. See [Compliance implementation](../compliance-implementation.md).

## shield (permissionless deposit)

```solidity
function shield(uint256 amountUnits, PrivacyCall calldata call) external;
```

Deposits `amountUnits * scale` wei of underlying from `msg.sender` and creates a shielded note of `amountUnits`.

| Property | Value |
|----------|-------|
| Permission | Anyone |
| `valueBalance` | `amountUnits \| (1 << 255)` (same encoding as pERC20 `mint`) |
| `recipientMeta` | `bytes32(0)` — recipient is the output note |
| `executor` | `address(0)` |
| Public accounting | `shieldedSupply += amountUnits` |
| Event | `Shielded(depositor, amountUnits, weiAmount)` |

### Preconditions

- `amountUnits != 0` and `amountUnits < SUBGROUP_ORDER`
- Caller has approved at least `amountUnits * scale` on the underlying token
- Underlying transfer increases pool balance by **exactly** `weiAmount` (rejects non-standard tokens)

### Proof bundle

- Typically a **single output action** with a DUMMY input (`nfOld` unique, non-zero) — same witness shape as pERC20 `mint`
- Binding signature binds `valueBalance = amountUnits | (1 << 255)`

### Call chain

```
shield → safeTransferFrom → _executeBundle(vb, executor=0) → shieldedSupply +=
```

---

Official infrastructure SHOULD screen the **recipient** before submitting unshield transactions. See [Compliance implementation](../compliance-implementation.md).

## unshield (withdraw to public address)

```solidity
function unshield(uint256 amountUnits, address recipient, PrivacyCall calldata call) external;
```

Spends shielded notes worth `amountUnits` and releases `amountUnits * scale` wei of underlying to `recipient`.

| Property | Value |
|----------|-------|
| Permission | Note holder (via spend-auth sig) |
| `valueBalance` | `amountUnits` (bit 255 = 0; same as pERC20 `burn`) |
| `recipientMeta` | `bytes32(uint256(uint160(recipient)))` — **bound in binding sighash** |
| `executor` | `address(0)` |
| Public accounting | `shieldedSupply -= amountUnits` (before transfer) |
| Event | `Unshielded(recipient, amountUnits, weiAmount)` |

### Recipient binding

`recipient` is hashed into the binding sighash as `recipientMeta`. A relayer submitting the transaction **cannot redirect** the ERC-20 payout — the proof must match the declared recipient.

### Partial unshield

Bundle may include a **change output note** back to the user (multi-action bundle), same as partial burn on pERC20.

### CEI + reentrancy

1. `_executeBundle` (verify proofs, spend nullifiers, insert outputs)
2. `shieldedSupply -= amountUnits`
3. `safeTransfer(underlying, recipient, weiAmount)`

Both `shield` and `unshield` use `nonReentrant`.

---

## transfer (private note→note)

```solidity
function transfer(PrivacyCall calldata call) external returns (bool);
function transfer(address executor, PrivacyCall calldata call) external returns (bool);
```

Identical to [pERC20 transfer](../perc20/operations.md):

| Property | Value |
|----------|-------|
| `valueBalance` | `0` |
| Custody | Unchanged — no underlying movement |
| `shieldedSupply` | Unchanged |
| Executor overload | When `executor != address(0)`, only that address may submit (atomic swap legs) |

---

## Disabled operations

```solidity
function mint(uint256, PrivacyCall calldata) external pure { revert MintDisabled(); }
function burn(uint256, PrivacyCall calldata) external pure { revert BurnDisabled(); }
```

Unbacked issuance would break the custody invariant. Use `shield` / `unshield` instead.

---

## valueBalance encoding

| Operation | valueBalance | Underlying movement |
|-----------|--------------|---------------------|
| `transfer` | `0` | none |
| `unshield` | bit255=0, low bits=`amountUnits` | pool → `recipient` |
| `shield` | bit255=1, low bits=`amountUnits` | `msg.sender` → pool |

Same sign-bit scheme as pERC20 — see [Signatures](../signatures.md) and [pERC20 operations](../perc20/operations.md).

---

## Error reference

| Error | When |
|-------|------|
| `MintDisabled` / `BurnDisabled` | Calling disabled `mint`/`burn` |
| `ZeroAmount` | `amountUnits == 0` |
| `AmountTooLarge` | `amountUnits >= SUBGROUP_ORDER` |
| `ZeroRecipient` | `unshield` to `address(0)` |
| `SupplyUnderflow` | `shieldedSupply < amountUnits` on unshield |
| `NonStandardToken` | Received underlying ≠ expected on shield |
| `Reentrancy` | Reentrant call during shield/unshield |
| `UnauthorizedExecutor` | Swap leg submitted by wrong executor |

---

## Call chain summary

```
IPERC20.transfer        → _executeBundle(vb=0)           → OrchardVerifier
IERC20Shield.shield     → pull ERC20 + _executeBundle     → shieldedSupply +=
IERC20Shield.unshield   → _executeBundle + transfer ERC20 → shieldedSupply -=
```

## Next

- [Compliance implementation](../compliance-implementation.md)
- [Deployment & factory](deployment.md)
- [pERC20 operations](../perc20/operations.md) (shared transfer semantics)
- [Universal call format](../universal-call-format.md)
