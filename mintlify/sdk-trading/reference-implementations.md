---
title: "Reference implementations"
description: "Overview of the Python, TypeScript, and Go reference implementations for TrueCurrent taker and maker SDK integrations."
updatedAt: "2026-05-06"
---

The `rfq-testing`[ repo](https://github.com/InjectiveLabs/rfq-testing) contains all reference implementations. Both Python and TypeScript are production-compatible against the live indexer.

### Python (WebSocket / gRPC-web)

| Aspect | Status | Notes |
| --- | --- | --- |
| Transport | gRPC-web over WebSocket | ✅ Production-compatible — `wss://testnet.rfq.ws.injective.network/…` with protobuf framing. |
| Signing | ✅ Correct | `keccak256 → secp256k1` with correct field order, `evmChainId` field #1. |
| Auth handshake | ✅ Implemented | `MakerStreamClient` handles `MakerChallenge` automatically when configured with `auth_private_key`, `auth_evm_chain_id`, `auth_contract_address`. |
| Ping/pong | ✅ Implemented | ~1s interval. |
| Entry points | `examples/python-mm/main.py` (WebSocket MM reference), `examples/test_settlement.py` (full E2E) |  |

### Python (native gRPC)

| Aspect | Status | Notes |
| --- | --- | --- |
| Transport | Native gRPC | ✅ Alternative transport using `testnet.rfq.grpc.injective.network:443`. |
| Entry points | `examples/python-mm/main-grpc.py` (MM reference), `examples/test_settlement_grpc.py` (full E2E) |  |

### TypeScript (native gRPC)

| Aspect | Status | Notes |
| --- | --- | --- |
| Transport | Native gRPC | ✅ Production-compatible via `ts-mm/main-grpc.ts`. |
| Signing | ✅ Correct | Full `signQuoteV2` and `signMakerChallengeV2` implementations using `ethers`. |
| Entry point | `examples/ts-mm/main-grpc.ts` |  |

### Go (native gRPC)

| Aspect | Status | Notes |
| --- | --- | --- |
| Transport | Native gRPC | ✅ Reference available. |
| Entry point | `examples/go-mm/main-grpc/main.go` |  |

---

### Repository

All examples live in [`InjectiveLabs/rfq-testing`](https://github.com/InjectiveLabs/rfq-testing).

```bash
git clone https://github.com/InjectiveLabs/rfq-testing.git
cd rfq-testing
pip install -e ".[dev]"
```

For TypeScript, see the TypeScript signing code in the [Signing quotes](/sdk-trading/signing-quotes) section for the standalone `signQuoteV2` and `signMakerChallengeV2` implementations.