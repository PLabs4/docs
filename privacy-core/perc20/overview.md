# pERC20 overview

**pERC20** is the **native private fungible token** asset protocol: Orchard notes plus ERC-20-like product semantics. Private assets created here can be used by higher-level applications (pSWAP, pDEX, pX402) documented outside this book.

## One line

> Fungible tokens that are **private from creation**, with an ERC-20-like API and different openness.

## Contract structure

```solidity
contract PERC20 is OrchardVerifier, IPERC20
```

| Inheritance | Provides |
|-------------|----------|
| `OrchardVerifier` | Privacy Core state machine |
| `IPERC20` | Metadata, `totalSupply`, `mint/burn/transfer` |

## Privacy Core vs pERC20 delta

| Capability | Privacy Core | pERC20 adds |
|------------|--------------|-------------|
| Note spend/create | ✅ | — |
| Groth16 + sigs + freeze | ✅ | — |
| `name/symbol/decimals` | — | ✅ |
| Public `totalSupply` | — | ✅ |
| Issuer-only `mint` | — | ✅ |
| Off-chain ERC-20-like API | — | ✅ (wallet) |

## Deployment

1. Deploy verifier + factory once per chain
2. `createPerc20(...)` → EIP-1167 clone (~321K gas)
3. Issuer mints to shielded notes

→ [Issuance & deployment](issuance-and-deployment.md)

## Standard

| Item | Value |
|------|-------|
| ERC | **8302** (submission) / 8289 (repo draft until synced) |
| Status | Draft |
| ERCs PR | [#1817](https://github.com/ethereum/ERCs/pull/1817) |

→ [ERC-8302 index](eip-8302.md) · [ERC-8289 (prior number)](eip-8289.md)

## Next

- [Interface & ERC-20 mapping](interface-and-erc20.md)
