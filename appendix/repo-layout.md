# Repository layout

## GitBook ↔ source map

```
gitbook/
├── privacy-core/          Privacy Core
│   ├── (crypto)           Orchard, circuits, freeze, …
│   ├── perc20/            pERC20 asset protocol
│   └── erc20-shield/      Shield ERC20 (coming soon)
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
| `privacy-core/erc20-shield/*` | Coming soon |
| `privacy-core/compliance-freeze` | `circuits/frozen_cmx_nonmember.circom` |
| `ecosystem/performance` | `docs/perc20-performance-benchmarks-en.md` |

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
| P1 | `privacy-core/erc20-shield/*` | Coming soon |
| P2 | Security / audit chapter | `docs/audit-*.md` |
