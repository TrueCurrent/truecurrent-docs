---
title: "WebSocket streams"
description: "Integration guide for TrueCurrent's TakerStream and MakerStream WebSocket APIs, including RFQ IDs, MakerStream authentication, quote payloads, and reconnect behavior."
updatedAt: "2026-05-27"
---

TrueCurrent's offchain coordination uses two bidirectional streams:

- **TakerStream** for takers to request quotes and receive maker responses.
- **MakerStream** for whitelisted makers to receive RFQs and submit signed quotes.

For a runnable maker/taker validation path, use the [Runbook](/sdk-trading/runbook).

---

## Market price updates

For market-data consumers, `indexPrice` is streamed with the same cadence and freshness expectations as `markPrice`. Use `indexPrice` as the primary source for displayed **Index Price** and unrealized P&L. Continue to use `markPrice` or onchain `mark_price` for contract-facing quote validation, trigger evaluation, liquidation risk, and funding logic.

---

## Endpoints

| Environment | WebSocket base | MakerStream | TakerStream |
| --- | --- | --- | --- |
| Testnet | `wss://testnet.rfq.ws.injective.network/injective_rfq_rpc.InjectiveRfqRPC` | append `/MakerStream` | append `/TakerStream` |
| Mainnet | Contact TrueCurrent | append `/MakerStream` | append `/TakerStream` |

The public service alias is `injective_rfq_rpc.InjectiveRfqRPC`. Do not replace it with an internal proto package name. The `injective-rfq-toolkit` clients store the base URL and append the stream path internally.

---

## TakerStream

TakerStream is used by traders or applications acting on their behalf.

Connection path:

```text
wss://<base>/TakerStream
```

Connect with the taker's Injective address as stream metadata. In the Python toolkit, pass it as `request_address` when constructing `TakerStreamClient`.

### Request

```json
{
  "client_id": "8f1c4b6e-0d7f-44ed-9f86-3fdfd381ba48",
  "market_id": "0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e",
  "direction": "long",
  "margin": "200",
  "quantity": "100",
  "worst_price": "4.70"
}
```

`client_id` is your correlation key. The indexer assigns the settlement `rfq_id` and returns it in the request ACK. Use that ACK-returned `rfq_id` for quote collection and settlement.

```python
taker_ws = TakerStreamClient(
    env.indexer.ws_endpoint,
    request_address=taker_wallet.inj_address,
    timeout=15.0,
)
await taker_ws.connect()

ack = await taker_ws.send_request(
    request_data,
    wait_for_response=True,
    response_timeout=10.0,
)
rfq_id = int(ack["rfq_id"])

quotes = await taker_ws.collect_quotes(
    rfq_id=rfq_id,
    timeout=0.5,
    min_quotes=1,
)
```

### Quote delivery

Quotes delivered to TakerStream are signed maker quotes:

```json
{
  "rfq_id": 1770848375348,
  "price": "4.45",
  "quantity": "100",
  "maker": "inj1maker...",
  "maker_subaccount_nonce": 0,
  "expiry": 1770848377348,
  "signature": "0x...",
  "sign_mode": "v2",
  "evm_chain_id": 1439
}
```

Collect for a short window, select one or more quotes, then submit `AcceptQuote`. TrueCurrent currently uses 500 ms; the value can vary by frontend and protocol configuration, and API takers can tune their own timeout. Waiting near the full quote expiry leaves little time for transaction inclusion.

---

## MakerStream

MakerStream is used by whitelisted makers. A maker must be registered, funded, and authorized before quotes can settle.

Connection path:

```text
wss://<base>/MakerStream
```

### Authentication challenge

After connection, the indexer sends a one-shot `MakerChallenge`. The maker signs a `StreamAuthChallenge` EIP-712 v2 message and replies with `MakerAuth`.

Until auth succeeds, the stream may remain open but no RFQ `request` messages are forwarded.

Using the reference client:

```python
mm_ws = MakerStreamClient(
    env.indexer.ws_endpoint,
    maker_address=maker_wallet.inj_address,
    subscribe_to_quotes_updates=True,
    subscribe_to_settlement_updates=True,
    auth_private_key=mm_private_key,
    auth_evm_chain_id=1439,
    auth_contract_address=contract_address,
    timeout=15.0,
)
await mm_ws.connect()
```

### Incoming messages

| Message | Meaning |
| --- | --- |
| `challenge` | Auth challenge that must be signed before requests arrive |
| `request` | RFQ from a taker, with indexer-assigned `rfq_id` |
| `quote_ack` | Indexer accepted or rejected a submitted quote |
| `quote_update` | Quote lifecycle update |
| `settlement_update` | Onchain settlement result for a quote |
| `ping` / `pong` | Stream liveness |

MakerStream can broadcast requests unrelated to the taker you are testing. Always filter by `rfq_id`, market, and taker before signing.

### Submitting quotes

```json
{
  "rfq_id": 1770848375348,
  "market_id": "0xdc70164d7120529c3cd84278c98df4151210c0447a65a2aab03459cf328de41e",
  "taker_direction": "long",
  "margin": "200",
  "quantity": "100",
  "price": "4.45",
  "expiry": 1770848377348,
  "maker": "inj1maker...",
  "maker_subaccount_nonce": 0,
  "taker": "inj1taker...",
  "signature": "0x...",
  "sign_mode": "v2",
  "evm_chain_id": 1439,
  "chain_id": "injective-888",
  "contract_address": "inj1qw7jk82hjvf79tnjykux6zacuh9gl0z0wl3ruk"
}
```

`sign_mode="v2"` and `evm_chain_id` are mandatory for the current signing path. `chain_id` is the Cosmos chain ID; keep it as `injective-888` on testnet. In camelCase clients, the same split is `evmChainId=1439` and `chainId="injective-888"`.

`quote_ack.status="success"` means the indexer accepted and routed the quote. It does not mean the taker accepted or filled it. Use `settlement_update` or onchain transaction state for fill confirmation.

---

## Reconnection

Both streams should reconnect with exponential backoff. During downtime, missed RFQs are not replayed; request a new quote as taker, or resume normal quoting as maker.

```python
import asyncio

for attempt in range(10):
    try:
        await ws.connect()
        await ws.run()
    except ConnectionError as exc:
        delay = min(30.0, 1.0 * (2 ** attempt))
        print(f"Disconnected: {exc}. Retrying in {delay}s")
        await asyncio.sleep(delay)
```

---

## Related pages

- [TakerStream](/sdk-trading/taker-stream)
- [MakerStream](/sdk-trading/maker-stream)
- [Building and signing quotes](/sdk-trading/signing-quotes)
- [Troubleshooting](/sdk-trading/troubleshooting)
