---
title: "Placing a trade"
description: "--"
updatedAt: "2026-04-06"
---

For a full step-by-step walkthrough, see **[How to trade](how-to-trade.md)**.

This page covers the specific parameters you control when placing a trade.

---

## Trade parameters

| Parameter | What it means |
|-----------|--------------|
| **Market** | Which perpetual to trade (e.g., INJ/USDC PERP) |
| **Direction** | Long (buy) or Short (sell) |
| **Margin** | USDC collateral you're allocating |
| **Leverage** | Multiplier on your margin (determines position size) |
| **Price tolerance** | How far from current price you'll accept a fill |

---

## Getting a price

When you click **Get Quote**, TrueCurrent finds the best available price from its liquidity network and returns it within seconds. The price is firm – it won't change while you decide.

Review the fill price, compare it to the mark price, and accept if you're satisfied.

---

## Opening vs. closing

**Opening:** Submit a long or short trade request with your desired parameters.

**Closing:** Submit a trade in the opposite direction of your open position. You can close fully or partially.

Closing works exactly like opening – you get a competitive price and confirm. The released margin returns to your available balance immediately.

---

## Fill guarantees

The price you accept is the price you get. There's no slippage between confirmation and settlement. If no price within your tolerance is available, the order is cancelled and your margin is returned.

Settlement completes onchain in approximately one second.

---

## For technical details

If you're a developer or market maker and want to understand the full settlement mechanics, see the [Technical reference](../technical/architecture.md) and [How RFQ works](../technical/how-rfq-works.md).
