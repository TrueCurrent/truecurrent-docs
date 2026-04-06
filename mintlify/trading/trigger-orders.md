---
title: "Trigger orders (TP/SL)"
description: "Set up automated take profit and stop loss trigger orders on TrueCurrent using quoted price execution to lock in gains and limit losses without constant market monitoring."
updatedAt: "2026-04-06"
---

Trigger orders let you automate your exit strategy. You set a target price in advance, and TrueCurrent closes your position automatically when that price is reached – without you having to watch the market.

There are two types: **Take Profit (TP)** and **Stop Loss (SL)**. They can be set when you open a position or added to any existing position at any time.

---

## Take profit (TP)

A take profit order closes your position when the price reaches a level that is favorable to you.

- **Long TP**: Triggers when the quoted price rises to or above your target
- **Short TP**: Triggers when the quoted price falls to or below your target

When triggered, TrueCurrent automatically closes your position at the best available quoted price. Your profit is locked in without requiring manual action.

**Example:** You open a long at $100 and set a TP at $120. When the quoted price reaches $120, your position closes and you realize the ~20% gain.

---

## Stop loss (SL)

A stop loss order closes your position when the price moves against you to a level you've pre-defined.

- **Long SL**: Triggers when the quoted price falls to or below your stop price
- **Short SL**: Triggers when the quoted price rises to or above your stop price

Stop losses cap your downside risk. If the market moves sharply against you, your position is closed automatically before losses grow further.

**Example:** You open a long at $100 and set a SL at $90. If the quoted price drops to $90, your position closes and your loss is limited to ~10%.

---

## How trigger prices work

Trigger orders on TrueCurrent are evaluated against the **quoted price** – the live execution price from liquidity providers – not the mark price or index price.

This means:

- A TP or SL will not fire based on a wick in the mark price that doesn't reflect actual tradeable liquidity
- The trigger fires when real prices from liquidity providers reach your level
- Once triggered, the close executes through the same competitive liquidity network as any other trade

---

## Setting TP/SL

### At order entry

When placing a new trade, you can set both a TP and SL directly in the order panel before confirming. These are attached to the position from the moment it opens.

<div class="image-placeholder">
  <img src="/img/tpsl-entry.png" alt="Setting TP/SL at entry" />
  <p><em>Setting take profit and stop loss when opening a position</em></p>
</div>

### On an existing position

You can add or modify TP/SL on any open position at any time:

1. Go to your **Positions** panel
2. Click the TP/SL icon on the position you want to manage
3. Enter or update your target prices
4. Confirm

<div class="image-placeholder">
  <img src="/img/tpsl-modify.png" alt="Modifying TP/SL on existing position" />
  <p><em>Adding or modifying TP/SL on an open position</em></p>
</div>

---

## Price tolerance on trigger orders

When a trigger order fires, TrueCurrent executes the close with a default price tolerance. If the market is moving fast and the price gaps beyond tolerance at execution time, the order will retry until it fills or you cancel it manually.

You can set a custom price tolerance for your trigger orders the same way you would for a regular trade.

---

## Managing risk with TP/SL

**Always set a stop loss.** Leverage amplifies losses just as quickly as it amplifies gains. A stop loss is your safety net against a position running away from you, especially overnight or when you're not watching.

**Use TP to lock in gains automatically.** Markets can reverse quickly. Setting a take profit removes the emotional burden of deciding when to exit a winning trade.

**Consider the spread between entry and SL.** Setting a stop loss very close to your entry price (e.g., 0.1% away at high leverage) risks being stopped out by normal market noise. Give your position room to breathe while still protecting against a meaningful adverse move.

See [Liquidation](/trading/liquidation) to understand the difference between a stop loss (which you control) and a liquidation (which happens automatically when margin runs out).
