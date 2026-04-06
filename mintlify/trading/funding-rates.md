---
title: "Funding rates"
description: "--"
updatedAt: "2026-04-06"
---

Perpetual contracts have no expiry, so they use **funding rates** to keep the perpetual price anchored to the underlying spot price. Funding is a periodic payment exchanged between long and short positions.

---

## How funding works

Every hour, a funding payment is calculated and exchanged between all open positions:

- If the **funding rate is positive**: longs pay shorts. This happens when the perpetual trades at a premium to spot – the payment incentivizes traders to short (and thus push the perpetual price back down).
- If the **funding rate is negative**: shorts pay longs. This happens when the perpetual trades at a discount to spot.

Funding payments are automatically applied to your margin. You don't need to do anything.

---

## How the funding rate is calculated

The funding rate on TrueCurrent is derived from the difference between the perpetual mark price and the underlying index price:

```
Funding rate = (Mark price − Index price) / Index price × Funding factor
```

The funding factor controls how aggressively the rate adjusts. The resulting rate is clamped to prevent extreme payments in highly dislocated markets.

On TrueCurrent, funding settles **hourly**. The rate displayed in the UI is the per-hour rate. To estimate your daily funding cost, multiply by 24.

---

## Impact on your position

**If you're long and funding is positive:**
You pay a small amount per hour to short holders. This is a cost of holding a long position when the perpetual trades at a premium.

**If you're short and funding is negative:**
You pay a small amount per hour to long holders. This is a cost of holding a short when the perpetual trades at a discount.

Funding is automatically deducted from (or added to) your margin each hour. If your funding payments are large enough relative to your remaining margin, they could accelerate your approach to liquidation.

---

## Reading the funding rate display

In the market header, TrueCurrent shows:

- **Current funding rate** – the rate that will be applied at the next hourly settlement
- **Countdown to next funding** – time until the next payment

A positive rate means longs are currently paying shorts. A negative rate means shorts are paying longs.

Funding rates close to zero indicate the perpetual price is well-anchored to spot – a healthy market condition.

---

## Strategic implications

Funding rates carry information about market sentiment. A persistently high positive funding rate suggests the market is heavily long and traders are paying a significant premium to hold long positions – historically a sign of elevated speculative interest that sometimes precedes reversals.

If you're paying significant funding on a position, factor that into your holding cost. For long-duration positions, funding can become a meaningful drag.
