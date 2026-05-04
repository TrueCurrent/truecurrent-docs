---
title: "Connecting to MakerStream"
description: "Complete guide to connecting to TrueCurrent's MakerStream WebSocket endpoint, receiving RFQ requests, submitting signed quotes within the 2-second window, and handling concurrent market making operations."
updatedAt: "2026-05-01"
---

The **MakerStream** is a WebSocket endpoint that delivers real-time RFQ requests from traders to market makers. Your quoting system connects to this stream, listens for requests, and responds with signed quotes.

---

## Connection

Connect to the MakerStream WebSocket endpoint:

| Environment | Endpoint |
|-------------|----------|
| Testnet | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/MakerStream` |
| Mainnet | *(TBD)* {/* TODO: add mainnet MakerStream URL when ready */} |

The MakerStream is the `MakerStream` bidirectional stream on the indexer's `injective_rfq_rpc.InjectiveRfqRPC` gRPC service — takers call `TakerStream` (see [TakerStream](/takers/taker-stream)); you call `MakerStream` to receive incoming RFQ requests and send signed quotes over the same connection.

The connection is gRPC-web framed over WebSocket, not raw JSON-RPC. Use the `rfq-testing` client library (Python or TypeScript) to handle the framing – see [`InjectiveLabs/rfq-testing`](https://github.com/InjectiveLabs/rfq-testing).

Authentication is based on your wallet address – the endpoint routes requests to your system based on your whitelisted maker address.

**Python example:**

```python
from rfq_test.clients.websocket import MakerStreamClient

mm_ws = MakerStreamClient(
    endpoint=env_config.indexer.ws_endpoint,
    timeout=15.0
)
await mm_ws.connect()
```

---

## Receiving requests

Once connected, you'll receive RFQ request events in real time. Each request contains:

| Field | Type | Description |
|-------|------|-------------|
| `rfq_id` | integer | Unique identifier for this RFQ |
| `market_id` | string | Injective market ID (hex string) |
| `direction` | string | `"long"` or `"short"` (taker's direction) |
| `margin` | string | Taker's margin in quote currency (USDC on the current testnet market) |
| `quantity` | string | Number of contracts requested |
| `worst_price` | string | Taker's worst acceptable price |
| `request_address` | string | Taker's Injective `inj1...` address |
| `expiry` | integer | Unix millisecond timestamp – request expires |

**Note on direction:** The `direction` field is the *taker's* direction. If the taker is `long`, you as the market maker are taking the *short* side. Price accordingly.

**Example request (Python):**

```python
request = await mm_ws.wait_for_request(timeout=15.0)

rfq_id = request["rfq_id"]
market_id = request["market_id"]
direction = request["direction"]     # taker's direction
margin = request["margin"]
quantity = request["quantity"]
taker = request["request_address"]
```

---

## Responding with a quote

You have **2 seconds** from when the request is broadcast to submit a quote. After that window closes, no further quotes for that `rfq_id` are accepted.

Your quote must include:

| Field | Type | Description |
|-------|------|-------------|
| `rfq_id` | integer | Must match the request |
| `market_id` | string | Must match the request |
| `taker_direction` | string | Must match the request direction |
| `margin` | string | Taker margin (can match or improve) |
| `quantity` | string | Quantity you'll fill |
| `price` | string | Your offered price |
| `expiry` | integer | Quote expiry (Unix ms) – recommend 30s from now |
| `maker` | string | Your Injective address |
| `taker` | string | Taker's Injective address (from request) |
| `signature` | string | `0x`-prefixed EIP-712 v2 signature (see [Signing quotes](/market-makers/signing-quotes)) |
| `sign_mode` | string | Must be `"v2"` |
| `maker_subaccount_nonce` | integer | Usually `0`; must match what you signed |
| `chain_id` | string | Cosmos chain ID for indexer compatibility, for example `"injective-888"` on testnet |
| `contract_address` | string | RFQ contract address for indexer compatibility |

**Example (Python):**

```python
from rfq_test.crypto.eip712 import sign_quote_v2

quote_expiry = int(time.time() * 1000) + 2_000  # short-lived quote, timestamp in ms

signature = sign_quote_v2(
    private_key=mm_wallet.private_key,
    evm_chain_id=1439,
    verifying_contract_bech32=contract_address,
    rfq_id=int(rfq_id),
    market_id=market_id,
    direction=direction,           # taker's direction
    taker=taker_address,
    taker_margin=margin,
    taker_quantity=quantity,
    maker=mm_wallet.inj_address,
    maker_margin=margin,           # your margin commitment
    maker_quantity=quantity,
    price=your_quoted_price,
    expiry_ms=quote_expiry,
    maker_subaccount_nonce=0,
    min_fill_quantity=None,
)

quote_data = {
    "rfq_id": rfq_id,
    "market_id": market_id,
    "taker_direction": direction,
    "margin": margin,
    "quantity": quantity,
    "price": your_quoted_price,
    "expiry": quote_expiry,
    "maker": mm_wallet.inj_address,
    "taker": taker_address,
    "signature": signature,         # already 0x-prefixed
    "sign_mode": "v2",
    "maker_subaccount_nonce": 0,
    "chain_id": chain_id,
    "contract_address": contract_address,
}

await mm_ws.send_quote(quote_data)
```

---

## Handling multiple simultaneous requests

In active markets, you may receive multiple RFQ requests in rapid succession. Your quoting system must be able to handle concurrent requests without one slowing down another. Each `rfq_id` is independent – there's no ordering requirement between responses to different requests.

Use async/concurrent processing to ensure your 2-second window isn't eaten up by sequential processing.

---

## Connection maintenance

Keep your MakerStream connection alive with periodic heartbeats. If your connection drops, requests during the disconnected period are missed. Implement reconnection logic with exponential backoff to handle transient network issues.

Persistent disconnections impact your response rate metrics, which can affect your market maker standing.
