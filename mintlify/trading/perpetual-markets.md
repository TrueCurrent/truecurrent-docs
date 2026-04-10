---
title: "Perpetual markets"
description: "Introduction to perpetual futures trading on TrueCurrent covering leverage, long and short positions, mark pricing, cross-margin, funding rates, liquidation mechanics, and oracle price feeds."
updatedAt: "2026-04-06"
---

TrueCurrent offers perpetual futures – leveraged positions on asset prices with no expiration date.

---

## What is a perpetual?

A perpetual futures contract lets you gain leveraged exposure to an asset's price without owning the asset itself. Unlike traditional futures (which expire on a fixed date), perpetuals can be held indefinitely.

Your profit or loss is determined by the difference between your entry price and the current price, multiplied by your position size.

**Going long:** You profit when the price rises. A 10% price increase on a 2x leveraged position = ~20% return on your margin.

**Going short:** You profit when the price falls. A 10% price decline on a 2x leveraged position = ~20% return on your margin.

**Leverage amplifies both gains and losses.** At 10x leverage, a 10% adverse move wipes out your margin entirely.

<div class="image-placeholder">
  <img src="/img/pnl-explainer.png" alt="P&L explainer" />
  <p><em>How leverage affects your P&L</em></p>
</div>

---

## Mark price vs. last traded price

**Mark price** is the fair value of the perpetual, derived from TrueCurrent's onchain price oracle. It's used for all P&L calculations and liquidation checks. It tracks the underlying spot price closely and is designed to resist short-term manipulation.

**Last traded price** is the most recent fill price on TrueCurrent. It may briefly diverge from mark price, but funding rates keep them anchored over time.

Your unrealized P&L is always based on **mark price**.

---

## Margin

When you open a position, you allocate **margin** – USDC collateral that backs your position.

```
Position notional = Quantity × Price
Leverage = Notional / Margin
```

**Initial margin** is the minimum required to open a position. **Maintenance margin** is the minimum to keep it open. When your margin falls below maintenance margin due to adverse price movement, your position is liquidated.

---

## Liquidation

When the mark price reaches your **liquidation price**, your position is automatically closed:

- The liquidation system closes your position at current market prices
- Your remaining margin covers the settlement
- Any shortfall is absorbed by the insurance fund
- Any surplus above the minimum cost is returned to your account

**Liquidation price** is visible for every open position. Monitor it. If you're getting close, you can:

- Add margin to your position
- Partially close to reduce exposure
- Close the position entirely

---

## Cross-margin

TrueCurrent uses **cross-margin** – all positions in your subaccount share the same margin pool. Profits from one position can offset losses in another, which is more capital efficient. However, a large losing position can draw on the margin of your other positions.

---

## Oracle and index price

Every market's mark price comes from TrueCurrent's onchain oracle, which aggregates data from multiple sources. This price is publicly visible onchain – liquidations and P&L calculations are fully transparent and verifiable.
