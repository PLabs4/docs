# Off-chain services

pERC20 uses the shared privacy off-chain infrastructure.

## Architecture

```
Wallet / dApp
    │ witness + PrivacyCall
    ▼
privacy-prover ──► Groth16 (rapidsnark ~1s)
    │
privacy-indexer ◄── NoteAdded / tree index
    │
privacy-relayer ──► submit tx (hide submitter EOA)
    │
PERC20 pool (OrchardVerifier on-chain)
```

## Services

| Service | Role | pERC20 |
|---------|------|--------|
| **privacy-prover** | witness + Groth16 | action circuit |
| **privacy-indexer** | event index | ✅ |
| **privacy-relayer** | broadcast txs | ✅ |
| **store-service** | airdrop dedup | ✅ |

## Wallet responsibilities

See [pERC20 approved spending](../privacy-core/perc20/approved-spending.md) and `apps/web/`.

- Note scan, trial-decrypt
- ZIP-32 subaccounts (approve)

## E2E

```bash
# Requires sibling repos: privacy-indexer, privacy-relayer
e2e/scripts/run_full_e2e.sh
```

## Next

- [Developer guide](../guides/developer-guide.md)
- [Multi-chain deployment](multichain.md)
