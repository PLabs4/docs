# Shield ERC20 overview

> **Status: coming soon** — documentation and reference implementation are not yet published.

**Shield ERC20** privatizes **existing public ERC-20** tokens. It is an **asset protocol** under Privacy Core (with [pERC20](../perc20/overview.md)), not a swap/payment application.

## Intended role

| Direction | User action | On-chain effect |
|-----------|-------------|-----------------|
| Shield | Deposit public ERC-20 | Mint / credit private notes |
| Unshield | Burn private notes | Release public ERC-20 to recipient |

This complements **natively private pERC20 issuance**: issuers can launch private-from-creation tokens today; Shield ERC20 lets users **privatize existing public ERC-20** without leaving the EVM asset ecosystem.

## Stack position

```
Shield ERC20  →  deposit / withdraw rules, ERC-20 custody
       │
pERC20 / OrchardVerifier  →  shielded note state machine
```

Built on the same [Privacy Core](../overview.md) primitives as [pERC20](../perc20/overview.md).

## Related

- [pERC20](../perc20/overview.md) — private fungible token (shipped)
- [EIP-8182](https://eips.ethereum.org/EIPS/eip-8182) — protocol-level shielded pool (complementary design space)
