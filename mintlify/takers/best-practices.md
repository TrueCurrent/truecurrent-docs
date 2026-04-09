---
title: "Best practices"
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

**For arbitrage** takers where the edge is known: set `worst_price` at your breakeven against the hedge venue. Any quote worse than that loses money – you'd rather not fill.

**Too tight** and you'll get rejected often in fast markets. **Too wide** and you're defenseless against adverse selection. Measure your fill rate and P&L distribution and tune.

---

## Expiry races

Quotes carry an `expiry` that's guaranteed valid for a short window from signing. You must submit `AcceptQuote` and have it confirm **before** that timestamp passes. The onchain check is `block_time_ms > expiry → reject`.

{/* TODO: CK to add updated expiry info after benchmarking */}

**What eats the budget:**

- Market maker response (a few hundred milliseconds)
- Your collection window (whatever's left after MM response)
- Quote selection and message building (~100ms)
- Network RTT to the chain gRPC endpoint (~50–200ms)
- Injective block time (~600ms average)
- Any retries you do on broadcast failure

The quote lifetime is short. By the time you've waited for makers to respond and a block to confirm, there is very little headroom. If you're running across regions, mempool contention is high, or you have backoff retries on the broadcast path, you can burn it.

**Mitigations:**

- **Submit immediately after the collection window closes.** Don't batch or wait.
- **Use short collection windows.** A few hundred milliseconds is typically enough once your maker set is warm – waiting longer just eats into the confirmation budget.
- **Co-locate** near `testnet-grpc.injective.dev` / `sentry.tm.injective.network` to minimize RTT.
{/* TODO: CK to add mainnet info ; also should we say colocation in docs ? */}
- **Don't retry failed broadcasts** on an expiry error – the quote is dead, get a new one.
- **Use `cid`** to echo your own trade ID in the settlement event – handy when debugging which quote won the race.

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

**Top up preemptively** – a taker that bounces on insufficient funds burns quotes and contributes to its own adverse selection. Run a monitor that tops up when balance falls below a threshold.

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

- **Partial fills** – some quotes consumed, others skipped, `fallback_quantity` remains
- **Zero fills** – every quote rejected, whole transaction fails

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

Track per-maker rejection reasons over time – a maker that consistently rejects with "insufficient balance" or "nonce replay" is broken, and you can down-rank them in your quote selection.

---

## Selection strategies

For longs, lowest price wins. For shorts, highest price wins. Beyond that, you have choices:

**Single-quote, cheapest:** Pick exactly one quote – the cheapest that covers your full requested `quantity`. Simplest, lowest variance, sometimes leaves quantity unfilled if no single maker is big enough.

**Multi-quote, sorted:** Sort by price ascending (longs) / descending (shorts), greedily consume until your `quantity` is covered, submit all. Best average price, larger message, slightly higher rejection risk (one bad quote poisons nothing but the transaction takes longer).

**Multi-quote, price-capped:** Same as above, but drop any quote worse than your `worst_price` before submitting. Reduces onchain work and improves settlement latency.

**Multi-quote, maker-scored:** Weight makers by historical fill reliability. Prefer makers that reliably settle even if their price is slightly worse. Worth it for high-frequency takers where the fill rate dominates the price delta.

**First-fit:** Accept the first quote that's inside `worst_price` and covers enough quantity. Lowest latency, worst execution – use only when latency is the primary metric.

---

## Reconnection and liveness

The TakerStream will drop you eventually – maintenance, network blips, indexer restarts. Your taker needs to survive it.

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

**Heartbeat the chain side too.** Your taker should fail-fast if the gRPC endpoint to Injective itself is unreachable – there's no point in collecting quotes you can't settle.

