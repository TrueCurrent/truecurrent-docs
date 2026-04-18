---
title: "Connecting to the RFQ stream"
description: "Technical details for connecting to TrueCurrent's MakerStream WebSocket: protocol, endpoints, message framing, protobuf definitions, and connection flow including keep-alive pings."
updatedAt: "2026-04-18"
---

### Protocol details

gRPC-web over WebSocket, protobuf framing.

| Property | Value |
|---|---|
| Protocol | gRPC-web over WebSocket |
| Subprotocol | `grpc-ws` |
| Serialization | Protobuf (binary framing) |
| Framing | `[1 byte flags][4 bytes length BE][protobuf payload]` |
| Keep-alive | Ping every ~1 second |

### Endpoints

Canonical public path per `rfq-testing/configs/testnet.yaml` (ground truth — what the working Python client connects to):

- Package / service: `injective_rfq_rpc.InjectiveRfqRPC`
- Makers subscribe via `MakerStream`
- Takers subscribe via `TakerStream`

| Environment | MakerStream URL |
|---|---|
| Testnet | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/MakerStream` |
| Mainnet | *(TBD)* |

Additional endpoints exposed by the indexer:

| Transport | URL |
|---|---|
| gRPC | `testnet.rfq.grpc.injective.network:443` |
| gRPC-web | `https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC` |
| HTTP/REST | `https://testnet.rfq.injective.network` |

> **Heads-up on service naming:** the indexer's internal proto file (`rfq-indexer/proto/rfq/v1/api.proto`) declares the service as `rfq.v1.RfqService` with methods `StreamRequest` / `StreamQuote`. Public traffic is fronted under the `InjectiveRfqRPC` alias with `MakerStream` / `TakerStream` method names. Use the public alias — that's what the testnet deployment actually accepts, and it's what every working client library (`rfq-testing`, `rfq-qa-python-tests`) uses.

### Connection flow

```
MM                                    Indexer
│                                        │
│  ── WebSocket Connect ──────────────▶  │
│     subprotocol: grpc-ws               │
│                                        │
│  ◀── Connection Established ────────   │
│                                        │
│  ── Ping (every 1s) ───────────────▶  │  ← REQUIRED: server drops
│  ◀── Pong ─────────────────────────   │    idle connections
│                                        │
│  ◀── Request (RFQ from retail) ─────  │
│                                        │
│  ── Quote (signed) ─────────────────▶  │
│  ◀── Quote ACK ────────────────────   │
│                                        │
```

### Message framing

```
┌──────────┬──────────────┬──────────────────────────┐
│ Flag (1) │ Length (4 BE) │ Protobuf Payload (N)     │
│   0x00   │              │                           │
└──────────┴──────────────┴──────────────────────────┘
```

- **Flag**: `0x00` = data, `0x80` = trailer (ignore).
- **Length**: big-endian uint32.
- **Payload**: protobuf-encoded request/response.

### Protobuf messages (maker side)

The canonical proto is `injective_rfq_rpc.proto` (in `rfq-testing/src/rfq_test/proto/`). Your bidi stream uses `MakerStream`:

```protobuf
service InjectiveRfqRPC {
  // Bidirectional stream for makers: receive requests, send quotes
  rpc MakerStream (stream MakerStreamStreamingRequest) returns (stream MakerStreamResponse);
  // (and TakerStream, Request, StreamRequest, Quote, StreamQuote, etc.)
}
```

**What you send** — a `MakerStreamStreamingRequest` with `message_type = "ping"` or `"quote"` (and a nested `RFQQuoteInputType` when it's a quote).

**What you receive** — a `MakerStreamResponse` tagged by `message_type`:

- `"pong"` — server-side heartbeat
- `"request"` — an `RFQRequestType` broadcast from a taker
- `"quote_ack"` — a `StreamAck` confirming the indexer accepted your quote
- `"error"` — a `StreamError` describing what went wrong

> Read `rfq-testing/src/rfq_test/proto/injective_rfq_rpc.proto` for the full message definitions. The Python client wraps all this for you in `src/rfq_test/clients/websocket.py::MakerStreamClient`.
