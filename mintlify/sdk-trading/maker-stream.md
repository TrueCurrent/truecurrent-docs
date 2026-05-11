---
title: "MakerStream"
description: "Connect to TrueCurrent's MakerStream WebSocket: protocol, endpoints, message framing, protobuf definitions, auth handshake, and keep-alive ping handling."
updatedAt: "2026-05-06"
---

### Protocol details

gRPC-web over WebSocket, protobuf framing.

| Property | Value |
| --- | --- |
| Protocol | gRPC-web over WebSocket |
| Subprotocol | `grpc-ws` |
| Serialization | Protobuf (binary framing) |
| Framing | `[1 byte flags][4 bytes length BE][protobuf payload]` |
| Keep-alive | Ping every ~1 second |

### Endpoints

Canonical public path per `injective-rfq-toolkit/configs/testnet.yaml` (ground truth — what the working Python client connects to):

- Package / service: `injective_rfq_rpc.InjectiveRfqRPC`
- Makers subscribe via `MakerStream`
- Takers subscribe via `TakerStream`

| Environment | MakerStream URL |
| --- | --- |
| Testnet | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/MakerStream` |
| Mainnet | Contact TrueCurrent for the current production endpoint |

Additional endpoints exposed by the indexer:

| Transport | URL |
| --- | --- |
| gRPC | `testnet.rfq.grpc.injective.network:443` |
| gRPC-web | `https://testnet.rfq.grpc.injective.network/injective_rfq_rpc.InjectiveRfqRPC` |
| HTTP/REST | `https://testnet.rfq.injective.network` |

<Info>
  **Service naming**:

  The indexer's internal proto file (`rfq-indexer/proto/rfq/v1/api.proto`) declares the service as `rfq.v1.RfqService` with methods `StreamRequest` / `StreamQuote`. Public traffic is fronted under the `InjectiveRfqRPC` alias with `MakerStream` / `TakerStream` method names. Use the public alias: that is what the testnet deployment actually accepts, and it's what every working client library (`injective-rfq-toolkit`, `rfq-qa-python-tests`) uses.
</Info>

### Connection flow

```text
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
│  ◀── Request (RFQ from taker) ──────  │
│                                        │
│  ── Quote (signed) ─────────────────▶  │
│  ◀── Quote ACK ────────────────────   │
│                                        │
```

### Message framing

```text
┌──────────┬──────────────┬──────────────────────────┐
│ Flag (1) │ Length (4 BE) │ Protobuf Payload (N)     │
│   0x00   │              │                           │
└──────────┴──────────────┴──────────────────────────┘
```

- **Flag**: `0x00` = data, `0x80` = trailer (ignore).
- **Length**: big-endian uint32.
- **Payload**: protobuf-encoded request/response.

### Protobuf messages (maker side)

The canonical proto is `injective_rfq_rpc.proto` (in `injective-rfq-toolkit/src/rfq_test/proto/`). Your bidi stream uses `MakerStream`:

```protobuf
service InjectiveRfqRPC {
  // Bidirectional stream for makers: receive requests, send quotes
  rpc MakerStream (stream MakerStreamStreamingRequest) returns (stream MakerStreamResponse);
  // (and TakerStream, Request, StreamRequest, Quote, StreamQuote, etc.)
}
```

**What you send** — a `MakerStreamStreamingRequest` with `message_type = "ping"` or `"quote"` (and a nested `RFQQuoteInputType` when it's a quote).

**What you receive** — a `MakerStreamResponse` tagged by `message_type`:

- `"challenge"` – a `MakerChallenge` one-shot auth challenge (see [Auth handshake](#auth-handshake) below — must be handled before any `request` events arrive)
- `"pong"` – server-side heartbeat
- `"request"` – an `RFQRequestType` broadcast from a taker
- `"quote_ack"` – a `StreamAck` confirming the indexer accepted your quote
- `"error"` – a `StreamError` describing what went wrong

> Read `injective-rfq-toolkit/src/rfq_test/proto/injective_rfq_rpc.proto` for the full message definitions. The Python client wraps all this for you in `src/rfq_test/clients/websocket.py::MakerStreamClient`.

---

### Auth handshake

Before the indexer forwards any `request` events, it issues a one-shot EIP-712 v2 challenge against the maker address announced in connection metadata. The stream stays open but silent until you reply correctly.

**Flow:**

```text
1. MM connects          with maker_address metadata
2. Indexer sends        MakerChallenge `{ nonce, evm_chain_id, expires_at }`
3. MM signs             StreamAuthChallenge typed-data (EIP-712 v2)
4. MM sends             MakerStreamStreamingRequest `{ message_type: "auth", auth: MakerAuth }`
5. Indexer streams      request / quote_ack / settlement_update / …
```

**`MakerChallenge` fields (indexer → MM):**

| # | Field | Type | Meaning |
| --- | --- | --- | --- |
| 1 | `nonce` | string | Hex-encoded 32 bytes. Decode to raw bytes for signing. |
| 2 | `evm_chain_id` | uint64 | Chain ID for the EIP-712 domain (`1439` testnet, `1776` mainnet). |
| 3 | `expires_at` | sint64 | Unix milliseconds. Sign and reply within ~30s. |

<Warning>
  The `MakerChallenge` message format uses a signed integer (`sint64`), while the EIP-712 signature uses an unsigned integer (`uint64`).

  Do _not_ use `uint64` when constructing this `MakerChallenge` message, only use `sint64` for the `expires_at` field.
</Warning>

**`StreamAuthChallenge` typed-data layout:**

Type string: `"StreamAuthChallenge(uint64 evmChainId,address maker,bytes32 nonce,uint64 expiresAt)"`

| # | Field | Type | Encoding |
| --- | --- | --- | --- |
| 1 | `evmChainId` | uint64 | Big-endian, right-aligned in 32 bytes. |
| 2 | `maker` | address | 20 bytes from `bech32_to_evm(maker_inj)`, left-padded. |
| 3 | `nonce` | bytes32 | Raw 32 bytes from `MakerChallenge.nonce`. Hex-decoded, **not** keccak-hashed. |
| 4 | `expiresAt` | uint64 | Big-endian, right-aligned in 32 bytes. |

Uses the same domain separator as `SignQuote` (name `"RFQ"`, version `"1"`, `chainId`, `verifyingContract`).

**`MakerAuth` fields (MM → indexer):**

| # | Field | Type | Meaning |
| --- | --- | --- | --- |
| 1 | `evm_chain_id` | uint64 | Same value as `MakerChallenge.evm_chain_id`. |
| 2 | `signature` | string | `0x`-prefixed 65 bytes (r ‖ s ‖ v, v=0/1). |

Wrap in `MakerStreamStreamingRequest{ message_type: "auth", auth: MakerAuth{…} }` and send.

**Manual challenge response shape:** Most integrations should use the reference client below. If you are implementing the stream yourself, the signing function has this shape; `_u`, `_addr`, `bech32_to_evm`, and `domain_separator` are the same low-level helpers used by the EIP-712 quote signer.

```python
from eth_utils import keccak
from eth_keys import keys

STREAM_AUTH_CHALLENGE_TYPE = (
    b"StreamAuthChallenge(uint64 evmChainId,address maker,bytes32 nonce,uint64 expiresAt)"
)

def sign_maker_challenge_v2(
    *,
    private_key,
    contract_address,
    maker_inj,
    evm_chain_id,
    nonce_hex,
    expires_at,
) -> str:
    nonce = bytes.fromhex(nonce_hex.removeprefix("0x"))
    assert len(nonce) == 32

    msg = b"".join((
        keccak(primitive=STREAM_AUTH_CHALLENGE_TYPE),
        _u(int(evm_chain_id), 8),
        _addr(bech32_to_evm(maker_inj)),
        nonce,  # raw bytes, not keccak-hashed
        _u(int(expires_at), 8),
    ))
    digest = keccak(
        primitive=b"\x19\x01"
        + domain_separator(evm_chain_id, contract_address)
        + keccak(primitive=msg)
    )
    sig = keys.PrivateKey(
        bytes.fromhex(private_key.removeprefix("0x"))
    ).sign_msg_hash(digest)
    v = sig.v - 27 if sig.v >= 27 else sig.v
    return "0x" + (
        sig.r.to_bytes(32, "big")
        + sig.s.to_bytes(32, "big")
        + bytes([v])
    ).hex()

# Inside the maker stream loop:
if resp.message_type == "challenge":
    sig = sign_maker_challenge_v2(
        private_key=MM_PRIVATE_KEY,
        contract_address=CONTRACT_ADDRESS,
        maker_inj=maker_address,
        evm_chain_id=int(resp.challenge.evm_chain_id),
        nonce_hex=resp.challenge.nonce,
        expires_at=int(resp.challenge.expires_at),
    )
    await send_queue.put(MakerStreamStreamingRequest(
        message_type="auth",
        auth={
            "evm_chain_id": int(resp.challenge.evm_chain_id),
            "signature": sig,
        },
    ))
```

**Using the `injective-rfq-toolkit` client (recommended):** Pass `auth_private_key`, `auth_evm_chain_id`, and `auth_contract_address` to `MakerStreamClient` — it handles the challenge-response automatically.

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

  If your stream connects, pings return pongs, but you never receive `request` events: The auth challenge was not handled, or the signature failed. Check that you handle `message_type: "challenge"` and that your `MakerAuth` uses the correct `evm_chain_id`.
</Info>