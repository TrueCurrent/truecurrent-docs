---
title: "Takers: best practices"
description: "Production guidance for programmatic RFQ takers on TrueCurrent, covering slippage and worst_price strategy, quote expiry races, subaccount balance requirements, idempotency, reconnection, error handling, and safe testnet-to-mainnet progression."
updatedAt: "2026-04-08"
---

This page is the collected wisdom for running a programmatic taker in production. If you've followed [Quickstart](/takers/quickstart), this is what saves you when something goes wrong at 3am.

---

## Slippage and `worst_price`

`worst_price` is your hard price limit. The contract rejects any individual quote worse than it, and `unfilled_action: {"market": {}}` won't cross it either. You must set it correctly.

**For longs** (you're buying): `worst_price` is the **maximum** price you'll accept. Set it *above* your expected fill.

**For shorts** (you're selling): `worst_price` is the **minimum** price you'll accept. Set it *below* your expected fill.

**A sensible formula** for a non-arb taker:

```
worst_price = mark_price * (1 ± max_slippage_bps / 10000)
```

Where `+` is for longs and `−` is for shorts. Start with `max_slippage_bps = 50` (0.5%) for liquid markets and widen for illiquid ones.

**For arbitrage** takers where the edge is known: set `worst_price` at your breakeven against the hedge venue. Any quote worse than that loses money — you'd rather not fill.

**Too tight** and you'll get rejected often in fast markets. **Too wide** and you're defenseless against adverse selection. Measure your fill rate and P&L distribution and tune.

---

## Expiry races

Quotes carry an `expiry` and are guaranteed valid for **at least 1.5 seconds** from signing. You must submit `AcceptQuote` and have it confirm **before** that timestamp passes. The onchain check is `block_time_ms > expiry → reject`.

{/* TODO: CK to add updated expiry info after benchmarking */}

**What eats the budget:**

- Market maker response (a few hundred milliseconds)
- Your collection window (whatever's left after MM response)
- Quote selection and message building (~100ms)
- Network RTT to the chain gRPC endpoint (~50–200ms)
- Injective block time (~600ms average)
- Any retries you do on broadcast failure

With a 1.5-second minimum quote lifetime, the budget is tight. By the time you've waited for makers to respond and a block to confirm, there is very little headroom. If you're running across regions, mempool contention is high, or you have backoff retries on the broadcast path, you can burn it.

**Mitigations:**

- **Submit immediately after the collection window closes.** Don't batch or wait.
- **Use short collection windows.** A few hundred milliseconds is typically enough once your maker set is warm — waiting longer just eats into the confirmation budget.
- **Co-locate** near `testnet-grpc.injective.dev` / `sentry.tm.injective.network` to minimize RTT.
{/* TODO: CK to add mainnet info ; also should we say colocation in docs ? */}
- **Don't retry failed broadcasts** on an expiry error — the quote is dead, get a new one.
- **Use `cid`** to echo your own trade ID in the settlement event — handy when debugging which quote won the race.

---

## Subaccount balance

Takers trade from their Injective **exchange subaccount**, not the main bank module. Before any trade, you need USDC sitting in subaccount index 0 (or whichever `subaccount_nonce` you passed).

**Deposit USDC to the subaccount:**

{/* TODO: the denom and comments below still reference the real testnet USDT Peggy contract because a USDC denom does not yet exist on injective-888. Replace both amount comment and denom hex once USDC is bridged to testnet. */}

```python
from pyinjective.composer_v2 import Composer

composer = Composer(network="testnet")
deposit_msg = composer.msg_deposit(
    sender=taker_inj_address,
    subaccount_id=f"{taker_eth_address}000000000000000000000000",
    amount=10_000_000_000,  # 10,000 USDT in peggy 6-decimal units
    denom="peggy0x87aB3B4C8661e07D6372361211B96ed4Dc36B1B5",  # testnet USDT
)
await broadcaster.broadcast([deposit_msg])
```

**Check your subaccount balance** before submitting requests. If it's below your requested margin, every `AcceptQuote` will fail with `insufficient balance`.

**Top up preemptively** — a taker that bounces on insufficient funds burns quotes and contributes to its own adverse selection. Run a monitor that tops up when balance falls below a threshold.

---

## Idempotency and `rfq_id` reuse

The contract treats `rfq_id` as a per-taker nonce. Reusing the same `rfq_id` for two different `AcceptQuote` calls from the same taker causes a nonce replay error on the second.

**Generate fresh IDs:**

```python
rfq_id = int(time.time() * 1_000_000)   # microsecond precision
```

Microsecond timestamps are collision-free for a single process. If you run multiple concurrent takers from one wallet, use a shared counter or UUID hash.

**Do not retry a failed `AcceptQuote` with the same `rfq_id`.** If a submission fails mid-flight (broadcast error, timeout, chain rejection), request fresh quotes with a new `rfq_id`. The original quotes are probably also expired or consumed.

---

## Handling partial and rejected quotes

When you submit a multi-quote `AcceptQuote`, the contract processes quotes in order and **skips** quotes that fail validation rather than aborting the whole transaction (see [Accepting quotes](/takers/accepting-quotes)).

This means you can get:

- **Partial fills** — some quotes consumed, others skipped, `fallback_quantity` remains
- **Zero fills** — every quote rejected, whole transaction fails

**Parse the settlement event** on success to know what actually happened:

```python
tx = await chain.get_tx(tx_hash)
for event in tx.events:
    if event.type == "wasm-accept_quote":
        for attr in event.attributes:
            if attr.key == "quote_results":
                results = json.loads(attr.value)
                for r in results:
                    if "e" in r:
                        print(f"Quote from {r['maker']} rejected: {r['e']}")
                    else:
                        print(f"Quote from {r['maker']} filled: qty={r['q']}")
```

Track per-maker rejection reasons over time — a maker that consistently rejects with "insufficient balance" or "nonce replay" is broken, and you can down-rank them in your quote selection.

---

## Selection strategies

For longs, lowest price wins. For shorts, highest price wins. Beyond that, you have choices:

**Single-quote, cheapest:** Pick exactly one quote — the cheapest that covers your full requested `quantity`. Simplest, lowest variance, sometimes leaves quantity unfilled if no single maker is big enough.

**Multi-quote, sorted:** Sort by price ascending (longs) / descending (shorts), greedily consume until your `quantity` is covered, submit all. Best average price, larger message, slightly higher rejection risk (one bad quote poisons nothing but the transaction takes longer).

**Multi-quote, price-capped:** Same as above, but drop any quote worse than your `worst_price` before submitting. Reduces onchain work and improves settlement latency.

**Multi-quote, maker-scored:** Weight makers by historical fill reliability. Prefer makers that reliably settle even if their price is slightly worse. Worth it for high-frequency takers where the fill rate dominates the price delta.

**First-fit:** Accept the first quote that's inside `worst_price` and covers enough quantity. Lowest latency, worst execution — use only when latency is the primary metric.

---

## Reconnection and liveness

The TakerStream will drop you eventually — maintenance, network blips, indexer restarts. Your taker needs to survive it.

**Minimum reconnect loop:**

```python
async def run_with_retries():
    while True:
        try:
            await taker_ws.connect()
            await taker_ws.run()
        except (ConnectionError, asyncio.TimeoutError) as e:
            logger.warning(f"Stream dropped: {e}")
            await asyncio.sleep(1.0)
```

**Better:** two connections, fail over on the first dropped frame. Cold reconnection on a single WS means you're blind for 1–3 seconds, which in a fast market is too long.

**Monitor staleness.** If you've sent a request and received zero quotes in your window, that's either a dead stream or a dead maker set. Alert on repeated empty windows.

**Heartbeat the chain side too.** Your taker should fail-fast if the gRPC endpoint to Injective itself is unreachable — there's no point in collecting quotes you can't settle.

---

## Rate limiting

### Today

There is **no rate limiting** on the testnet RFQ indexer. The reference scripts in `rfq-testing` connect directly and run as fast as the network and your code allow. You will not get throttled.

### Once the RFQ Gateway is deployed

The [RFQ Gateway](https://github.com/InjectiveLabs/rfq-gateway) will sit in front of the indexer and apply token-bucket rate limiting per API key. The relevant facts for programmatic takers:

| Knob | Default | Notes |
|---|---|---|
| `GATEWAY_DEFAULT_RATE_LIMIT` | `10` req/s | Per key, refill rate |
| `GATEWAY_DEFAULT_BURST_SIZE` | `20` | Per key, bucket capacity |
| `api`-tier custom rate | typically `1000` req/s, burst `2000` | What the gateway README's example HFT key uses |

**WebSocket-specific behavior:** rate limits only consume tokens on **write** methods. Read methods are unlimited. From `rfq-gateway/internal/proxy/websocket.go`:

```go
var writeMethods = map[string]bool{
    "create": true,    // taker submitting an RFQ
    "quote":  true,    // maker submitting a quote
}
```

So `subscribe`/`unsubscribe` (passively listening to the quote stream) are free. Each new RFQ submission (`create`) consumes one token from your taker's bucket. A 1000 req/s `api` key gets you ~1000 RFQs/sec sustained.

**Failure modes:**

| Transport | Throttled response |
|---|---|
| HTTP/REST | `429 Too Many Requests` with `Retry-After: 1` header and `{"error":"rate limit exceeded"}` body |
| gRPC | `ResourceExhausted` status code |
| gRPC-Web | `429 Too Many Requests` |
| WebSocket | JSON-RPC error response on the offending `create` message; the connection stays open |

Your taker should treat any of these as a transient error: back off briefly and retry on the next request, but **never** retry the same `rfq_id` (use a fresh one) and **don't** retry an `AcceptQuote` that's already in flight.

**Settlement is unaffected.** `AcceptQuote` calls go directly to the smart contract on Injective, not through the gateway. The gateway's rate limit only governs quote *discovery* (RFQ submission and quote streaming). On-chain settlement is bounded only by Injective's block time and your wallet's sequence number ordering.

### What HFT takers should ask for

When you request an `api`-tier key from the TrueCurrent team, you'll need to specify:

- **Sustained req/s** — your expected RFQ submission rate at peak
- **Burst** — short-burst capacity (burst should be roughly 2× sustained)
- **Source IP or origin range** — for additional gating, if applicable
- **Use case** — arb, market neutral, integrator, etc.

Don't accept the default 10 req/s — it will throttle you on the second request of any reasonable test run.

{/* TODO: CK — once the gateway is deployed in front of the public indexer:
     - Document the actual key issuance process (who to contact, expected turnaround)
     - Confirm whether the `api` tier is provisioned and what rate limits are typical
     - Add code samples showing 429 / ResourceExhausted / WS JSON-RPC error handling for the `create` method
     - Confirm whether multi-region clients need separate keys per region (token bucket is in-memory per gateway instance) */}

---

## Testing progression

Do not take your taker to mainnet until it has successfully handled, on testnet:

1. **Happy path** — single quote, full fill
2. **Multi-quote** — three different makers, aggregated fill
3. **Partial fill** — requested quantity exceeds total maker quantity, `unfilled_action` exercised
4. **Stale quote** — quote expiry passes before submission; confirm transaction fails cleanly
5. **Bad worst_price** — submit a quote that exceeds `worst_price`; confirm it's skipped with the right error
6. **Missing grant** — drop one authz grant, attempt a settlement, confirm the `unauthorized` error, re-grant, confirm recovery
7. **Stream disconnect** — pull network mid-collection, confirm your reconnection logic works and you don't reuse `rfq_id`
8. **Insufficient balance** — drain the subaccount, attempt a settlement, confirm the right error
9. **Concurrent RFQs** — multiple in-flight requests on one connection; confirm routing is correct
10. **Broadcast failure retry** — kill the chain gRPC mid-broadcast, confirm you don't accidentally double-spend
11. **Rate limit (when gateway is live)** — exceed your `api`-tier key's req/s; confirm you handle the `429` / `ResourceExhausted` / WebSocket JSON-RPC error correctly without retrying the same `rfq_id` or double-submitting an `AcceptQuote` {/* TODO: CK — promote this from optional to required once the gateway is live */}

The [`rfq-testing`](https://github.com/InjectiveLabs/rfq-testing) examples exercise the happy and multi-quote paths out of the box. The rest is on you.

---

## Monitoring metrics worth tracking

| Metric | Why it matters |
|---|---|
| **Fill rate** | Quotes accepted / quotes submitted. A dropping fill rate means your `worst_price` is too tight or makers are rejecting your requests. |
| **Median time to fill** | From request sent to `AcceptQuote` confirmed. Tracks end-to-end latency. |
| **Quote count per request** | How many makers are quoting you. Falling = degraded liquidity. |
| **Price improvement vs. mark** | `(mark - entry) / mark` on longs. Negative = adverse selection. |
| **Rejected-quote reasons** | From settlement events. Frequent `insufficient maker balance` is a broken maker; frequent `expired` is a timing problem on your side. |
| **Subaccount balance** | Keep a runway of at least 10× your typical trade margin. |
| **Stream gap duration** | Longest gap between WS messages. Spikes indicate flaky connectivity. |

---

## Security hygiene

- **Use dedicated wallets** for automated taking. Never grant authz from a treasury, cold wallet, or human trading account.
- **Rotate private keys** on a schedule if your keys ever touch a long-running server.
- **Monitor authz grants** — a surprise revocation (by you or an attacker) kills trading; detect and alert.
- **Rate-limit your own traffic** at the application level. Making yourself expensive to serve doesn't help you.
- **Treat the settlement event as source of truth** — client-side optimism is how you think you have a position that doesn't exist.
