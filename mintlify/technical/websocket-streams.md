# WebSocket streams

TrueCurrent's off-chain coordination uses two real-time WebSocket streams: the **TakerStream** for traders and the **MakerStream** for market makers.

---

## TakerStream

The TakerStream is used by traders (or applications acting on their behalf) to submit RFQ requests and receive quotes.

### Connection

```
wss://[indexer-endpoint]/stream/rfq/taker
```

Authentication: Connect with the taker's Injective address as an identifier. The stream is scoped so that quotes are only returned to the address that submitted the corresponding request.

### Submitting a request

**Message format:**

```json
{
  "type": "rfq_request",
  "taker_address": "inj1...",
  "market_id": "0xabc123...",
  "direction": "long",
  "margin": "200",
  "quantity": "100",
  "worst_price": "4.7000",
  "rfq_id": 1708000700000
}
```

### Receiving quotes

Quotes arrive as WebSocket messages on the TakerStream:

```json
{
  "type": "rfq_quote",
  "rfq_id": 1708000700000,
  "price": "4.4550",
  "quantity": "100",
  "maker": "inj1makerwallet...",
  "expiry": 1708000830000,
  "signature": "0x..."
}
```

Use a collection window (e.g., 5 seconds after submitting the request) to gather all quotes before selecting the best one.

**Python example:**

```python
retail_ws = TakerStreamClient(
    endpoint=env_config.indexer.ws_endpoint,
    request_address=retail_wallet.inj_address,
    timeout=15.0,
)
await retail_ws.connect()

await retail_ws.send_request(request_data)

quotes = await retail_ws.collect_quotes(
    rfq_id=rfq_id,
    timeout=5.0,
    min_quotes=1,
)

best = min(quotes, key=lambda q: float(q.get("price", "999")))
```

---

## MakerStream

The MakerStream is used exclusively by whitelisted market makers to receive trade requests and submit quotes.

### Connection

```
wss://[indexer-endpoint]/stream/rfq/maker
```

Only whitelisted maker addresses can successfully subscribe to the MakerStream. Unauthorized connections will be rejected.

### Receiving requests

RFQ requests arrive as WebSocket messages:

```json
{
  "type": "rfq_request",
  "rfq_id": 1708000700000,
  "market_id": "0xabc123...",
  "direction": "long",
  "margin": "200",
  "quantity": "100",
  "worst_price": "4.7000",
  "taker_address": "inj1takerwallet..."
}
```

### Submitting quotes

```json
{
  "rfq_id": 1708000700000,
  "market_id": "0xabc123...",
  "taker_direction": "long",
  "margin": "200",
  "quantity": "100",
  "price": "4.4550",
  "expiry": 1708000830000,
  "maker": "inj1makerwallet...",
  "taker": "inj1takerwallet...",
  "signature": "0x...",
  "chain_id": "injective-1",
  "contract_address": "inj1contract..."
}
```

**Python example:**

```python
mm_ws = MakerStreamClient(
    endpoint=env_config.indexer.ws_endpoint,
    timeout=15.0,
)
await mm_ws.connect()

request = await mm_ws.wait_for_request(timeout=30.0)
# ... price and sign your quote ...
await mm_ws.send_quote(quote_data)
```

---

## Endpoints by environment

| Environment | Indexer WebSocket |
|-------------|-------------------|
| Testnet | *(TBD)* |
| Mainnet | *(TBD)* |

---

## Reliability and reconnection

Both streams should implement automatic reconnection. The indexer may close connections for maintenance or due to network issues. A robust client retries with exponential backoff:

```python
import asyncio

MAX_RETRIES = 10
BASE_DELAY = 1.0

for attempt in range(MAX_RETRIES):
    try:
        await ws.connect()
        await ws.run()           # your main loop
    except ConnectionError as e:
        delay = BASE_DELAY * (2 ** attempt)
        print(f"Disconnected: {e}. Retrying in {delay}s...")
        await asyncio.sleep(delay)
```

Note that during a disconnection, any requests broadcast to other market makers will still be serviced by them – you only miss the requests during your specific downtime window.
