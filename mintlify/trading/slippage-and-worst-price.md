---
title: "Price tolerance"
description: "Set the worst price you are willing to accept on TrueCurrent and learn how the price tolerance setting protects RFQ execution from slippage and bad fills."
updatedAt: "2026-05-06"
---

When you place a trade on TrueCurrent, you set a **maximum slippage** – from which a worst-case price you're willing to accept is derived. TrueCurrent will never execute your trade at a price less favorable than this limit.

---

## How it works

Before confirming a trade, you'll see an **estimated price** based on current market conditions. You also set a **worst price**: the highest price you will pay when going long, or the lowest price you will receive when going short.

TrueCurrent automatically looks for the best available price from liquidity providers.
If none can meet your worst price, the trade is rejected, your margin is returned, and you can try again.

- For a **long** position: your worst price is the highest you're willing to pay
- For a **short** position: your worst price is the lowest you're willing to receive

---

## How to use it

Price tolerance controls how far from the quoted price your trade can fill.
You set it each time you open or close a position.
It then acts as a hard, on-chain enforced limit.

- **Default**:
  0.5% is applied automatically when you open the trade form.
- **Normal conditions**:
  0.5% is appropriate for most markets and sizes.
  It gives enough room for minor price movement between quote and execution
  without exposing you to large adverse fills.
- **Volatile markets**:
  Widen to 1-2% if prices are moving fast and you are getting non-fills.
  A wider tolerance improves your chance of filling
  but raises the worst-case price you could receive.
- **Where to set it**:
  In the trade form, adjust the slippage field before clicking on *trade*.

<Info>
See [Price tolerance](/trading/how-to-trade)
for where it appears in the full trading workflow.
</Info>

---

## What happens if your order can't fill?

If no liquidity provider can beat your worst price, the order is cancelled and your margin is returned. Prices update continuously – you can re-submit immediately.

Setting too tight a tolerance increases the chance of non-fills. Setting it wider gives you a better fill rate but with a less favorable worst-case price.

---

## Your worst price is always enforced

The trade is executed automatically by TrueCurrent – you don't review the final price before it goes through. But your worst price limit is enforced onchain. The trade can never settle at a price worse than what you specified.
