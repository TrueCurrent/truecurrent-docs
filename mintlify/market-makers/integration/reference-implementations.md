---
title: "Reference implementations"
description: "Overview and production-readiness assessment of the TypeScript and Python reference implementations for TrueCurrent market maker integration."
updatedAt: "2026-04-18"
---

Two example clients exist. Only the Python client is production-compatible against the live indexer.

### `rfq-ts-example` (TypeScript)

| Aspect | Status | Notes |
|---|---|---|
| Transport | JSON-RPC over raw WebSocket | ⚠️ **Outdated** — uses `ws://localhost:4476/ws` with JSON-RPC, which is the local-dev indexer shape. Does not work with the gRPC-web-framed production indexer. |
| Signing | ✅ Correct | `keccak256 → secp256k1` with correct field order. |
| Verdict | **Not production-ready.** Good for understanding the signing logic, but the transport layer is incompatible with live testnet/mainnet. |

### `rfq-qa-python-tests` (Python)

| Aspect | Status | Notes |
|---|---|---|
| Transport | gRPC-web over WebSocket | ✅ Production-compatible — `wss://testnet.rfq.ws.injective.network/…` with protobuf framing. |
| Signing | ✅ Correct | Same `keccak256 → secp256k1`. |
| Ping/pong | ✅ Implemented | ~1s interval. |
| Verdict | **Use this as your reference.** Tested against live testnet. |

### If you build in TypeScript

The **signing logic is identical** — copy it from `rfq-ts-example/mm-scripts/main.ts` and trust it. Replace the WebSocket transport with gRPC-web framing (see the [Connecting to the RFQ stream](/market-makers/integration/connecting) section).
