---
title: "Quickstart"
description: "Run a complete taker-side RFQ flow on testnet: submit a request, collect quotes, choose the best price, and settle with AcceptQuote."
updatedAt: "2026-05-14"
---

This page walks you through the full lifecycle of a single RFQ trade as a programmatic taker. By the end, you will have submitted a request, received signed quotes, and settled onchain on Injective testnet.


Working code lives in [`InjectiveLabs/injective-rfq-toolkit`](https://github.com/InjectiveLabs/injective-rfq-toolkit):

- Python: [`examples/test_settlement.py`](https://github.com/InjectiveLabs/injective-rfq-toolkit/blob/main/examples/test_settlement.py)
- TypeScript taker example: [`examples/ts-retail/main.ts`](https://github.com/InjectiveLabs/injective-rfq-toolkit/blob/main/examples/ts-retail/main.ts)

---

## Prerequisites

- An Injective wallet (a 32-byte secp256k1 private key)
- A small amount of **INJ** for gas on `injective-888` (testnet) – [testnet faucet](https://testnet-faucet.injective.dev)
- **USDC** in the wallet's exchange subaccount to cover your margin
- Network connectivity to RFQ WebSockets endpoint.
- Python 3.11+ or Node.js 18+

<Info>
See [connecting](/technical/connecting)
for all endpoints exposed by the indexer.
</Info>

---

## 1. Install

**Python** – clone the toolkit repo and install the library:

```bash
git clone https://github.com/InjectiveLabs/injective-rfq-toolkit.git
cd injective-rfq-toolkit
python -m venv .venv && source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
```

**TypeScript** – install the Injective SDK and the example deps:

```bash
cd injective-rfq-toolkit/examples
npm install
```

---

## 2. Configure environment

Create a `.env` file at the repo root:

```bash
RFQ_ENV=testnet
TESTNET_RETAIL_PRIVATE_KEY=0x...   # your taker wallet private key
```

The testnet config (`configs/testnet.yaml`) already comes preconfigured
with connection details.

See: [`https://github.com/InjectiveLabs/injective-rfq-toolkit/blob/main/configs/testnet.yaml`](https://github.com/InjectiveLabs/injective-rfq-toolkit/blob/main/configs/testnet.yaml)

<Info>
See [connecting](/technical/connecting)
for all endpoints exposed by the indexer.
</Info>

Anybody can access the public RFQ streams. The reference scripts in `injective-rfq-toolkit` connect without additional stream authentication.

---

## 3. Grant authz permissions (once)

Before you can accept any quotes, you must grant the TrueCurrent contract three message types via Injective's `authz` module. See [Authorization setup](/sdk-trading/authz) for the full explanation.

**Python:**

```python
import os
from rfq_test.clients.chain import ChainClient
from rfq_test.config import get_environment_config
from rfq_test.utils.setup import RETAIL_AUTHZ_GRANTS, setup_authz_grants
from rfq_test.crypto.wallet import Wallet

TAKER_PRIVATE_KEY = os.environ["TESTNET_RETAIL_PRIVATE_KEY"]

config = get_environment_config()
chain = ChainClient(config.chain)
await chain.connect()

taker = Wallet.from_private_key(TAKER_PRIVATE_KEY)

await setup_authz_grants(
    chain_client=chain,
    wallet=taker,
    contract_address=config.contract.address,
    msg_types=RETAIL_AUTHZ_GRANTS,
)
await chain.close()
```

`RETAIL_AUTHZ_GRANTS` expands to three message types:

1. `/injective.exchange.v2.MsgPrivilegedExecuteContract`
2. `/injective.exchange.v2.MsgBatchUpdateOrders`
3. `/cosmos.bank.v1beta1.MsgSend`

Each grant is a separate transaction. Wait for all three to confirm before continuing.

---

## 4. Submit a request

Open a TakerStream WebSocket and send an RFQ request.

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
    "request_address": taker.inj_address,
    "client_id": str(uuid.uuid4()),
    "market_id": config.default_market.id,
    "direction": "long",                  # lowercase string
    "margin": "200",
    "quantity": "100",
    "worst_price": "5",                   # hard price limit
    "expiry": expiry_ms,                  # 5 minutes from now
}

ack = await taker_ws.send_request(
    request_data,
    wait_for_response=True,
    response_timeout=5.0,
)
rfq_id = int(ack["rfq_id"])               # indexer-assigned id
```

Always use the ACK's `rfq_id` for quote collection and settlement. The request uses a client UUID for correlation; the indexer assigns the RFQ id that makers quote against.

**TypeScript:**

```ts
import WebSocket from "ws";

const ws = new WebSocket(
  "wss://rfq.ws.testnet.injective.network/injective_rfq_rpc.InjectiveRfqRPC/TakerStream",
);
const clientId = crypto.randomUUID();
const expiry = Date.now() + 5 * 60 * 1000;

const request = {
  type: "rfq_request",
  client_id: clientId,
  market_id: INJUSDC_MARKET_ID,
  direction: "long",                    // lowercase string – NOT an integer
  margin: "200",
  quantity: "100",
  worst_price: "5",
  request_address: takerInjAddress,
  expiry,
};

ws.send(JSON.stringify(request));

const ack = await waitForRequestAck(ws, clientId);
const rfqId = Number(ack.rfq_id);
```

For TypeScript, parse the request ACK and use the returned `rfq_id` for quote filtering and settlement. Do not use `Date.now()` as the settlement `rfq_id`.

See [TakerStream](/sdk-trading/taker-stream) for the full request schema, the gRPC-web framing details, and the [rfq_id correlation patterns](/sdk-trading/taker-stream#how-rfq-id-correlation-works).

---

## 5. Collect quotes

Quotes stream in over the same WebSocket. Open a collection window, gather every quote matching your `rfq_id`, and pick the best one (or several – see [Accepting quotes](/sdk-trading/accepting-quotes) for multi-quote aggregation).

**Python:**

```python
quotes = await taker_ws.collect_quotes(
    rfq_id=rfq_id,
    timeout=0.5,           # 500ms collection window; tune against real latency
    min_quotes=1,
)

if not quotes:
    raise RuntimeError("No quotes received within window")

best = min(quotes, key=lambda q: float(q["price"]))
print(f"Best: {best['price']} from {best['maker']}")
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

await new Promise((r) => setTimeout(r, 500)); // tune against real latency

const best = quotes.sort(
  (a, b) => Number(a.price) - Number(b.price),
)[0];
```

Keep the collection window short. A few hundred milliseconds is a reasonable starting point; waiting near the full live quote expiry window leaves little time for settlement. See [Best practices](/sdk-trading/taker-best-practices) for tuning.

---

## 6. Accept onchain

Now submit `AcceptQuote` to the TrueCurrent contract. This is where the three encoding gotchas bite – read [Accepting quotes](/sdk-trading/accepting-quotes) for the full story.

**Python** (the `ContractClient.accept_quote` helper handles encoding for you):

```python
from decimal import Decimal
from rfq_test.clients.contract import ContractClient
from rfq_test.models.types import Direction

contract = ContractClient(config.contract, config.chain)

contract_quote = {
    "maker": best["maker"],
    "margin": best["margin"],
    "quantity": best["quantity"],
    "price": best["price"],
    "expiry": int(best["expiry"]),         # defensive cast – client wraps to {"ts": ...}
    "signature": best["signature"],        # hex with 0x prefix – client converts to base64
}

tx_hash = await contract.accept_quote(
    private_key=TAKER_PRIVATE_KEY,
    quotes=[contract_quote],
    rfq_id=str(rfq_id),
    market_id=config.default_market.id,
    direction=Direction.LONG,
    margin=Decimal("200"),
    quantity=Decimal("100"),
    worst_price=Decimal("5"),
    unfilled_action=None,                  # RFQ-only; current product does not expose orderbook fallback
)

print(f"Settled: {tx_hash}")
```

**TypeScript** – because no high-level helper exists yet, you build the execute message yourself. The raw JSON shape with every gotcha applied:

```ts
import { MsgExecuteContractCompat, MsgBroadcasterWithPk } from "@injectivelabs/sdk-ts";

// Contract expects base64, indexer delivers hex
const signatureB64 = Buffer.from(
  best.signature.replace(/^0x/, ""),
  "hex",
).toString("base64");

const msg = MsgExecuteContractCompat.fromJSON({
  sender: takerInjAddress,
  contractAddress: CONTRACT_ADDRESS,
  msg: {
    accept_quote: {
      rfq_id: rfqId,                       // NUMBER, not string
      market_id: INJUSDC_MARKET_ID,
      direction: "long",                   // lowercase STRING for contract
      margin: "200",
      quantity: "100",
      worst_price: "5",
      quotes: [
        {
          maker: best.maker,
          margin: best.margin,
          quantity: best.quantity,
          price: best.price,
          expiry: { ts: Number(best.expiry) },   // wrap + defensive cast
          signature: signatureB64,               // base64
        },
      ],
      unfilled_action: null,
    },
  },
});

const broadcaster = new MsgBroadcasterWithPk({
  privateKey: RETAIL_PRIVATE_KEY,
  network: Network.TestnetSentry,
});

const { txHash } = await broadcaster.broadcast({ msgs: msg });
console.log("Settled:", txHash);
```

If the transaction succeeds, your position is open. Query it via the standard Injective exchange APIs – there is no RFQ-specific position state.

---

## What happens on failure

| Error | Cause | Fix |
|---|---|---|
| `unauthorized` | Missing authz grant | Run step 3 |
| `quote expired` | Quote expiry passed before your tx confirmed | Re-collect quotes, submit faster |
| `signature verification failed` | Passed signature as hex, not base64 | Decode the indexer's hex and re-encode base64 |
| `deserialize Expiry` | Passed `expiry` as a raw int | Wrap as `{"ts": <ms>}` |
| `insufficient balance` | Subaccount margin too low | Deposit more USDC to the exchange subaccount |
| `maker not whitelisted` | Indexer forwarded a stale quote | Skip that quote, try the next best |

See [Best practices](/sdk-trading/taker-best-practices) for reconnection and error handling patterns.

---

## Next

- [Authorization setup](/sdk-trading/authz) – grant details and revocation
- [TakerStream](/sdk-trading/taker-stream) – request schema and quote collection
- [Accepting quotes](/sdk-trading/accepting-quotes) – multi-quote aggregation and every field of the `AcceptQuote` message
- [Best practices](/sdk-trading/taker-best-practices) – slippage, expiry, reconnection, testing
