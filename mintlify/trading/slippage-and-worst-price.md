---
title: "Price tolerance"
description: "--"
updatedAt: "2026-04-06"
---

When you place a trade on TrueCurrent, you set a **price tolerance** – a worst-case price you're willing to accept. TrueCurrent will never execute your trade at a price less favorable than this limit.

---

## How it works

Before confirming a trade, you'll see an **estimated price** based on current market conditions. You also set a **worst price** – your floor (for longs) or ceiling (for shorts).

TrueCurrent then automatically finds the best available price from liquidity providers. If no provider can beat your worst price, the trade doesn't go through. Your margin is returned and you can try again.

- For a **long** position: your worst price is the highest you're willing to pay
- For a **short** position: your worst price is the lowest you're willing to receive

---

## Setting your tolerance

You can set price tolerance as a **percentage from mark price** (recommended) or as a specific price.

**Percentage (recommended):** TrueCurrent automatically calculates your worst price based on the current mark price at confirmation time. For example, with 1% tolerance on a long when an asset is at $4.50:

```
Worst price = $4.50 × 1.01 = $4.545
```

**Recommended settings:**

| Market conditions | Recommended tolerance |
|---|---|
| Normal | 0.5% – 1% |
| Moderate volatility | 1% – 2% |
| High volatility | 2% – 5% |

---

## What happens if your order can't fill?

If no liquidity provider can beat your worst price, the order is cancelled and your margin is returned. Prices update continuously – you can re-submit immediately.

Setting too tight a tolerance increases the chance of non-fills. Setting it wider gives you a better fill rate but with a less favorable worst-case price.

---

## Your worst price is always enforced

The trade is executed automatically by TrueCurrent – you don't review the final price before it goes through. But your worst price limit is enforced onchain. The trade can never settle at a price worse than what you specified.
