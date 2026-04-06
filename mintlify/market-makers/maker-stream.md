---
title: "Connecting to MakerStream"
description: "--"
updatedAt: "2026-04-06"
---

The **MakerStream** is a WebSocket endpoint that delivers real-time RFQ requests from traders to market makers. Your quoting system connects to this stream, listens for requests, and responds with signed quotes.

---

## Connection

Connect to the MakerStream WebSocket endpoint:

| Environment | Endpoint |
|-------------|----------|
| Testnet | `wss://testnet-indexer.injective.network/stream/rfq/maker` *(confirm exact URL)* |
| Mainnet | `wss://indexer.injective.network/stream/rfq/maker` *(confirm exact URL)* |

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
| `margin` | string | Taker's margin in quote currency (USDT) |
| `quantity` | string | Number of contracts requested |
| `worst_price` | string | Taker's worst acceptable price |
| `taker_address` | string | Taker's Injective `inj1...` address |
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
taker = request["taker_address"]
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
| `signature` | string | Your cryptographic signature (see [Signing quotes](/market-makers/signing-quotes)) |
| `chain_id` | string | Injective chain ID |
| `contract_address` | string | TrueCurrent contract address |

**Example (Python):**

```python
from rfq_test.crypto.signing import sign_quote

quote_expiry = int(time.time() * 1000) + 30_000  # 30 seconds from now

signature = sign_quote(
    private_key=mm_wallet.private_key,
    rfq_id=str(rfq_id),
    market_id=market_id,
    direction=direction,           # taker's direction
    taker=taker_address,
    taker_margin=margin,
    taker_quantity=quantity,
    maker=mm_wallet.inj_address,
    maker_margin=margin,           # your margin commitment
    maker_quantity=quantity,
    price=your_quoted_price,
    expiry=quote_expiry,
    chain_id=chain_id,
    contract_address=contract_address,
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
    "signature": signature,
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
