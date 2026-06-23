# Developer guide

Integrate pERC20 into wallets, dApps, and backend services.

## Integration checklist

- [ ] Connect to correct chain (`chainId` in proofs = pool chain)
- [ ] Implement note scanning (`NoteAdded` + trial-decrypt)
- [ ] Integrate prover (rapidsnark service or WASM)
- [ ] Encode `PrivacyCall` / `BundleAction` (see `bundleCodec.ts`)
- [ ] Handle mint/burn public amounts vs private transfers
- [ ] Optional: relayer for submitter privacy
- [ ] Optional: indexer for faster sync
- [ ] Do **not** assume ERC-20 ABI compatibility

## Key libraries

| Task | Reference |
|------|-----------|
| ABI encode/decode | `apps/web/src/issuance/bundleCodec.ts` |
| EVM reads | `apps/web/src/crypto/evmRead.ts` |
| Witness build | `crates/privacybtc-witness/` |
| Groth16 prove | `crates/privacybtc-groth16/`, `privacy-prover` service |
| Contract interfaces | `contracts/interfaces/IPERC20.sol` |

## Contract addresses

Query factory → pool mapping on your target chain. Test env addresses in [Performance benchmarks](../ecosystem/performance.md).

## Testing

```bash
# Unit tests
forge test --match-contract PERC20Test

# Real-proof E2E
forge test --match-contract PERC20E2ETest

# Full stack
e2e/scripts/run_full_e2e.sh
```

## Common pitfalls

| Pitfall | Fix |
|---------|-----|
| Wrong `chainId` in sighash | Use pool's chain, not wallet network |
| Trusting `NoteAdded` alone | Recipient must trial-decrypt to confirm |
| On-chain `approve` | Use wallet ZIP-32 flow; no contract method |
| `pubFields` mismatch | Must equal top-level calldata fields |
| Skipping `< Fr` checks | Reference impl enforces; match in custom verifiers |

## *(expand)*

- REST API for prover-service
- Indexer query schema
- Mobile / hardware wallet considerations

## Next

- [Off-chain services](../ecosystem/off-chain-services.md)
- [Universal call format](../privacy-core/universal-call-format.md)
- [ERC-8302](../privacy-core/perc20/eip-8302.md)
