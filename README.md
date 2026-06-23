# pERC20 Protocol Documentation

**Privacy Core** — cryptographic primitives and **private asset** protocols (pERC20, Shield ERC20).

```
┌─────────────────────────────────────────────────────────────┐
│  Privacy Core                                               │
│  ┌─ Cryptographic layer ─────────────────────────────────┐  │
│  │ Orchard notes · Groth16 · state machine · freeze    │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─ Asset protocols (note creation) ────────────────────┐  │
│  │ pERC20 (native private mint) · Shield ERC20 (public→private) │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Module map

| Module | Questions it answers | Audience |
|--------|---------------------|----------|
| **Privacy Core · crypto** | How are notes proved, stored, spent? Compliance freeze? | Crypto / contracts / audit |
| **Privacy Core · assets** | How do **private assets** get created? (mint, shield) | Issuers / integrators |

## Reading path

**First time**

1. [Privacy Core overview](privacy-core/overview.md)
2. [Orchard concepts](privacy-core/orchard-concepts.md)
3. [pERC20 overview](privacy-core/perc20/overview.md)

**Issue or integrate**

→ [pERC20 interface & ERC-20 mapping](privacy-core/perc20/interface-and-erc20.md)  
→ [Quick start](guides/quick-start.md)

## Repository & services

- Reference impl: [PERC20Labs/PERC20](https://github.com/PERC20Labs/PERC20)
- Off-chain: `privacy-indexer`, `privacy-relayer`, `privacy-prover`
- Repo map: [Appendix · repository layout](appendix/repo-layout.md)
