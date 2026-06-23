# pERC20 · Issuance & deployment

## One-time infra (per chain)

| Contract | Role | Deploy gas (~) |
|----------|------|----------------|
| `ActionGroth16Verifier` | Groth16 verify | ~3.1M |
| `PERC20Factory` | EIP-1167 clones | ~5.9M |

Total ~9M gas (~$433 mainnet @ 20 gwei / ~$0.09 L2).

## Create asset

```solidity
factory.createPerc20("MyToken", "MTK", 6, issuerAddress);
```

~321K gas per pool. Emits `Perc20Created`.

## Issuer roles

| Role | Powers |
|------|--------|
| `issuer` | `mint` |
| `admin` (default = issuer) | `setFrozenRoot`, `setGroth16Verifier`, admin transfer |

## Multi-chain

Same bytecode per chain; signatures bind `chainId`.

→ [Multi-chain deployment](../../ecosystem/multichain.md)

## Tools

| Tool | Path |
|------|------|
| Deploy scripts | `script/DeployOfficial.s.sol`, `script/deploy-all-chains.sh` |
| Issuance UI | `apps/web/src/issuance/` |
| Issuer guide | [guides/issuer-guide.md](../../guides/issuer-guide.md) |

## Next

- [ERC-8302 index (approve extension)](eip-8302.md)
