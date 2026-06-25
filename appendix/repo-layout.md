# Repository layout

## GitBook ↔ source map

```
gitbook/
├── privacy-core/          Privacy Core
│   ├── (crypto)           Orchard, circuits, freeze, …
│   ├── perc20/            pERC20 asset protocol
│   └── erc20-shield/      Shield ERC20 asset protocol
├── applications/          Operations on privacy assets
│   ├── private-swap/      pSWAP (SwapCoordinator design)
│   ├── private-dex/       pDEX (coming soon)
│   └── private-x402/      pX402 (coming soon)
├── ecosystem/             Multi-chain, performance, services
├── guides/                How-to guides
└── appendix/
```

| GitBook | Source |
|---------|--------|
| `privacy-core/*` (crypto) | `contracts/orchardverifier/`, `circuits/action.circom` |
| `privacy-core/perc20/*` | `contracts/ptoken/`, ERC submissions |
| `privacy-core/perc20/eip-8302.md` | [PR #1817](https://github.com/ethereum/ERCs/pull/1817) `ERCS/erc-8302.md` |
| `privacy-core/perc20/eip-8289.md` | Prior draft number; repo `docs/eip-draft.md` |
| `privacy-core/erc20-shield/*` | Shield ERC20 protocol documentation |
| `applications/private-swap/*` | Target: `SwapCoordinator` |
| `applications/private-dex/*` | Coming soon |
| `applications/private-x402/*` | Coming soon |
| `privacy-core/compliance-freeze` | `circuits/frozen_cmx_nonmember.circom` |
| `ecosystem/performance` | Canonical benchmark doc (was `docs/perc20-performance-benchmarks-en.md`) |

## Core repo tree

```
PERC20/
├── contracts/orchardverifier/   Privacy Core (crypto)
├── contracts/ptoken/              pERC20
├── circuits/                      action + ceremony
├── crates/privacy-prover/
├── apps/web/
└── docs/
```

## Local preview

```bash
npm install -g honkit
cd gitbook && honkit serve
# http://localhost:4000
```

## Chapters to expand

| Priority | Chapter | Source |
|----------|---------|--------|
| P0 | `privacy-core/perc20/approved-spending` | `docs/zip-32-approve-extend.md` |
| P1 | `privacy-core/erc20-shield/*` | Expand implementation and source mapping |
| P0 | `applications/private-swap/*` | Implement `SwapCoordinator` per design spec |
| P1 | `applications/private-dex/*` | Coming soon |
| P1 | `applications/private-x402/*` | Coming soon |
| P2 | Security / audit chapter | `docs/audit-*.md` |
