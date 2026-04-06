---
title: "Quoting guidelines"
description: "Best practices for TrueCurrent market makers covering spread management, volatility-based pricing, inventory hedging, response rate optimization, and avoiding common pitfalls like stale prices and signature errors."
updatedAt: "2026-04-06"
---

This page covers best practices for market makers on TrueCurrent. Following these guidelines will improve your fill rate, protect you from adverse selection, and keep your market maker standing in good shape.

---

## Pricing fundamentals

**Always reference a reliable mid-market price.** Your quote price should be derived from a real-time reference price – typically the best available mid from a liquid CEX (e.g., Binance INJ/USDT), Injective's onchain oracle, or a composite of both. Never quote blind.

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

---

## Response rate

**Always try to quote within 2 seconds.** A consistent response rate is important both for your standing and for the quality of the RFQ system. Traders who request quotes and receive none fall back to the order book – a worse experience for them.

**Fail gracefully.** If you genuinely can't price a request (e.g., your pricing feed is down, your margin is depleted), it's better to not submit a quote than to submit a random price. The system handles no-quote gracefully by falling back to the order book.

**Monitor latency.** The 2-second window includes network round-trip time. If your system is co-located far from the indexer, account for that latency. A quote that arrives after the window is ignored.

---

## Inventory and hedging

**Hedge promptly.** When you fill a taker long, you take on a short perpetuals position. Your goal as a market maker is typically to be market-neutral – capture the spread while hedging away directional risk.

Common hedging approaches:
- Open an offsetting spot position on a CEX immediately after settlement confirms
- Hedge via another derivatives venue
- Allow inventory to build within limits and hedge periodically

**Set inventory limits.** Don't let any single position grow large enough to threaten your capital. Define hard limits on net delta per market and stop quoting (or skew heavily) when those limits are approached.

---

## Common pitfalls

**Quoting stale prices.** If your reference price feed has a lag, you're quoting old prices into a market that may have moved. This is the primary source of adverse selection for market makers. Use low-latency price feeds and monitor feed freshness.

**Insufficient margin at settlement.** If your subaccount doesn't have enough margin when a taker accepts your quote, the settlement transaction fails. Monitor your available margin and stop quoting when you're running low.

**Clock skew.** The quote expiry is verified onchain using block time. If your system clock is off by more than a few seconds, you may produce quotes that appear expired to the chain even though they look valid to you. Use NTP synchronization.

**Signature field order errors.** The most common cause of settlement failures for new market makers. Test thoroughly on testnet and verify the exact serialization format against the contract's expectations before going live.
