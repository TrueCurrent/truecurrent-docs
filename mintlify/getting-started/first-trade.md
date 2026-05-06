---
title: "Your first trade"
description: "Step-by-step tutorial for executing your first perpetual futures trade on TrueCurrent, covering market selection, leverage setup, price tolerance, position monitoring, and funding rate considerations."
updatedAt: "2026-05-06"
---

A step-by-step guide to opening your first position on TrueCurrent. All workflows below reflect the platform as of 2026 — typical fill times under one second, zero taker fees, and hourly funding intervals.

---

## Before you start

Make sure you've:

- Connected your wallet ([guide](/getting-started/connect-wallet))
- Completed the one-time authorization (prompted automatically on first visit)
- Deposited USDC ([guide](/getting-started/deposit-funds))

---

## Step 1: Choose a market

From the trading interface, select the market you want to trade. Each market shows the current price, 24h volume, and funding rate.

{/* SCREENSHOT SLOT: market-select — market picker dropdown open, showing several markets with current price and 24h volume. */}

---

## Step 2: Long or short?

**Long** – you profit if the price goes up.

**Short** – you profit if the price goes down.

---

## Step 3: Set your size and leverage

Enter the **amount of USDC** you want to use as margin (your collateral). Then set your leverage – this determines the size of your position relative to your margin.

For example, 100 USDC margin at 2x leverage means a 200 USDC position. A 10% price move in your favor results in a 20 USDC profit (20% on your margin). A 10% adverse move results in a 20 USDC loss.

**Start with low leverage (1x–3x) while learning.** Higher leverage amplifies both gains and losses.

{/* SCREENSHOT SLOT: trade-size — trade panel with margin field, leverage slider, and computed position size visible. */}

---

## Step 4: Set your price tolerance

Before confirming, you'll see an **estimated price** and a **worst price** you're willing to accept.
You can adjust how tight or loose this tolerance is.

TrueCurrent will never execute your trade at a price worse than your limit.
If no liquidity provider can beat your limit, the trade simply won't go through.

---

## Step 5: Confirm

Click **Trade**. TrueCurrent automatically collects prices from institutional liquidity providers, picks the best one, and executes your trade – all in under a second.

{/* SCREENSHOT SLOT: trade-confirm — confirm modal showing estimated price, worst price, and the Trade button. */}

---

## Step 6: Monitor your position

Your open position appears in the **Positions** panel below the chart. You'll see:

- Entry price and current mark price
- Unrealized P&L
- Your **liquidation price** – if the mark price reaches this level, your position is automatically closed and you lose your margin.
Keep an eye on it.

{/* SCREENSHOT SLOT: positions-panel — Positions panel with at least one open position; entry, mark, uPNL, margin, and liquidation columns clearly readable. */}

---

## Closing your position

Click **Close** on any open position. Choose full or partial close and confirm.

---

## Tips

**Watch your liquidation price.** If you're getting close, you can add margin or partially close to reduce risk.

**Check funding rates.** If you're holding a position overnight or longer, funding payments can add up.
See [Funding rates](/trading/funding-rates).
