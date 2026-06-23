# Multi-chain deployment

pERC20 runs on **Ethereum, Base, Arbitrum, and Monad** with identical bytecode per chain.

## Why contracts don't change

`OrchardVerifier` binds signatures to `block.chainid`. Independent deployments are automatically isolated.

**Client rule**: proof `chain_id` MUST equal the pool's chain, not the wallet's current network.

## Deployment model

| Component | Per chain | Shared |
|-----------|-----------|--------|
| Groth16 verifier | ✓ | — |
| PERC20Factory | ✓ | — |
| Asset pool clone | ✓ per token | — |
| privacy-prover | optional | ✓ |
| store-service | — | ✓ coordinator |

## Frontend

- `apps/web/src/config/chains.ts`
- Env: `VITE_CHAINS` JSON (indexer + relayer URLs per chain)

## Airdrop dedup

One claim per wallet **per chain**: key = `(wallet_address, chain_id)`.

## Test addresses

See [Performance benchmarks](performance.md) and `docs/perc20-performance-benchmarks-en.md` §3.2.

## Source

Full backend integration: `docs/multichain.md`

## Next

- [pERC20 issuance](../privacy-core/perc20/issuance-and-deployment.md)
- [Home](../README.md)
