# Quick start

Run the reference implementation locally.

## Prerequisites

- **Foundry** (`forge`, `anvil`, `cast`)
- **Rust** stable + `cargo`
- **Node 18+** + **circom v2.x**
- (Optional) `privacy-indexer` + `privacy-relayer` sibling repos

## Fast path — fixture E2E (no local zkey)

```bash
git clone https://github.com/PERC20Labs/PERC20.git
cd PERC20
forge test --match-path test/PERC20E2E.t.sol
```

## Build contracts

```bash
forge build
forge test --gas-report --match-contract PERC20E2ETest
```

## Compile circuit (optional)

```bash
cd circuits
npm install
npm run compile:action
snarkjs r1cs info build/action/action.r1cs
```

## Web wallet (Docker Compose)

See root `README.md` and deployment docs for full stack.

## Full E2E

```bash
e2e/scripts/run_full_e2e.sh
```

## Next steps

| Goal | Doc |
|------|-----|
| Understand modules | [Privacy Core](../privacy-core/overview.md) · [Applications](../applications/overview.md) |
| Issue a token | [Issuer guide](issuer-guide.md) |
| Integrate wallet | [Developer guide](developer-guide.md) |
