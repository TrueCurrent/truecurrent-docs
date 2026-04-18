---
title: "Quickstart"
updatedAt: "2026-04-08"
---

This page walks you through the full lifecycle of a single RFQ trade as a programmatic taker. By the end, you will have submitted a request, received signed quotes, and settled onchain on Injective testnet.

{/* TODO: replace `rfq-testing` with the canonical prod repo name before mainnet launch. The word "testing" in the repo name is a testnet-era artifact; we need a permanent home for the taker SDK / examples under a name that reads right in production docs. Once the new repo exists, update every link and reference on this page and across the taker section. */}

Working code lives in [`InjectiveLabs/rfq-testing`](https://github.com/InjectiveLabs/rfq-testing):

- Python: [`examples/test_settlement.py`](https://github.com/InjectiveLabs/rfq-testing/blob/main/examples/test_settlement.py)
- TypeScript: [`examples/ts-retail/main.ts`](https://github.com/InjectiveLabs/rfq-testing/blob/main/examples/ts-retail/main.ts)

---

## Prerequisites

- An Injective wallet (a 32-byte secp256k1 private key)
- A small amount of **INJ** for gas on `injective-888` (testnet) – [testnet faucet](https://testnet-faucet.injective.dev)
- **USDC** in the wallet's exchange subaccount to cover your margin
- Network connectivity to `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/TakerStream`
{/* TODO : to add mainnet WS info */}
- Python 3.11+ or Node.js 18+

---

## 1. Install

**Python** – clone the testing repo and install the library:

```bash
git clone https://github.com/InjectiveLabs/rfq-testing.git
cd rfq-testing
python -m venv .venv && source .venv/bin/activate
pip install -e .
```

**TypeScript** – install the Injective SDK and the example deps:

```bash
cd rfq-testing/examples
npm install
```

---

## 2. Configure environment

Create a `.env` file at the repo root:

```bash
RFQ_ENV=testnet
PRIVATE_KEY=0x...       # your taker wallet private key
CONTRACT_ADDRESS=inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q
CHAIN_ID=injective-1   # injective-888 for testnet
```

The testnet config (`configs/testnet.yaml`) already points at:

| Setting | Value |
|---|---|
| Indexer WebSocket (TakerStream) | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/TakerStream` |
| Indexer HTTP | `https://testnet.rfq.injective.network` |
| Chain gRPC | `testnet-grpc.injective.dev:443` |
| Chain LCD | `https://testnet.sentry.lcd.injective.network` |
| RFQ contract | `inj1t8hyyle68vd0kzsdehxg0sywttrwmt58jzk29q` |
| INJ/USDC PERP | `0x17ef48032cb24375ba7c2e39f384e56433bcab20cbee9a7357e4cba2eb00abe6` |
{/* TODO: to add mainnet indexer info */}

> **API key – currently not required.** Today, you can connect to the testnet indexer without any authentication. The reference scripts in `rfq-testing` work as-is. Once the [RFQ Gateway](https://github.com/InjectiveLabs/rfq-gateway) is deployed in front of the public indexer, you'll need to add `RFQ_API_KEY=...` to your `.env` and pass it on every WebSocket connect and HTTP request. Programmatic / HFT takers will also need to ask the TrueCurrent team for an `api`-tier key – the default rate limit on a regular key (10 req/s) is way below what HFT needs. See [Rate limiting](/takers/best-practices#rate-limiting) for the breakdown.
>
> {/* TODO: once the gateway is live, add `RFQ_API_KEY` to this `.env` block, document the API key request process, and update all the example code to pass the key via `?api_key=` (WebSocket) and `X-API-Key` header (HTTP/REST). */}

---

## 3. Grant authz permissions (once)

Before you can accept any quotes, you must grant the TrueCurrent contract four message types via Injective's `authz` module. See [Authorization setup](/takers/authz-setup) for the full explanation.

**Python:**

```python
from rfq_test.clients.chain import ChainClient
from rfq_test.config import get_environment_config
from rfq_test.utils.setup import RETAIL_AUTHZ_GRANTS, setup_authz_grants
from rfq_test.crypto.wallet import Wallet

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

`RETAIL_AUTHZ_GRANTS` expands to four message types:

1. `/cosmos.bank.v1beta1.MsgSend`
2. `/injective.exchange.v2.MsgPrivilegedExecuteContract`
3. `/injective.exchange.v2.MsgBatchUpdateOrders`
4. `/injective.exchange.v2.MsgCreateDerivativeMarketOrder`

Each grant is a separate transaction. Wait for all four to confirm before continuing.

---

## 4. Submit a request

Open a TakerStream WebSocket and send an RFQ request.

**Python:**

```python
import time
from rfq_test.clients.websocket import TakerStreamClient

taker_ws = TakerStreamClient(
    endpoint=config.indexer.ws_endpoint,
    request_address=taker.inj_address,
    timeout=10.0,
)
await taker_ws.connect()

rfq_id = int(time.time() * 1000)

request_data = {
    "request_address": taker.inj_address,
    "rfq_id": rfq_id,
    "market_id": config.default_market.id,
    "direction": "long",                  # lowercase string
    "margin": "200",
    "quantity": "100",
    "worst_price": "5.00",                # hard price limit
    "expiry": rfq_id + 5 * 60 * 1000,     # 5 minutes from now
}

await taker_ws.send_request(request_data)
```

This is the exact pattern used by the reference scripts – see [`examples/test_settlement.py`](https://github.com/InjectiveLabs/rfq-testing/blob/main/examples/test_settlement.py).

**TypeScript:**

```ts
import WebSocket from "ws";

const ws = new WebSocket(
  "wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC/TakerStream",
);
const rfqId = Date.now();

const request = {
  type: "rfq_request",
  rfq_id: rfqId,
  market_id: INJUSDC_MARKET_ID,
  direction: "long",                    // lowercase string – NOT an integer
  margin: "200",
  quantity: "100",
  worst_price: "5.00",
  request_address: takerInjAddress,
  expiry: rfqId + 5 * 60 * 1000,
};

ws.send(JSON.stringify(request));
```

See [TakerStream](/takers/taker-stream) for the full request schema, the gRPC-web framing details, and the [rfq_id correlation patterns](/takers/taker-stream#how-rfq-id-correlation-works).

---

## 5. Collect quotes

Quotes stream in over the same WebSocket. Open a collection window, gather every quote matching your `rfq_id`, and pick the best one (or several – see [Accepting quotes](/takers/accepting-quotes) for multi-quote aggregation).

**Python:**

```python
quotes = await taker_ws.collect_quotes(
    rfq_id=rfq_id,
    timeout=0.5,           # sub-second; tune against benchmarks
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

await new Promise((r) => setTimeout(r, 5000));

const best = quotes.sort(
  (a, b) => Number(a.price) - Number(b.price),
)[0];
```

A sub-second collection window is typical – quotes are only valid for a short window, so waiting longer just eats into your settlement budget. See [Best practices](/takers/best-practices) for tuning. {/* TODO: to add precise collection window guidance once benchmarked */}

---

## 6. Accept onchain

Now submit `AcceptQuote` to the TrueCurrent contract. This is where the three encoding gotchas bite – read [Accepting quotes](/takers/accepting-quotes) for the full story.

**Python** (the `ContractClient.accept_quote` helper handles encoding for you):

```python
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
    worst_price=Decimal("5.00"),
    unfilled_action={"market": {}},        # fall back to orderbook if short-filled
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
      worst_price: "5.00",
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
      unfilled_action: { market: {} },
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

See [Best practices](/takers/best-practices) for reconnection and error handling patterns.

---

## Next

- [Authorization setup](/takers/authz-setup) – grant details and revocation
- [TakerStream](/takers/taker-stream) – request schema and quote collection
- [Accepting quotes](/takers/accepting-quotes) – multi-quote aggregation and every field of the `AcceptQuote` message
- [Best practices](/takers/best-practices) – slippage, expiry, reconnection, testing
