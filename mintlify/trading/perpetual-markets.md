---
title: "Perpetual markets"
description: "Learn the core mechanics of perpetual futures on TrueCurrent, including leverage, margin, funding, index price, mark price, and liquidation."
updatedAt: "2026-05-27"
---

TrueCurrent offers perpetual futures – leveraged positions on asset prices with no expiration date.

---

## What is a perpetual futures contract?

A perpetual futures contract lets you gain leveraged exposure to an asset’s price without owning the asset itself. Unlike traditional futures, which expire on a fixed date, perpetuals can be held indefinitely as long as the position continues to meet margin requirements.

Your unrealized profit or loss is determined by the difference between your entry price and the current index price, multiplied by your position size.

**Going long:** You profit when the price rises. A 10% price increase on a 2x leveraged position results in an approximate 20% return on your margin.

**Going short:** You profit when the price falls. A 10% price decline on a 2x leveraged position results in an approximate 20% return on your margin.

**Leverage amplifies both gains and losses.** At 10x leverage, an approximate 10% adverse move wipes out your margin entirely.

{/* SCREENSHOT SLOT: pnl-explainer — annotated diagram showing how a 10% price move translates to P&L at different leverage levels (1x, 5x, 10x, 20x). */}

---

## Index price, mark price, and last traded price

**Index price** is the streamed position reference used as the primary source for unrealized P&L in the TrueCurrent UI. The Positions panel labels this value **Index Price**.

**Mark price** is the onchain risk reference used for margin checks, liquidations, funding, TP/SL trigger evaluation, and quote validation. It is derived from external oracle sources.

**Last traded price** is the most recent fill price on TrueCurrent. It can differ from mark price because RFQ fills execute against quoted prices from liquidity providers.

Your unrealized P&L is based primarily on **index price**. Liquidation risk remains based on **mark price**. Neither value is the same as last traded price.

---

## Margin

When you open a position, you allocate **margin**: USDC collateral that backs your position.

```text
Position notional = Quantity × Price
Leverage = Notional / Margin
```

**Initial margin** is the minimum margin required to open a position.

**Maintenance margin** is the minimum margin required to keep the position open.

Liquidation trigger is when your margin falls below maintenance margin due to adverse price movement. This is when your position is liquidated.

As the index price moves, your displayed unrealized P&L changes. Separately, the protocol monitors mark-price-based risk: if the position moves against you and remaining equity falls below the maintenance margin requirement, the position becomes eligible for liquidation.

---

## Liquidation vs. bankruptcy 

Liquidation and bankruptcy are related, but they are not the same thing.

**Liquidation price** is the mark price at which your position no longer satisfies the maintenance margin requirement. When the mark price reaches this level, the position becomes eligible to be forcibly closed by the liquidation system.

**Bankruptcy price** is the price at which your position’s margin would be fully depleted. At the bankruptcy price, there is no remaining position equity left to absorb further losses.

In normal conditions, liquidation is designed to happen before bankruptcy. The maintenance margin requirement creates a buffer between the liquidation price and the bankruptcy price.

For example, a long position may be liquidated once the mark price falls to the liquidation price. But the position’s bankruptcy price would be lower than that: it is the price where the trader’s remaining margin would be completely exhausted.

---

## What happens during liquidation?

When the mark price reaches your liquidation price, your position may be automatically closed.

The liquidation system attempts to close the position before it reaches bankruptcy. The actual price achieved during this process is the liquidation closing price.

There are three important prices:

- Liquidation price = the mark price that triggers liquidation eligibility
- Bankruptcy price = the price where the position has zero remaining equity
- Closing price = the actual execution price achieved during liquidation

If the position is closed at a price better than the bankruptcy price, there is remaining equity after the position is closed. That surplus is then split between the liquidator and the insurance fund for that market.

If the position is closed at a price worse than the bankruptcy price, the position’s remaining margin is not enough to cover the loss. In that case, the shortfall is covered by the insurance fund for that market. The insurance fund protects market solvency. It **does not** protect an individual trader from being liquidated.

---

## Liquidation price

Your liquidation price is visible for every open position. Monitor it carefully.

If your position is approaching liquidation, you can:

- Add margin to increase your buffer
- Partially close the position to reduce exposure
- Close the position entirely

---

## Liquidation vs. bankruptcy example

Assume a trader opens a **1 BTC short** at an entry price of **\$80,000**, with max leverage (50x).

Entry price = 80,000 USDC\
Position size = 1 BTC\
Initial margin ratio = 1.9231%\
Maintenance margin ratio = 1%

The trader’s initial margin is: 80,000 × 1.9231% = 1,538.48 USDC

For a short position, the trader loses money as the mark price rises. The **bankruptcy price** is the price where the trader’s margin is fully depleted:

Bankruptcy price = Entry price × (1 \+ IMR)\
Bankruptcy price = 80,000 × (1 \+ 0.019231)\
Bankruptcy price = 81,538.48

At **81,538.48**, the short has lost **1,538.48 USDC**, which equals the trader’s full initial margin. The position has no remaining equity.

The **liquidation price** is reached earlier, when remaining equity falls to the maintenance margin requirement:

Liquidation price = Entry price × (1 \+ IMR) / (1 \+ MMR)\
Liquidation price = 80,000 × (1 \+ 0.019231) / (1 \+ 0.01)\
Liquidation price ≈ 80,731.17

At **80,731.17**, the short has lost approximately **731.17 USDC**.

The trader still has remaining equity: 1,538.48 - 731.17 = 807.31 USDC

So the position becomes eligible for liquidation at approximately **80,731.17**, before reaching the bankruptcy price of **81,538.48**. The excess margin - if any - is distributed to the liquidator and insurance fund for the market.

---

## Isolated margin

TrueCurrent uses **isolated margin**. Each position has its own margin pool.

This means the margin assigned to one position is separate from the margin assigned to another position. Profits from one position do not automatically offset losses in another position.

You can be liquidated on one isolated position even if another position is profitable.

---

## Price sources

Every market has an onchain mark price. The source may vary by market, but the resulting mark price is publicly visible and used for liquidation checks, funding, trigger-order evaluation, margin checks, and quote validation.

The indexer also streams `indexPrice` alongside `markPrice`. TrueCurrent uses `indexPrice` as the primary source for position valuation and unrealized P&L display.
