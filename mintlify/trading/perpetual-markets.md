---
title: "Perpetual markets"
updatedAt: "2026-04-08"
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

## Prices

TrueCurrent uses a few distinct prices, each serving a specific purpose.

**Index price.** The real-time spot price of the underlying asset, aggregated from multiple major external exchanges. It represents the broader market's consensus on where the asset is actually trading.

**Mark price.** The fair value of the perpetual, used for all P&L calculations, margin ratio, and liquidation checks. It comes from TrueCurrent's onchain oracle.

Today, **mark price tracks index price 1:1** – no funding-basis adjustment is applied. This may change as funding-rate mechanics evolve, but for now the two are interchangeable in practice.

{/* TODO: CK to update once mark price diverges from index (funding-basis adjustment goes live) */}

**Quoted price.** The actual execution price delivered by the liquidity providers who compete to fill your trade. This is what you bought or sold at.

**Indicative price.** A pre-trade estimate of the quoted price, shown in the UI before you confirm. In stable markets the two are nearly identical; in fast-moving markets they may differ slightly, which is what your worst-price tolerance is for.

**Which price applies when:**

| Price | Used for |
|-------|----------|
| Index / Mark | Unrealized P&L, margin ratio, liquidation price, funding rate basis |
| Quoted | Trade entry and exit, realized P&L, TP/SL trigger evaluation |
| Indicative | UI preview before you confirm a trade |

All of these prices are publicly visible onchain – liquidations and P&L calculations are fully transparent and verifiable.

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
