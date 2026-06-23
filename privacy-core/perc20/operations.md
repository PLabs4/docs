# pERC20 · Operations

## transfer

```solidity
function transfer(PrivacyCall calldata call) external returns (bool);
```

- `valueBalance = 0`
- Parties and amount **private**
- Emits `NoteAdded` / `NoteConfirmed`, not `Transfer`

## mint (issuer only)

```solidity
function mint(uint256 amount, PrivacyCall calldata call) external;
```

- `totalSupply += amount`
- `valueBalance = amount | (1 << 255)`
- **Amount public**, recipient **private**

## burn

```solidity
function burn(uint256 amount, PrivacyCall calldata call) external;
```

- `totalSupply -= amount`
- **Amount public**, burner **private**

## valueBalance encoding

| Operation | valueBalance |
|-----------|--------------|
| transfer | `0` |
| burn | bit255=0, low bits=amount |
| mint | bit255=1, low bits=amount |

## Gas (single action, measured)

| Op | Gas |
|----|-----|
| mint | ~5.68M |
| transfer | ~5.47M |
| burn | ~4.96M |

→ [Performance benchmarks](../../ecosystem/performance.md)

## Call chain

```
IPERC20.mint/burn/transfer → PERC20._executeBundle → OrchardVerifier
```

## Next

- [Approved spending](approved-spending.md)
