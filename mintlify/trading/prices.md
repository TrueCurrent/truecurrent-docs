---
title: "Index, mark, and quoted prices"
description: "--"
updatedAt: "2026-04-06"
---

TrueCurrent uses three distinct prices, each serving a specific purpose. Understanding the difference helps you interpret your P&L, anticipate liquidations, and know exactly what price you traded at.

---

## Index price

The **index price** is the real-time spot price of the underlying asset, aggregated from multiple major external exchanges. It represents the broader market's consensus on where an asset is actually trading.

The index price is the foundation for the mark price. It is not the price you trade at.

---

## Mark price

The **mark price** is TrueCurrent's fair-value price for a perpetual contract. It is derived from the index price with adjustments for the current funding rate basis:

$$P_{mark} = P_{index} + \text{Basis}$$

where the basis reflects the premium or discount at which the perpetual trades relative to spot. In practice, for well-anchored markets, mark price closely tracks index price.

Mark price is the price used for:

- **Unrealized P&L** – your open position value
- **Margin ratio** – determining how close you are to liquidation
- **Liquidation** – your position is liquidated when mark price reaches your liquidation price
- **Funding payments** – calculated based on the mark/index divergence

Mark price is **not** the price your trade executes at.

---

## Quoted price

The **quoted price** is the actual execution price – the price at which your trade fills. On TrueCurrent, this comes directly from institutional liquidity providers who compete to offer you the best rate.

Because liquidity providers price each trade based on live market conditions, the quoted price will typically be very close to the mark price, with a small competitive spread.

Quoted price is used for:

- **Trade execution** – your entry and exit prices
- **Realized P&L** – what you actually bought or sold at
- **Trigger orders (TP/SL)** – take profit and stop loss orders are evaluated against the quoted price, not mark price

---

## Indicative price

Before you confirm a trade, TrueCurrent displays an **indicative price** – an estimate of what your quoted price is likely to be, based on current market conditions.

The indicative price is not a firm quote. The actual quoted price is determined at execution time when liquidity providers respond. In stable markets, the two will be nearly identical. In fast-moving markets, they may differ slightly.

Your [price tolerance](/trading/slippage-and-worst-price) setting determines the maximum deviation you'll accept between the indicative price and the actual quoted price. The trade will never execute at a price worse than your specified limit.

---

## Summary

| Price | What it is | Used for |
|-------|-----------|---------|
| **Index price** | Aggregated spot from external exchanges | Mark price calculation, funding rates |
| **Mark price** | Fair value with funding basis | P&L, margin ratio, liquidation |
| **Quoted price** | Actual execution price from liquidity providers | Trade fills, realized P&L, TP/SL triggers |
| **Indicative price** | Pre-trade estimate of quoted price | Shown in UI before you confirm |
