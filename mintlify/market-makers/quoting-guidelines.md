---
title: "Quoting guidelines"
description: "Best practices for TrueCurrent market makers covering spread management, volatility-based pricing, inventory hedging, response rate optimization, and avoiding common pitfalls like stale prices and signature errors."
updatedAt: "2026-04-30"
---

This page covers best practices for market makers on TrueCurrent. Following these guidelines will improve your fill rate, protect you from adverse selection, and keep your market maker standing in good shape.

---

## Pricing fundamentals

**Always reference a reliable mid-market price.** Your quote price should be derived from a real-time reference price – typically the best available mid from a liquid CEX, Injective's onchain oracle, or a composite of both. Never quote blind.

**Quote around mid, not just on one side.** Your spread should be symmetric around mid unless you have a deliberate inventory reason to skew. Systematically quoting wide on one side makes you easier to pick off.

**Respect the worst price.** There's no benefit to quoting worse than the taker's `worst_price` – you won't win the trade. Evaluating `worst_price` in your logic can help you decide whether to quote at all for requests with very tight limits.

---

## Spread management

**Scale spread to volatility.** Your spread represents the compensation for the risk you take by being on the other side of a trade. In low-volatility periods, tight spreads win more flow. In high-volatility periods, widen accordingly – the risk of being picked off is higher.

A simple starting model:

```
spread = base_spread + volatility_factor × realized_vol_30s
```

**Scale spread to order size.** Larger orders expose you to more market impact when you hedge. A basic model:

```
spread = base_spread × (1 + size_factor × quantity / reference_quantity)
```

**Account for your inventory.** If you're net long INJ from previous fills, you're already exposed to downside. When the next request is for a taker long (you'd go shorter), you may want to tighten your spread to encourage the fill. For a taker short (you'd go longer), widen slightly.

### Numerical spread construction example

Below is a worked example of building a competitive two-sided quote for an INJ/USDC RFQ.

**Inputs:**
- Mid-market: $25.00
- 30-second realised volatility: 0.08% (annualised ~40%; intraday spike)
- Request size: 500 INJ (~$12,500 notional)
- Reference size: 100 INJ
- Current inventory: +300 INJ net long (skew toward selling)

**Step 1 — Base spread:** 2 bps = $0.005 per INJ (0.02% of mid)

**Step 2 — Volatility adjustment:** 0.08% × 0.5 = +0.04% → add 2 bps each side → spread = 4 bps each side

**Step 3 — Size adjustment:** factor = 0.5, size ratio = 500/100 = 5 → `1 + 0.5 × (5-1)` = 3× → spread = 4 bps × 3 = 12 bps each side

**Step 4 — Inventory skew:** +300 INJ net long → tighten offer (selling), widen bid (buying). Apply −2 bps to offer, +2 bps to bid.

**Final quote:**
- Bid: $25.00 × (1 − 0.0014) = **$24.965**
- Offer: $25.00 × (1 + 0.0010) = **$25.025**
- Half-spread: 10 bps bid / 10 bps offer (asymmetric due to inventory skew)

**4 bps maker fee impact:** Filling the $12,500 request costs $5.00 in maker fees. The minimum spread to break even on fees alone is `$5.00 / $12,500 = 4 bps` each side. The 10 bps half-spread above earns $12.50 gross spread minus $5.00 fee = **$7.50 net spread income**, before hedging costs.

---

## Response rate

**Always try to quote within 2 seconds.** A consistent response rate is important both for your standing and for the quality of the RFQ system. Traders who request quotes and receive none fall back to the order book – a worse experience for them.

**Fail gracefully.** If you genuinely can't price a request (e.g., your pricing feed is down, your margin is depleted), it's better to not submit a quote than to submit a random price. The system handles no-quote gracefully by falling back to the order book.

**Monitor latency.** The 2-second window includes network round-trip time. If your system is co-located far from the indexer, account for that latency. A quote that arrives after the window is ignored.

---

## Performance metrics

TrueCurrent monitors market maker performance continuously. Understanding these metrics lets you self-monitor your standing and take corrective action before reaching review thresholds.

### Metric definitions

**Response rate**

$$\text{Response rate} = \frac{\text{Quotes submitted within the 2 s window}}{\text{RFQ requests received}} \times 100\%$$

Measured over a rolling 24-hour window. A response is counted only if the quote arrives within the 2-second deadline; late quotes are treated as no-quote.

**Price competitiveness**

The percentage of your submitted quotes that are selected as the best price across all competing market makers for the same RFQ. Measured over the same rolling 24-hour window. A low competitiveness score indicates your spreads are systematically wider than competitors — review your spread model.

**Settlement success rate**

$$\text{Settlement success rate} = \frac{\text{Quotes accepted by takers that settle successfully}}{\text{Quotes accepted by takers}} \times 100\%$$

Settlement failures occur when your subaccount has insufficient margin at the time the taker accepts your quote, or when a signature validation error occurs. A low settlement success rate is a serious standing issue.

### Threshold table

| Metric | Green (healthy) | Amber (monitor) | Red (review risk) |
|--------|----------------|-----------------|-------------------|
| Response rate | ≥ 80% | 60% – 79% | < 60% |
| Price competitiveness | ≥ 20% | 10% – 19% | < 10% |
| Settlement success rate | ≥ 98% | 95% – 97% | < 95% |

**Amber:** No immediate action from TrueCurrent, but you should investigate the cause. Sustained amber performance may trigger a standing review.

**Red:** TrueCurrent will initiate a standing review. Persistent red performance in any metric, particularly settlement success rate, may result in temporary whitelist suspension until the issue is resolved.

{/*
TODO replace with actual API endpoint

### Self-querying your metrics

You can retrieve your own performance metrics via the TrueCurrent indexer API:

```
GET (... API endpoint URL)
```

This endpoint returns response rate, fill rate, and settlement success rate for the specified window. Build a monitoring dashboard or alert to track these in real time.
*/}
---

## Inventory and hedging

**Hedge promptly.** When you fill a taker long, you take on a short perpetuals position. Your goal as a market maker is typically to be market-neutral – capture the spread while hedging away directional risk.

Common hedging approaches:
- Open an offsetting spot position on a CEX immediately after settlement confirms
- Hedge via another derivatives venue
- Allow inventory to build within limits and hedge periodically

**Set inventory limits.** Don't let any single position grow large enough to threaten your capital. Define hard limits on net delta per market and stop quoting (or skew heavily) when those limits are approached.

### Oracle basis risk

TrueCurrent's mark price (and the reference mid most makers use) is derived from the Injective oracle, which updates on each block (~1–2 seconds). Your pricing model likely consumes a CEX mid that updates faster (milliseconds). The gap between your reference price and the oracle means:

- When a major CEX price move occurs, there is a brief window before the oracle updates during which informed takers can pick off stale quotes at the pre-move oracle mid.
- This is the primary source of adverse selection for market makers on oracle-based systems.

**Mitigations:**
1. **Widen spread during oracle lag.** When your CEX feed shows a fast move (e.g., >0.1% in 1 second), temporarily widen or pause quoting until the oracle has had time to catch up (1–2 block times).
2. **Track block times.** If block times lengthen (e.g., during network congestion), oracle updates slow down and your oracle-lag risk increases.
3. **Quote against the oracle, not just the CEX mid.** Incorporate the current oracle price into your reference mid. If CEX mid and oracle diverge by more than your spread, either pause or price relative to the oracle to avoid selling taker longs at a price that will immediately be underwater when the oracle catches up.

---

## Replicating the index price

To minimise adverse selection, market makers should construct the same reference price that TrueCurrent uses — rather than relying on a single CEX or a different composite.

TrueCurrent's index price is derived from the **Injective oracle module**, which aggregates from sources including major centralised exchanges and onchain oracle providers. For most markets, the primary source CEXes are large-volume venues trading the same asset.

**To replicate:**

1. Identify the source exchanges used for the relevant market's oracle feed. These are listed in the market's onchain parameters (queryable via the Injective exchange module).
2. Subscribe to the top-of-book or trade feed from each source.
3. Compute a median (or equal-weight average) of the best bids and asks from each source.
4. Use this composite as your reference mid.

The closer your reference price is to TrueCurrent's oracle, the smaller the basis between your quote and the oracle-derived mark price — and the less adverse selection you experience from oracle-aware takers.

---

## Funding rate carry as a quoting cost component

Funding rate carry affects your cost basis on open positions and should be factored into your quote pricing.

**The mechanic:**

- **Positive funding** (perpetual trading at premium): longs pay shorts. If you fill a taker long, you take on a short position and **receive** funding.
- **Negative funding** (perpetual trading at discount): shorts pay longs. If you fill a taker long, you take on a short and **pay** funding.

For market makers who hold positions for hours before hedging fully (or who maintain a residual book), funding carry is a real P&L component.

**Incorporating carry into your fair value:**

```
fair_value_mid = oracle_mid + funding_carry_adjustment

funding_carry_adjustment = −(hourly_funding_rate × expected_hold_hours × notional)
```

If funding is +0.005% per hour and you expect to hold a short hedge for ~4 hours, the carry benefit is `0.005% × 4 = 0.02%` of notional. On a $10,000 position, that is $2.00 of carry income — which can be reflected as a 2 bps tighter offer when positive funding makes short positions attractive.

Conversely, in a negative funding regime, shorts pay longs. If you are net short (as a maker filling taker longs), negative funding is a cost that should be reflected by widening your offer.

**Practical guideline:** At hourly funding rates below ±0.002%, carry is immaterial for typical hold periods. At rates above ±0.010%, carry is worth incorporating explicitly into your pricing model, especially for large notional positions.

---

## Common pitfalls

**Quoting stale prices.** If your reference price feed has a lag, you're quoting old prices into a market that may have moved. This is the primary source of adverse selection for market makers. Use low-latency price feeds and monitor feed freshness.

**Insufficient margin at settlement.** If your subaccount doesn't have enough margin when a taker accepts your quote, the settlement transaction fails. Monitor your available margin and stop quoting when you're running low.

**Clock skew.** The quote expiry is verified onchain using block time. If your system clock is off by more than a few seconds, you may produce quotes that appear expired to the chain even though they look valid to you. Use NTP synchronization.

**Signature field order errors.** The most common cause of settlement failures for new market makers. Test thoroughly on testnet and verify the exact serialization format against the contract's expectations before going live.
