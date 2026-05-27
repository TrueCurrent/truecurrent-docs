---
title: "Index, mark, and quoted prices"
description: "Understand the difference between index price, mark price, quoted price, estimated price, and worst price on TrueCurrent."
updatedAt: "2026-05-27"
---

TrueCurrent uses **index price** as the primary displayed position reference and unrealized P&L source, **mark price** as the protocol risk reference, and **quoted price** as the execution price. The Positions panel displays **Index Price** where this streamed position reference is shown.

---

## Index price

The **index price** is the streamed market reference used to value open positions in the TrueCurrent UI. It is exposed as `indexPrice` and updates the same way as `markPrice`.

Index price is used for:

- **Index Price display** — the current position reference shown in the Positions panel
- **Unrealized P&L** — how your open position is valued before you close it
- **Position monitoring** — the primary UI reference for open-position performance

Index price is **not** your execution price. You do not buy or sell at index price unless a maker's quote happens to match it.

---

## Mark price

The **mark price** is the onchain reference price for a TrueCurrent perpetual market. It is exposed as `mark_price` onchain and `markPrice` in streamed market data.

Mark price is used for:

- **Margin ratio** — how close your position is to liquidation
- **Liquidation** — liquidation checks are evaluated against mark price
- **Trigger orders** — take profit and stop loss triggers fire when mark price crosses your trigger level
- **Price validation** — `worst_price` and maker quotes must stay within the allowed band around mark price

Mark price is **not** your execution price. It remains the contract-facing reference for risk and validation.

---

## Quoted price

The **quoted price** is the price offered by a maker through TrueCurrent's RFQ flow. This is the price that becomes your fill price if the quote is selected and settles onchain.

When you submit a trade, TrueCurrent requests signed quotes from liquidity providers. Each quote includes a price, quantity, expiry, maker address, and signature. TrueCurrent selects the best executable quote automatically:

- For a **long**, lower quoted prices are better because you are buying
- For a **short**, higher quoted prices are better because you are selling

Your quoted price is used for:

- **Trade execution** — the entry or exit price of the fill
- **Realized P&L** — the price used when your position closes
- **Trade history** — the fill price shown for settled trades

Quoted price can differ from index price and mark price. That difference is the effective spread you receive from the RFQ maker.

---

## Estimated price

Before you confirm a trade, the UI may show an **estimated price**. This is a preview based on current market conditions. It is not a signed quote and is not guaranteed.

The final execution price is determined only after makers respond with signed quotes. In calm markets, the estimated price and final quoted price should usually be close. In fast-moving markets, they can differ.

---

## Worst price

Your **worst price** is the least favorable execution price you are willing to accept. It is derived from your price tolerance or set directly, depending on the flow.

- For a **long**, worst price is the highest price you are willing to pay
- For a **short**, worst price is the lowest price you are willing to receive

TrueCurrent will not settle a trade at a price worse than your `worst_price`. If no maker quote can satisfy that limit, the trade does not execute.

`worst_price` is also checked against mark price. The contract rejects prices that are too far away from the current mark price, even if the taker signed them.

---

## Trigger orders

Take profit and stop loss orders use both mark price and quoted price:

1. The **trigger condition** is evaluated against mark price
2. Once triggered, the close is executed through the RFQ quote path
3. The selected quote must satisfy the signed `worst_price`

This means a trigger can fire without guaranteeing a fill. If the market moves beyond your allowed worst price before settlement, the close can fail instead of filling at a worse price.

---

## Summary

| Price | What it is | Used for |
|-------|------------|----------|
| **Index price** | Streamed position reference | Displayed Index Price, unrealized P&L, position monitoring |
| **Mark price** | Onchain risk reference price | Margin ratio, liquidation, TP/SL triggers, price validation |
| **Quoted price** | Signed maker quote price | Trade fills, realized P&L, trade history |
| **Estimated price** | Pre-trade UI preview | Showing an approximate expected fill before quotes are collected |
| **Worst price** | Your execution limit | Preventing settlement at a worse price than you accepted |
