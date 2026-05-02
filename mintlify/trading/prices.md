---
title: "Index, mark, and quoted prices"
description: "Differentiate between index price, mark price, quoted price, and indicative price on TrueCurrent for accurate P&L tracking, liquidation monitoring, and trigger order execution on perpetual markets."
updatedAt: "2026-04-30"
---

TrueCurrent uses three distinct prices, each serving a specific purpose. Understanding the difference helps you interpret your P&L, anticipate liquidations, and know exactly what price you traded at.

---

## Index price

The **index price** is the real-time spot price of the underlying asset, aggregated from external sources. It represents the broader market's consensus on where an asset is actually trading.

The index price is the foundation for the mark price and funding rate calculation. It is not the price you trade at.

### Oracle construction

TrueCurrent's index price is sourced from the **Injective oracle module**, which aggregates prices from a set of vetted external providers. The oracle network includes:

- **Major centralised exchanges** — spot price feeds from leading venues by volume
- **Onchain oracle providers** — integrated via the Injective oracle module, which supports providers such as Band Protocol and Pyth Network for certain markets

**Aggregation methodology:** The index price is computed as a **median** (or weighted average, as configured per market) of the available source prices at the time of sampling. Using a median rather than an average means a single outlier or temporarily manipulated source cannot materially move the index.

**Update cadence:** Oracle prices on Injective are updated on each block (approximately every 1–2 seconds). TrueCurrent's mark price and funding rate calculations consume the latest available oracle price at each settlement event.

**Staleness circuit breakers:** If oracle price feeds fall behind by more than a defined staleness threshold, the Injective exchange module halts funding settlements and may restrict new position openings until fresh prices are available. This prevents liquidations from being triggered by stale or manipulated prices.

<Note>
**TrueCurrent does not apply a CEX-weighted median.** Unlike some other perpetual venues (which weight CEX sources by volume), TrueCurrent's index price uses the Injective oracle aggregation, which is governed by the Injective protocol and treats sources according to the oracle module's configuration — not a proprietary weighting scheme controlled by TrueCurrent. This means the index price methodology is transparent and verifiable onchain.
</Note>

For the full list of oracle providers integrated with a specific market, refer to the [Injective oracle documentation](https://docs.injective.network/develop/modules/injective/oracle/) and the market's onchain parameters.

---

## Mark price

The **mark price** is TrueCurrent's fair-value price for a perpetual contract. It is derived from the index price with adjustments for the current funding rate basis:

$$P_{mark} = P_{index} + \text{Basis}$$

where the basis reflects the premium or discount at which the perpetual trades relative to spot.

### Basis calculation

The basis is not simply the instantaneous difference between the perpetual mid and spot. It is computed using a smoothed premium:

- **Impact price:** The basis is measured using an impact-notional-weighted price — the average execution price for a hypothetical buy and sell order of a defined notional size against the order book. This is more manipulation-resistant than using the raw order book mid.
- **EMA smoothing:** The raw basis is passed through an exponential moving average (EMA) to reduce the influence of transient spikes. The EMA window is set per market as a governance parameter.
- **Blend:** The final mark price blends the smoothed basis with the index price. In liquid, well-anchored markets the mark price tracks the index closely; in dislocated markets, the EMA prevents the mark from snapping violently.

**Manipulation resistance:** Because the basis is derived from impact prices rather than quoted prices, spoofing the order book mid does not directly move the mark price — an attacker would need to execute actual trades of significant notional size to shift the impact price.

Mark price is the price used for:

- **Unrealized P&L** – your open position value
- **Margin ratio** – determining how close you are to liquidation
- **Liquidation** – your position is liquidated when mark price reaches your liquidation price
- **Funding payments** – calculated based on the mark/index divergence

Mark price is **not** the price your trade executes at.

For how P&L is calculated across the position lifecycle (open, hold, funding, close), see [Margin trading](/trading/margin-trading).

---

## Quoted price

The **quoted price** is the actual execution price – the price at which your trade fills. On TrueCurrent, this comes directly from institutional liquidity providers who compete to offer you the best rate.

Because liquidity providers price each trade based on live market conditions, the quoted price will typically be very close to the mark price, with a small competitive spread.

Quoted price is used for:

- **Trade execution** – your entry and exit prices
- **Realized P&L** – what you actually bought or sold at
- **Trigger orders (TP/SL)** – take profit and stop loss triggers are evaluated against mark price; the resulting close executes against the best available quote within `worst_price`

---

## Indicative price

Before you confirm a trade, TrueCurrent displays an **indicative price** – an estimate of what your quoted price is likely to be, based on current market conditions.

The indicative price is not a firm quote. The actual quoted price is determined at execution time when liquidity providers respond. In stable markets, the two will be nearly identical. In fast-moving markets, they may differ slightly.

Your [price tolerance](/trading/slippage-and-worst-price) setting determines the maximum deviation you'll accept between the indicative price and the actual quoted price. The trade will never execute at a price worse than your specified limit.

---

## Summary

| Price | What it is | Used for |
|-------|-----------|---------|
| **Index price** | Median of external oracle feeds (CEX spot + onchain oracles) | Mark price calculation, funding rates |
| **Mark price** | Index price + EMA-smoothed impact-price basis | P&L, margin ratio, liquidation |
| **Quoted price** | Actual execution price from liquidity providers | Trade fills, realized P&L, TP/SL triggers |
| **Indicative price** | Pre-trade estimate of quoted price | Shown in UI before you confirm |
