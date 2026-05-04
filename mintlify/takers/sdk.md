---
title: "SDK"
description: "Canonical SDK reference for programmatic takers on TrueCurrent: repository links, installation, a minimal hello-world, and a table of available SDK helpers."
updatedAt: "2026-05-01"
---

This page is the starting point for programmatic takers. It covers where to find the SDK, how to install it, a minimal end-to-end example, and the available helper components.

For a full lifecycle walkthrough — including authz setup, quote collection, and onchain settlement — see [Quickstart](/takers/quickstart).

---

## Repository

{/* TODO: replace `rfq-testing` with the canonical prod repo name before mainnet launch. The word "testing" in the repo name is a testnet-era artifact; we need a permanent home for the taker SDK / examples under a name that reads right in production docs. Once the new repo exists, update the links below and everywhere in the taker section. */}

| SDK | Repository | Status |
|---|---|---|
| **Python (taker + MM reference)** | [`InjectiveLabs/rfq-testing`](https://github.com/InjectiveLabs/rfq-testing) | Testnet-era name; prod repo TBD |
| **TypeScript (Injective SDK)** | [`InjectiveLabs/injective-ts`](https://github.com/InjectiveLabs/injective-ts) | Production-ready; no TrueCurrent-specific taker helpers yet |

The Python `rfq_test` package is the canonical reference for takers. It is the only client tested against the live testnet indexer. The TypeScript side relies on `@injectivelabs/sdk-ts` directly, with no TrueCurrent-specific wrapper yet.

For market maker reference implementations (both Python and TypeScript), see [Reference implementations](/market-makers/integration/reference-implementations).

---

## Installation

**Python** — clone the testnet repo and install the `rfq_test` package with its development dependencies:

```bash
git clone https://github.com/InjectiveLabs/rfq-testing.git
cd rfq-testing
python -m venv .venv && source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
```

The `.[dev]` extra is intentional. The live testnet flow depends on the current `injective-py` client stack; installing only the base package can leave you with an older resolver result that cannot run the chain client.

**TypeScript** — install the Injective SDK:

```bash
npm install @injectivelabs/sdk-ts @injectivelabs/networks
```

---

## Minimal hello-world (Python)

Three steps: connect to the TakerStream, submit an RFQ request, and accept the best quote.

This example uses ACK-based correlation. The indexer assigns the real `rfq_id`; do not assume a locally generated timestamp is the same ID that makers quote against.

```python
import asyncio
import os
import time
import uuid
from decimal import Decimal
from rfq_test.clients.websocket import TakerStreamClient
from rfq_test.clients.contract import ContractClient
from rfq_test.config import get_environment_config
from rfq_test.crypto.wallet import Wallet
from rfq_test.models.types import Direction

PRIVATE_KEY = os.environ["TESTNET_RETAIL_PRIVATE_KEY"]

async def main():
    config = get_environment_config()  # reads RFQ_ENV from .env
    taker = Wallet.from_private_key(PRIVATE_KEY)

    # 1. Connect
    ws = TakerStreamClient(
        endpoint=config.indexer.ws_endpoint,
        request_address=taker.inj_address,
    )
    await ws.connect()

    try:
        # 2. Request quote; wait for the indexer-assigned rfq_id.
        expiry_ms = int(time.time() * 1000) + 5 * 60 * 1000
        ack = await ws.send_request({
            "request_address": taker.inj_address,
            "client_id": str(uuid.uuid4()),
            "market_id": config.default_market.id,
            "direction": "long",
            "margin": "200",
            "quantity": "100",
            "worst_price": "5",
            "expiry": expiry_ms,
        }, wait_for_response=True, response_timeout=5.0)
        rfq_id = int(ack["rfq_id"])

        # 3. Accept best quote
        quotes = await ws.collect_quotes(rfq_id=rfq_id, timeout=0.5, min_quotes=1)
        best = min(quotes, key=lambda q: float(q["price"]))

        contract = ContractClient(config.contract, config.chain)
        tx_hash = await contract.accept_quote(
            private_key=PRIVATE_KEY,
            quotes=[{
                "maker": best["maker"],
                "margin": best["margin"],
                "quantity": best["quantity"],
                "price": best["price"],
                "expiry": int(best["expiry"]),
                "signature": best["signature"],
            }],
            rfq_id=str(rfq_id),
            market_id=config.default_market.id,
            direction=Direction.LONG,
            margin=Decimal("200"),
            quantity=Decimal("100"),
            worst_price=Decimal("5"),
            unfilled_action={"market": {}},
        )
        print(f"Settled: {tx_hash}")
    finally:
        await ws.close()

asyncio.run(main())
```

Before this runs, you must complete [authz setup](/takers/authz-setup) once per wallet. See [Quickstart](/takers/quickstart) for the full annotated walkthrough including environment config, TypeScript equivalents, error handling, and the complete `AcceptQuote` message structure.

---

## SDK helpers (Python — `rfq_test` package)

| Helper | Module | Description |
|---|---|---|
| `Wallet.from_private_key` | `rfq_test.crypto.wallet` | Load a wallet from a raw secp256k1 private key; exposes `inj_address` and signing methods |
| `TakerStreamClient` | `rfq_test.clients.websocket` | WebSocket client for the TakerStream gRPC endpoint; handles gRPC-web framing, reconnection, and ACK-based `rfq_id` correlation |
| `ContractClient` | `rfq_test.clients.contract` | High-level client for submitting `AcceptQuote` and `AcceptSignedIntent` to the TrueCurrent contract; handles signature encoding (hex → base64) and expiry wrapping |
| `ChainClient` | `rfq_test.clients.chain` | Low-level Injective chain client for broadcasting transactions and querying state |
| `setup_authz_grants` | `rfq_test.utils.setup` | One-shot authz grant setup; submits all four required `MsgGrant` transactions for a taker wallet |
| `RETAIL_AUTHZ_GRANTS` | `rfq_test.utils.setup` | The four message types required for taker authz (bank send, privileged execute, batch update orders, create derivative market order) |
| `get_environment_config` | `rfq_test.config` | Reads `RFQ_ENV` from `.env` and returns the full config object (endpoints, contract address, market IDs) |
| `Direction` | `rfq_test.models.types` | Enum: `Direction.LONG`, `Direction.SHORT` — use when calling `ContractClient.accept_quote` |

## SDK helpers (TypeScript — `@injectivelabs/sdk-ts`)

TrueCurrent does not yet have a TypeScript-specific taker wrapper. The Injective SDK provides the building blocks:

| Helper | Package | Description |
|---|---|---|
| `MsgExecuteContractCompat` | `@injectivelabs/sdk-ts` | Builds a CosmWasm execute message for `AcceptQuote` and `AcceptSignedIntent`; handles Amino/Direct encoding |
| `MsgBroadcasterWithPk` | `@injectivelabs/sdk-ts` | Signs and broadcasts a transaction from a private key; handles sequence management |
| `Network` | `@injectivelabs/networks` | Enum for `Network.TestnetSentry`, `Network.MainnetSentry`; used by `MsgBroadcasterWithPk` |

See [Quickstart](/takers/quickstart) for a complete TypeScript `AcceptQuote` example with all encoding gotchas documented.

---

## Environment configuration

The SDK reads connection parameters from a `.env` file at the repo root. Minimum testnet keys:

```bash
RFQ_ENV=testnet
TESTNET_RETAIL_PRIVATE_KEY=0x...   # taker secp256k1 private key
```

The `configs/testnet.yaml` file in the repo pre-populates all testnet endpoints and the current testnet contract:

```text
RFQ contract: inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk
Cosmos chain ID: injective-888
EIP-712 chainId: 1439
```

Do not override the contract address or chain IDs unless you are intentionally pointing at a different deployment. See the [Quickstart config table](/takers/quickstart#2-configure-environment) for the full list of testnet values.

---

## Related pages

- [Quickstart](/takers/quickstart) — full lifecycle walkthrough with annotated code
- [Authorization setup](/takers/authz-setup) — one-time authz grants required before any trade
- [TakerStream](/takers/taker-stream) — request schema, quote collection, and rfq_id correlation
- [Accepting quotes](/takers/accepting-quotes) — `AcceptQuote` message structure, multi-quote aggregation, encoding requirements
- [Signed taker intents](/takers/signed-intents) — programmatic TP/SL and conditional orders
- [Reference implementations (MM)](/market-makers/integration/reference-implementations) — market maker side SDK and Python/TypeScript reference clients
