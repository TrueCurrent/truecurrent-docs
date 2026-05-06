---
title: "Trigger orders (TP/SL)"
description: "Set take profit and stop loss orders that trigger from mark price and close through TrueCurrent’s RFQ quote flow. "
updatedAt: "2026-05-04"
---

Trigger orders let you predefine exits for an open position. When the mark price reaches your trigger level, TrueCurrent attempts to close the position through the RFQ quote path.

There are two types: **Take Profit (TP)** and **Stop Loss (SL)**. They can be set when you open a position or added to an existing position.

---

## Take profit (TP)

A take profit order closes your position when the price reaches a level that is favorable to you.

- **Long TP**: Triggers when the mark price rises to or above your target
- **Short TP**: Triggers when the mark price falls to or below your target

When triggered, TrueCurrent automatically closes your position at the best available quote within your configured price tolerance. Your profit is locked in without requiring manual action.

**Example:** You open a long at \$100 and set a TP order at \$120. When the mark price reaches \$120, your position closes and you realize the ~20% gain, subject to the quoted exit satisfying your `worst_price`.

---

## Stop loss (SL)

A stop loss order closes your position when the price moves against you to a level you've pre-defined.

- **Long SL**: Triggers when the mark price falls to or below your stop price
- **Short SL**: Triggers when the mark price rises to or above your stop price

Stop losses help limit downside, but they are not guaranteed fills. When the trigger fires, the close still needs a valid quote that satisfies your `worst_price`.

**Example:** You open a long at \$100 and set a SL at \$90. If the mark price drops to \$90, TrueCurrent attempts to close the position as long as the quoted exit can satisfy your limit.

---

## How trigger prices work

Trigger orders on TrueCurrent are evaluated against the **mark price**. The signed intent uses trigger variants such as `mark_price_gte` and `mark_price_lte`, and the contract re-evaluates that mark-price condition at execution time.

This means:

- A TP or SL fires from the same mark-price basis used by the signed-intent contract path
- The trigger is not latched: if mark price moves back before the relayer lands the transaction, the settlement can revert as `trigger_not_satisfied`
- Once triggered, the close still executes through the RFQ quote path, and your `worst_price` is enforced on the quoted exit

---

## Setting TP/SL

### At order entry

When placing a new trade, you can set both a TP and SL directly in the order panel before confirming. These are attached to the position from the moment it opens.

{/* SCREENSHOT SLOT: tpsl-entry — trade panel with the TP/SL section expanded, both TP and SL price fields filled in. */}

### On an existing position

You can add or modify TP/SL on any open position at any time:

1. Go to your **Positions** panel
2. Click the TP/SL icon on the position you want to manage
3. Enter or update your target prices
4. Confirm

{/* SCREENSHOT SLOT: tpsl-modify — TP/SL editor modal opened from a row in the Positions panel. */}

---

## Price tolerance on trigger orders

When a trigger order fires, TrueCurrent executes the close with a default price tolerance. If the market is moving fast and the price gaps beyond tolerance at execution time, the order will retry until it fills or you cancel it manually.

You can set a custom price tolerance for your trigger orders the same way you would for a regular trade.

---

## Managing risk with TP/SL

**Always set a stop loss.** Leverage amplifies losses just as quickly as it amplifies gains. A stop loss is your safety net against a position running away from you, especially overnight or when you're not watching.

**Use TP to lock in gains automatically.** Markets can reverse quickly. Setting a take profit removes the emotional burden of deciding when to exit a winning trade.

**Consider the spread between entry and SL.** Setting a stop loss very close to your entry price (e.g., 0.1% away at high leverage) risks being stopped out by normal market noise. Give your position room to breathe while still protecting against a meaningful adverse move.

See [Liquidation](/trading/liquidation) to understand the difference between a stop loss (which you control), and a liquidation (which happens automatically when margin runs out).

---

## API and SDK users

If you are using the API or SDK to place trigger orders, check out [advanced operational use cases for TP/SL](/trading/trigger-orders-advanced) — including off-chain execution details, silent failure modes, `worst_price` gap scenarios, and differences between frontend and programmatic behavior.