---
title: "Quoting guidelines"
description: "Best practices for TrueCurrent makers covering spread management, volatility-based pricing, inventory hedging, response rate optimization, and avoiding common pitfalls like stale prices and signature errors."
updatedAt: "2026-05-27"
---

This page covers best practices for makers on TrueCurrent. Following these guidelines will improve your fill rate, protect you from adverse selection, and keep your maker standing in good shape.

---

## Pricing fundamentals

**`worst_price` scale**: The v2 indexer endpoints return `worst_price` in human-readable scale (e.g. `"14.85"`), not 1e18-scaled integers. Reading it as a raw integer from a v1 endpoint (which would return `14850000000000000000`) will cause `worst price exceeds limit` rejections. Always use the v2 endpoint values directly.

**Scale spread to volatility**: Your spread represents the compensation for the risk you take by being on the other side of a trade. In low-volatility periods, tight spreads win more flow. In high-volatility periods, widen accordingly, as the risk of being picked off is higher.

A simple starting model:

```text
spread = base_spread + volatility_factor × realized_vol_30s
```

**Scale spread to order size.** Larger orders expose you to more market impact when you hedge. A basic model:

```text
spread = base_spread × (1 + size_factor × quantity / reference_quantity)
```

**Account for your inventory.** If you're net long INJ from previous fills, you're already exposed to downside. When the next request is for a taker long (you'd go shorter), you may want to tighten your spread to encourage the fill. For a taker short (you'd go longer), widen slightly.

### Numerical spread construction example

Below is a worked example of building a competitive two-sided quote for an INJ/USDC RFQ.

**Inputs**:

- Mid-market: \$25.00
- 30-second realised volatility: 0.08% (annualised ~40%; intraday spike)
- Request size: 500 INJ (~\$12,500 notional)
- Reference size: 100 INJ
- Current inventory: \+300 INJ net long (skew toward selling)

**Step 1 — Base spread**: 2 bps = \$0.005 per INJ (0.02% of mid)

**Step 2 — Volatility adjustment**: 0.08% × 0.5 = \+0.04% → add 2 bps each side → spread = 4 bps each side

**Step 3 — Size adjustment**: factor = 0.5, size ratio = 500/100 = 5 → `1 + 0.5 × (5-1)` = 3× → spread = 4 bps × 3 = 12 bps each side

**Step 4 — Inventory skew**: \+300 INJ net long → tighten offer (selling), widen bid (buying). Apply −2 bps to offer, \+2 bps to bid.

**Final quote**:

- Bid: \$25.00 × (1 − 0.0014) = **\$24.965**
- Offer: \$25.00 × (1 \+ 0.0010) = **\$25.025**
- Half-spread: 10 bps bid / 10 bps offer (asymmetric due to inventory skew)

**4 bps maker fee impact**: Filling the \$12,500 request costs \$5.00 in maker fees. The minimum spread to break even on fees alone is `$5.00 / $12,500 = 4 bps` each side. The 10 bps half-spread above earns \$12.50 gross spread minus \$5.00 fee = **\$7.50 net spread income**, before hedging costs.

---

## Response rate

**Always try to quote within 2 seconds.** A consistent response rate is important both for your standing and for the quality of the RFQ system. Traders who request quotes and receive none get their order cancelled – a worse experience for them and lost flow for the maker pool.

**Fail gracefully.** If you genuinely can't price a request (e.g. your pricing feed is down, your margin is depleted), it's better to not submit a quote than to submit one. The system handles no-quote correctly: the request expires and the taker is told to retry.

**Monitor latency.** The 2-second window includes network round-trip time. If your system is co-located far from the indexer, account for that latency. A quote that arrives after the window is ignored.

---

## Tracking the mark price

To minimize adverse selection and avoid rejected quotes, makers should price around the same reference the contract validates against: the current onchain mark price.

TrueCurrent's RFQ validation uses `mark_price` directly. The market's `mark_price` is the reference used for quote validation, `worst_price` validation, trigger evaluation, and liquidation checks. Position P&L display uses the streamed `indexPrice`, so do not use UI P&L or index price as the source for contract-facing quote bands.

**To track it:**

1. Query the relevant derivative market from the Injective exchange module.
2. Read the current `mark_price`.
3. Normalize price decimals and tick size before using it in quote logic.
4. Refresh continuously and treat stale or unavailable mark price data as a no-quote condition.

You can still use external venues, private feeds, and inventory models to calculate your own fair value. Just keep contract-facing validation centered on mark price: quotes too far from the current mark will be rejected even if your offchain model says they are fair.
