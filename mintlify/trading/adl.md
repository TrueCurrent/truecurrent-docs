---
title: "Auto-deleveraging (ADL)"
description: "How auto-deleveraging works on TrueCurrent: when it activates, how the ADL queue ranks positions by leverage and unrealized profit, what price a deleveraged position exits at, and how to reduce your ADL exposure."
updatedAt: "2026-04-30"
---

Auto-deleveraging (ADL) is the final backstop in TrueCurrent's risk management waterfall. It is not a routine event — it activates only when an insolvent position cannot be fully covered by the insurance fund. When it fires, the most profitable positions on the opposite side are partially or fully closed to absorb the shortfall.

Understanding ADL matters for any leveraged trader or market maker: a profitable position at high leverage can be reduced without your manual action, and the exit price is determined by the insolvent counterparty's bankruptcy price, not by the current market price.

---

## The liquidation waterfall

When a position's margin ratio falls at or below the maintenance margin rate (MMR), the following sequence runs in order:

### Stage 1 — Partial liquidation

The liquidation engine attempts to reduce the position size enough to restore the margin ratio above MMR, leaving the remaining portion open. If this is sufficient and the proceeds cover losses, the process ends here. Residual margin above the bankruptcy price is returned to the trader.

### Stage 2 — Full liquidation

If partial liquidation is insufficient (position is too far underwater), the entire position is closed. The close attempts to execute at or near the liquidation price. Any surplus between the close price and the bankruptcy price is returned. Any shortfall between the close price and the bankruptcy price is absorbed by the insurance fund.

### Stage 3 — ADL

If the insurance fund balance is insufficient to cover the shortfall from Stage 2, ADL activates. Profitable positions on the opposite side of the market are selected from the ADL queue and closed at the insolvent position's **bankruptcy price** to absorb the remaining deficit.

---

## ADL queue ranking

When ADL activates, TrueCurrent must select which positions to deleverage. The selection is not random — positions are ranked by their **ADL score**:

$$\text{ADL score} = \text{leverage} \times \text{profit rate}$$

where profit rate is the unrealized P&L as a fraction of the initial margin:

$$\text{profit rate} = \frac{uPnL}{IM} = \frac{uPnL}{Q \times P_{entry} / L}$$

Positions with the highest score — those combining high leverage with high unrealized profit — are selected first. This prioritises closing positions that have benefited most from the market move that caused the insolvency.

**Practical implications:**

- A 20× short with a 60% unrealized gain scores 10× higher than a 2× short with the same percentage gain
- Reducing leverage while holding the same dollar profit lowers your score relative to higher-leverage positions
- Partially closing a winning position reduces both the score (lower uPnL numerator) and your absolute ADL exposure (smaller position size)

---

## Numerical scenario

**Setup:** INJ/USDC PERP, mark price drops sharply from $10.00 to $8.80.

### Insolvent position (Trader A — long)

| Parameter | Value |
|---|---|
| Entry price | $10.00 |
| Quantity | 2,000 INJ |
| Leverage | 20× |
| Initial margin | $1,000 |
| Liquidation price | $9.74 |
| Bankruptcy price | $9.50 |
| Mark price (post-move) | $8.80 |
| Loss beyond bankruptcy | 2,000 × ($9.50 − $8.80) = **$1,400** |
| Insurance fund balance | **$800** |
| Unabsorbed shortfall | **$600** |

The market gaps below the bankruptcy price before the liquidation engine can close the position. The insurance fund covers $800 of the $1,400 loss. The remaining $600 shortfall triggers ADL.

### ADL queue (profitable shorts, mark = $8.80)

| Trader | Entry | Qty | Leverage | Initial margin | uPnL | Profit rate | ADL score |
|---|---|---|---|---|---|---|---|
| B | $9.80 | 1,000 INJ | 10× | $980 | $1,000 | 102% | **10.2** |
| C | $9.50 | 500 INJ | 5× | $950 | $350 | 37% | **1.8** |

Trader B has the highest score and is selected first. A portion of Trader B's short position is closed at **$9.50** — Trader A's bankruptcy price — not at the current mark price of $8.80.

### Trader B's outcome

| | Closing at mark ($8.80) | ADL exit at bankruptcy ($9.50) |
|---|---|---|
| Entry price | $9.80 | $9.80 |
| Exit price | $8.80 | $9.50 |
| P&L per INJ | +$1.00 | +$0.30 |

Trader B's position is still closed at a profit. ADL does not result in a loss — it reduces the profit on the ADL'd quantity by the difference between the bankruptcy price and the mark price at the time of ADL ($0.70/INJ in this example).

---

## The trader's experience

When ADL closes your position:

1. **No advance warning.** ADL fires immediately when the insurance fund shortfall condition is met. There is no notification before the close.
2. **Position closes without manual action.** The closure appears in your position history with an ADL closure event. You do not need to approve or take any action — the position is already closed when you see it.
3. **Exit price is the bankruptcy price of the insolvent counterparty.** This is always less favorable than the current mark price for a profitable position. The difference between the two is the implicit cost of ADL.
4. **Remaining position is unaffected.** If ADL closes only part of your position (because the shortfall was covered before your full size was needed), your remaining position continues as normal.

---

## ADL indicator

Some interfaces display an ADL risk indicator for your open positions — typically shown as a series of bars or a percentile reading. This indicator reflects your position's relative ranking in the ADL queue at the current mark price and leverage:

- **Higher reading:** your position is near the top of the queue; it would be among the first selected if ADL activates
- **Lower reading:** your position would only be reached after higher-ranked positions are exhausted

The indicator is a relative ranking, not a probability prediction. ADL itself activates only in extreme conditions (insurance fund depleted). A high indicator reading means elevated exposure *if* ADL were to activate, not that it will.

---

## Reducing ADL risk

You cannot completely eliminate ADL risk while holding a leveraged position during an adverse market event, but two levers reduce your exposure:

**Lower leverage.** The ADL score formula is `leverage × profit rate`. At the same dollar profit, a 5× position scores half what a 10× position scores. Reducing leverage while maintaining your trade thesis moves you further back in the ADL queue.

**Smaller position size.** Even if your position is selected for ADL, a smaller notional size means a smaller absolute portion is closed. This limits the dollar impact of an ADL event on your account.

**Take profit before extreme events.** Realising profit by closing or partially closing a winning position removes it from the ADL queue entirely. A closed position cannot be ADL'd.

---

## ADL vs liquidation — key differences

|  | Liquidation | ADL |
|---|---|---|
| Who is affected | The trader whose margin ratio fell below MMR | Profitable traders on the opposite side |
| When it fires | When margin ratio ≤ MMR | When insurance fund cannot cover liquidation shortfall |
| Exit price | Liquidation price (or as close as possible) | Bankruptcy price of the insolvent position |
| Advance notice | No | No |
| Can be avoided | Yes (add margin, reduce leverage) | Yes (reduce leverage, reduce size, take profit) |

---

## Related pages

- [Liquidation](/trading/liquidation) — when and how liquidation triggers, liquidation price formula, bankruptcy price
- [Margin trading](/trading/margin-trading) — margin ratio, initial margin, maintenance margin rate
- [Contract specifications](/trading/contract-specifications) — MMR values per market
