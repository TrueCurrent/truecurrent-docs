---
title: "Perpetual markets"
description: "Learn the core mechanics of perpetual futures on TrueCurrent, including leverage, margin, funding, mark price, and liquidation."
updatedAt: "2026-05-06"
---

TrueCurrent offers perpetual futures – leveraged positions on asset prices with no expiration date.

---

## What is a perpetual?

A perpetual futures contract lets you gain leveraged exposure to an asset's price without owning the asset itself. Unlike traditional futures (which expire on a fixed date), perpetuals can be held indefinitely.

Your profit or loss is determined by the difference between your entry price and the current price, multiplied by your position size.

**Going long:** You profit when the price rises. A 10% price increase on a 2x leveraged position = ~20% return on your margin.

**Going short:** You profit when the price falls. A 10% price decline on a 2x leveraged position = ~20% return on your margin.

**Leverage amplifies both gains and losses.** At 10x leverage, a 10% adverse move wipes out your margin entirely.

{/* SCREENSHOT SLOT: pnl-explainer — annotated diagram showing how a 10% price move translates to P&L at different leverage levels (1x, 5x, 10x, 20x). */}

---

## Mark price vs. last traded price

**Mark price** is the fair-value price used for P&L, margin checks, liquidations, funding, and TP/SL trigger evaluation. It is derived from external oracle sources.

**Last traded price** is the most recent fill price on TrueCurrent. It can differ from mark price because RFQ fills execute against quoted prices from liquidity providers.

Your unrealized P&L is always based on **mark price**.

---

## Margin

When you open a position, you allocate **margin** – USDC collateral that backs your position.

```text
Position notional = Quantity × Price
Leverage = Notional / Margin
```

**Initial margin** is the minimum required to open a position.

**Maintenance margin** is the minimum to keep it open.

**Liquidation trigger** is when your margin falls below maintenance margin due to adverse price movement. This is when your position is liquidated.

---

## Liquidation

When the mark price reaches your **liquidation price**, your position is automatically closed:

- The liquidation system closes your position at current market prices
- Your remaining margin covers the settlement
- Any shortfall is absorbed by the insurance fund
- Any surplus after liquidation is split between the liquidator and the insurance fund for that market

**Liquidation price** is visible for every open position. Monitor it. If you're getting close, you can:

- Add margin to your position
- Partially close to reduce exposure
- Close the position entirely

---

## Isolated margin

TrueCurrent uses **isolated margin** – all positions in your subaccount have a unique margin pool. Profits from one position cannot offset losses in another. You can get liquidated on one position while making profits on another.

---

## Mark price source

Every market has an onchain mark price. The source may vary by market, but the resulting mark price is publicly visible and used for P&L, liquidation checks, funding, and trigger-order evaluation.
