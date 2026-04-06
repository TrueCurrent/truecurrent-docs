---
title: "Margin trading"
description: "--"
updatedAt: "2026-04-06"
---

Margin trading lets you control a position larger than your deposited collateral by using leverage. This page explains how margin is calculated, how your account equity is tracked, and how to read the key metrics in your positions panel.

---

## Key terms

| Term | Definition |
|------|-----------|
| **Margin (M)** | The USDC collateral you deposit to back a position |
| **Leverage (L)** | The multiplier applied to your margin to determine position size |
| **Notional value (N)** | The total USD value of your position at current mark price |
| **IMR** | Initial Margin Rate – minimum margin required to *open* a position, equal to $1/L$ |
| **MMR** | Maintenance Margin Rate – minimum margin required to *keep* a position open |
| **Mark price** | The fair-value price used for all P&L and margin calculations |
| **uPnL** | Unrealized profit or loss on an open position |

---

## Notional value

The notional value of your position is the total exposure at the current mark price:

$$N = Q \times P_{mark}$$

where $Q$ is your position size (quantity of contracts) and $P_{mark}$ is the current mark price.

---

## Initial margin

The initial margin required to open a position is:

$$IM = Q \times P_{entry} \times IMR = \frac{Q \times P_{entry}}{L}$$

For example, opening a 10 BTC position at $50,000 with 10× leverage requires:

$$IM = \frac{10 \times 50{,}000}{10} = \$50{,}000$$

---

## Unrealized P&L

Your unrealized P&L updates continuously with the mark price.

**Long position:**

$$uPnL = (P_{mark} - P_{entry}) \times Q$$

**Short position:**

$$uPnL = (P_{entry} - P_{mark}) \times Q$$

---

## Account equity

Account equity is the current value of your position, including unrealized gains or losses:

$$E = M + uPnL$$

When $uPnL$ is negative (position moving against you), equity decreases. When equity falls toward the maintenance margin threshold, you approach liquidation.

---

## Margin ratio

The margin ratio expresses your current equity as a fraction of your position's notional value:

$$MR = \frac{M + uPnL}{Q \times P_{mark}}$$

Liquidation is triggered when:

$$MR \leq MMR$$

You can monitor your margin ratio in the Positions panel. The closer it is to MMR, the closer you are to liquidation.

---

## Available margin

Available margin is the excess equity above what is required to maintain your position:

$$M_{avail} = E - Q \times P_{mark} \times MMR = M + uPnL - Q \times P_{mark} \times MMR$$

This is the amount you can withdraw without closing any positions, and the buffer you have before liquidation.

---

## Maintenance margin rate

Each market has a fixed MMR. Positions are liquidated when the margin ratio falls at or below this threshold. MMR is lower than IMR, which means you have some buffer before liquidation even if the market moves against you immediately after entry.

For example, if $IMR = 5\%$ (20× leverage) and $MMR = 2.5\%$, you have a 2.5% buffer between your entry and your liquidation threshold.

---

## Cross-margin

TrueCurrent uses **cross-margin** across all positions in your subaccount. All open positions share the same equity pool – profits on one position offset losses on another.

This is more capital-efficient than isolated margin but means a large losing position can draw on the margin of your other positions, increasing the risk of cascading liquidations if multiple positions move against you simultaneously.

---

## Managing your margin

**Add margin** to a position at any time to increase your equity and push your liquidation price further away. Go to your position in the Positions panel and click **Add Margin**.

**Reduce leverage** by partially closing a position, which releases margin and reduces your notional exposure.

**Monitor the margin ratio** – if it approaches MMR, act proactively rather than waiting for liquidation.

See [Liquidation](/trading/liquidation) for exactly how the liquidation price is calculated.
