---
title: "TakerStream"
description: "Reference for TrueCurrent's TakerStream WebSocket API including connection setup, gRPC-web framing, request message schema, quote subscription, collection windows, filtering, and reconnection patterns for programmatic takers."
updatedAt: "2026-05-06"
---

The **TakerStream** is the WebSocket endpoint you use as a taker to submit RFQ requests and receive quotes. It is the offchain half of the RFQ flow – once you've picked a quote, everything after is onchain.

---

## Connection

| Environment | WebSocket URL |
|---|---|
| Testnet | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/TakerStream` |
| Mainnet | Contact TrueCurrent for the current production endpoint |

The connection is **gRPC-web over WebSocket**. The URL is `<host>/<service>/<method>` where the service is `injective_rfq_rpc.InjectiveRfqRPC` and the method for takers is `TakerStream` (makers use `MakerStream`). Messages are framed with gRPC-web length prefixes and use protobuf payloads. The `injective-rfq-toolkit` Python library and reference examples handle this framing for you. If you're building from scratch, see `src/rfq_test/clients/websocket.py` for the implementation.

---

## Authentication

Anybody can access the public RFQ streams. TakerStream is address-routed: you provide your Injective address when opening the stream, and the indexer routes quotes for your requests back to that address. The reference scripts in `injective-rfq-toolkit` connect this way out of the box.

---

## Submitting a request

An RFQ request declares what you want to trade. The indexer broadcasts it to every active maker, who each have a few hundred milliseconds to respond.

**Request fields:**

| Field | Type | Description |
|---|---|---|
| `client_id` | string | A UUID you generate for request/ACK correlation. The indexer assigns the `rfq_id`; use the ACK's `rfq_id` for quote collection and settlement. |
| `market_id` | string | Injective derivative market ID (hex) |
| `direction` | string | Lowercase `"long"` or `"short"` |
| `margin` | string | Your margin in USDC, as a decimal string (e.g. `"200"`) |
| `quantity` | string | Contracts requested (e.g. `"100"`) |
| `worst_price` | string | Hard price limit. Quotes worse than this are rejected onchain. |
| `expiry` | uint64 | Unix ms after which the request is ignored. Typically now + 5 min. |

`request_address` is required as TakerStream connection metadata. With `injective-rfq-toolkit`, set it on `TakerStreamClient`; do not put it in the RFQ request body.

**Python:**

```python
import time
import uuid
from rfq_test.clients.websocket import TakerStreamClient

taker_ws = TakerStreamClient(
    endpoint=config.indexer.ws_endpoint,
    request_address=taker.inj_address,
    timeout=10.0,
)
await taker_ws.connect()

expiry_ms = int(time.time() * 1000) + 5 * 60 * 1000

request_data = {
    "client_id": str(uuid.uuid4()),
    "market_id": config.default_market.id,
    "direction": "long",
    "margin": "200",
    "quantity": "100",
    "worst_price": "5",
    "expiry": expiry_ms,
}

ack = await taker_ws.send_request(
    request_data,
    wait_for_response=True,
    response_timeout=5.0,
)
rfq_id = int(ack["rfq_id"])
```

This ACK-based pattern is the one to use for production and testnet validation. It avoids silently filtering out every quote when the local clock and indexer-assigned `rfq_id` differ.

> A convenience wrapper, `rfq_test.factories.request.RequestFactory.create_indexer_request(...)`, produces the same dict shape if you'd rather not hand-build it. Both approaches are equivalent.

**TypeScript:**

```ts
const clientId = crypto.randomUUID();
const expiry = Date.now() + 5 * 60 * 1000;

const request = {
  type: "rfq_request",
  client_id: clientId,
  market_id: INJUSDC_MARKET_ID,
  direction: "long",                   // lowercase string – NOT an integer
  margin: "200",
  quantity: "100",
  worst_price: "5",
  expiry,
};

// request_address is sent as TakerStream metadata when opening the stream.
ws.send(JSON.stringify(request));

const ack = await waitForRequestAck(ws, clientId);
const rfqId = Number(ack.rfq_id);
```

> **Note:** some older examples in the repo use `direction: 0` against a local dev JSON-RPC indexer. On testnet (gRPC-web framed over WebSocket), the indexer expects the lowercase string form – `"long"` or `"short"`.

---

## Receiving quotes

After submitting, quotes stream in over the same connection as they're produced by makers. Each quote is a separate message. You should expect anywhere from zero to `N` quotes (one per active maker) within the collection window.

**Quote fields** (as delivered by `collect_quotes()` – see `websocket.py` `_quote_to_dict`):

| Field | Type | Description |
|---|---|---|
| `rfq_id` | string | Matches your request (note: stringified on the Python side) |
| `market_id` | string | Matches your request |
| `maker` | string | Maker's `inj1...` address |
| `taker` | string | Your address (echoed) |
| `taker_direction` | string | Echoes your direction as `"long"` / `"short"` |
| `margin` | string | Margin the maker is committing to cover their side |
| `quantity` | string | Quantity the maker is willing to fill |
| `price` | string | Maker's quoted price |
| `expiry` | int | Quote expiry (Unix ms). Cast with `int(quote["expiry"])` defensively before passing to the contract. |
| `signature` | string | Maker's secp256k1 signature, delivered as **hex with `0x` prefix** (e.g. `"0xabc123..."`) |
| `status` | string | Quote state. `"pending"` when first received. |
| `nonce` | uint64 \| null | Reserved for non-standard quote variants; normal RFQ quotes carry `null` |

> **Signature format:** the indexer delivers the signature as hex (e.g. `"0xabc123..."`). The onchain contract requires **base64**. You must convert before building the `AcceptQuote` message. See [Accepting quotes](/sdk-trading/accepting-quotes).

> **Expiry format:** here it arrives as a plain integer in Unix milliseconds. In the onchain message it must be wrapped as the `Expiry` enum variant – see [Accepting quotes](/sdk-trading/accepting-quotes).

---

## Collection window pattern

Because quotes arrive asynchronously, the standard pattern is:

1. Submit the request
2. Collect incoming messages into a list, filtering by `rfq_id`
3. After a short fixed window (a few hundred milliseconds is typical – quotes expire quickly, so you can't wait long), stop collecting and pick the best quote(s)
4. Proceed to settlement immediately


**Python:**

```python
quotes = await taker_ws.collect_quotes(
    rfq_id=rfq_id,
    timeout=0.5,         # collect for ~500ms, then proceed – tune against real benchmarks
    min_quotes=1,
)

if not quotes:
    raise RuntimeError("No quotes received")

# For a long: lowest price wins. For a short: highest price wins.
best = min(quotes, key=lambda q: float(q["price"]))
```

**TypeScript:**

```ts
const quotes: Quote[] = [];

ws.on("message", (raw: Buffer) => {
  const msg = JSON.parse(raw.toString());
  if (msg.type === "rfq_quote" && msg.rfq_id === rfqId) {
    quotes.push(msg);
  }
});

await new Promise((r) => setTimeout(r, 500));   // tune against real benchmarks

if (quotes.length === 0) {
  throw new Error("No quotes received");
}

const best = quotes
  .slice()
  .sort((a, b) => Number(a.price) - Number(b.price))[0];
```

**Choosing the window size:**

The hard ceiling is the quote's `expiry`. By the time you've accounted for a few hundred milliseconds of MM response latency, Injective block time (~600ms), and your own broadcast path, there is very little room.

- **A few hundred milliseconds** is a reasonable default – long enough to receive quotes from warm makers, short enough to leave settlement headroom.
- **Too short** (&lt;100ms): you'll miss slower makers and reduce competition.
- **Too long** (&gt;1s): quote expiry becomes a race you will lose.


You can also use a **first-fit pattern**: accept the first quote that satisfies your conditions (price inside `worst_price`, quantity ≥ your needs) and skip the window. This is lowest latency but gives worse execution.

---

## How `rfq_id` correlation works

The wire protocol is slightly different from what most of the examples suggest. It's worth understanding if you're building a production taker.

**On the wire, the taker's outgoing request (`RFQRequestInputType`) carries a `client_id` (UUID), not an `rfq_id`.** The indexer assigns the `rfq_id` when it receives the request, then broadcasts an inbound request (`RFQRequestType`) to all MMs with both `client_id` and the newly-minted `rfq_id`. MMs quote against the assigned `rfq_id`. When the taker calls `collect_quotes(rfq_id=...)`, it's matching on that indexer-assigned value. The `Request` unary RPC's `RequestResponse` returns `{status, client_id, rfq_id}` so you can learn the assigned id synchronously — see the `injective_rfq_rpc.proto` message definitions in `injective-rfq-toolkit/src/rfq_test/proto/`.

Use ACK-based correlation. Generating a local millisecond timestamp and treating it as the settlement `rfq_id` is not reliable: the indexer assigns the actual `rfq_id`, and makers quote against that assigned value.

```python
request_data = {
    "client_id": str(uuid.uuid4()),
    "market_id": market.id,
    "direction": "long",
    "margin": "200",
    "quantity": "100",
    "worst_price": "5",
    "expiry": int(time.time() * 1000) + 300_000,
}

ack = await taker_ws.send_request(request_data, wait_for_response=True, response_timeout=5.0)
real_rfq_id = int(ack["rfq_id"])
quotes = await taker_ws.collect_quotes(rfq_id=real_rfq_id, timeout=0.5, min_quotes=1)
```

The ACK adds one round-trip, but it removes the main silent failure mode in quote collection.

---

## Filtering and routing

If you operate multiple takers or submit multiple concurrent RFQs, you need to route incoming quotes to the right request. Every quote message carries its `rfq_id`, so a simple map works:

**Python:**

```python
class QuoteRouter:
    def __init__(self):
        self.pending: dict[int, asyncio.Queue] = {}

    async def submit(self, rfq_id: int, request: dict) -> list[dict]:
        queue: asyncio.Queue = asyncio.Queue()
        self.pending[rfq_id] = queue
        await taker_ws.send_request(request)

        quotes = []
        try:
            while True:
                quote = await asyncio.wait_for(queue.get(), timeout=5.0)
                quotes.append(quote)
        except asyncio.TimeoutError:
            pass
        finally:
            del self.pending[rfq_id]
        return quotes

    def on_message(self, msg: dict):
        rfq_id = msg.get("rfq_id")
        if rfq_id in self.pending:
            self.pending[rfq_id].put_nowait(msg)
```

This lets you submit many requests over a single connection and collect their quotes independently.

---

## Reconnection

WebSocket connections drop. Your taker must reconnect gracefully or it becomes blind.

**Exponential backoff:**

```python
import asyncio

MAX_RETRIES = 10
BASE_DELAY = 1.0

for attempt in range(MAX_RETRIES):
    try:
        await taker_ws.connect()
        await taker_ws.run()
    except ConnectionError as e:
        delay = min(BASE_DELAY * (2 ** attempt), 30.0)
        print(f"Disconnected: {e}. Retry in {delay}s")
        await asyncio.sleep(delay)
```

**During a disconnect, in-flight quotes are lost.** There is no replay. If the disconnect happens mid-collection-window, you have to re-send the RFQ with a new `rfq_id`. (Reusing the same `rfq_id` triggers nonce replay protection onchain.)

For production systems, maintain a warm standby connection and fail over at the first dropped frame rather than tearing down and rebuilding.

---

## Inspecting the stream

A useful debug pattern while developing is to dump every incoming message to stdout:

```python
taker_ws = TakerStreamClient(endpoint, request_address, timeout=15.0)
taker_ws.on_message = lambda m: print("<-", m)
await taker_ws.connect()
```

If you send a request and see nothing come back:

- Check that `request_address` exactly matches the wallet you're using
- Verify there are active whitelisted makers on testnet – query the contract's `list_makers` entry
- Check `configs/testnet.yaml` for the current indexer URL; the path component matters (`/injective_rfq_rpc.InjectiveRfqRPC/TakerStream` for takers, `/MakerStream` for makers)
- If using gRPC-web framing manually, confirm your length-prefix and compression flag bytes are correct

---

## Next

- [Accepting quotes](/sdk-trading/accepting-quotes) – turning a collected quote into an onchain settlement
- [Best practices](/sdk-trading/taker-best-practices) – window tuning, nonce handling, reconnection strategy
