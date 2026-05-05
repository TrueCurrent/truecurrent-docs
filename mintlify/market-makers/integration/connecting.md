---
title: "Connecting to the RFQ stream"
description: "Technical details for connecting to TrueCurrent's MakerStream WebSocket: protocol, endpoints, message framing, protobuf definitions, and connection flow including keep-alive pings."
updatedAt: "2026-05-05"
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

- `"challenge"` — a `MakerChallenge` one-shot auth challenge (see [Auth handshake](#auth-handshake) below — must be handled before any `request` events arrive)
- `"pong"` — server-side heartbeat
- `"request"` — an `RFQRequestType` broadcast from a taker
- `"quote_ack"` — a `StreamAck` confirming the indexer accepted your quote
- `"error"` — a `StreamError` describing what went wrong

> Read `rfq-testing/src/rfq_test/proto/injective_rfq_rpc.proto` for the full message definitions. The Python client wraps all this for you in `src/rfq_test/clients/websocket.py::MakerStreamClient`.

---

### Auth handshake

Before the indexer forwards any `request` events,
it issues a one-shot EIP-712 v2 challenge against the maker address announced in connection metadata.
The stream stays open but silent until you reply correctly.

**Flow:**

```
1. MM connects          with maker_address metadata
2. Indexer sends        MakerChallenge `{ nonce, evm_chain_id, expires_at }`
3. MM signs             StreamAuthChallenge typed-data (EIP-712 v2)
4. MM sends             MakerStreamStreamingRequest `{ message_type: "auth", auth: MakerAuth }`
5. Indexer streams      request / quote_ack / settlement_update / …
```

**`MakerChallenge` fields (indexer → MM):**

| # | Field | Type | Meaning |
|---|---|---|---|
| 1 | `nonce` | string | Hex-encoded 32 bytes. Decode to raw bytes for signing. |
| 2 | `evm_chain_id` | uint64 | Chain ID for the EIP-712 domain (`1439` testnet, `1776` mainnet). |
| 3 | `expires_at` | sint64 | Unix milliseconds. Sign and reply within ~30s. |

<Warning>
The `MakerChallenge` message format uses a signed integer (`sint64`),
while the EIP-712 signature uses an unsigned integer (`uint64`).

Do *not* use `uint64` when constructing this `MakerChallenge` messages,
only use `sint64` for the `expires_at` field.
<Warning>

**`StreamAuthChallenge` typed-data layout:**

Type string: `"StreamAuthChallenge(uint64 evmChainId,address maker,bytes32 nonce,uint64 expiresAt)"`

| # | Field | Type | Encoding |
|---|---|---|---|
| 1 | `evmChainId` | uint64 | Big-endian, right-aligned in 32 bytes. |
| 2 | `maker` | address | 20 bytes from `bech32_to_evm(maker_inj)`, left-padded. |
| 3 | `nonce` | bytes32 | Raw 32 bytes from `MakerChallenge.nonce`. Hex-decoded, **not** keccak-hashed. |
| 4 | `expiresAt` | uint64 | Big-endian, right-aligned in 32 bytes. |

Uses the same domain separator as `SignQuote` (name `"RFQ"`, version `"1"`, `chainId`, `verifyingContract`).

**`MakerAuth` fields (MM → indexer):**

| # | Field | Type | Meaning |
|---|---|---|---|
| 1 | `evm_chain_id` | uint64 | Same value as `MakerChallenge.evm_chain_id`. |
| 2 | `signature` | string | `0x`-prefixed 65 bytes (r ‖ s ‖ v, v=0/1). |

Wrap in `MakerStreamStreamingRequest{ message_type: "auth", auth: MakerAuth{…} }` and send.

**Using the `rfq-testing` client (recommended):** Pass `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address` to `MakerStreamClient` — it handles the challenge-response automatically.

```python
mm_ws = MakerStreamClient(
    endpoint=env_config.indexer.ws_endpoint,
    auth_private_key=MM_PRIVATE_KEY,
    auth_evm_chain_id=1439,            # 1439 for testnet; 1776 for mainnet
    auth_contract_address=CONTRACT_ADDRESS,
    timeout=15.0,
)
await mm_ws.connect()
```

<Info>
**Troubleshooting**:

If your stream connects, pings return pongs,
but you never receive `request` events:
The auth challenge was not handled, or the signature failed.
Check that you handle `message_type: "challenge"`
and that your `MakerAuth` uses the correct `evm_chain_id`.
</Info>
