---
title: "Liquidation"
description: "Learn when liquidation happens on TrueCurrent perpetuals, how the liquidation price is calculated, and how the insurance fund and ADL protect solvency."
updatedAt: "2026-04-30"
---

Liquidation happens when your account equity falls below the maintenance margin required for an open position. This page explains the trigger, liquidation price, bankruptcy price, insurance fund, and ways to reduce liquidation risk.

---

## What triggers liquidation

Liquidation is triggered when your margin ratio falls at or below the maintenance margin rate:

$MR = \frac{M + uPnL}{Q \times P_{mark}} \leq MMR$

As the mark price moves against your position, $uPnL$ decreases, which decreases $MR$. When $MR$ hits $MMR$, your position is liquidated.

---

## Liquidation price

The liquidation price is the mark price at which your margin ratio exactly equals MMR. To find it, substitute the liquidation condition $MR = MMR$ and solve for $P_{mark}$.

### Long position

For a long, $uPnL = (P_{mark} - P_{entry}) \times Q$. Setting $MR = MMR$:

$\frac{M + (P_{liq} - P_{entry}) \times Q}{Q \times P_{liq}} = MMR$

Solving for $P_{liq}$:

$M + Q \times P_{liq} - Q \times P_{entry} = Q \times P_{liq} \times MMR$

$M - Q \times P_{entry} = Q \times P_{liq} \times (MMR - 1)$

$\boxed{P_{liq}^{long} = \frac{Q \times P_{entry} - M}{Q \times (1 - MMR)}}$

Since initial margin $M = Q \times P_{entry} / L$, this simplifies to:

$P_{liq}^{long} = P_{entry} \times \frac{1 - \frac{1}{L}}{1 - MMR}$

### Short position

For a short, $uPnL = (P_{entry} - P_{mark}) \times Q$. Setting $MR = MMR$:

$\frac{M + (P_{entry} - P_{liq}) \times Q}{Q \times P_{liq}} = MMR$

Solving for $P_{liq}$:

$M + Q \times P_{entry} - Q \times P_{liq} = Q \times P_{liq} \times MMR$

$M + Q \times P_{entry} = Q \times P_{liq} \times (1 + MMR)$

$\boxed{P_{liq}^{short} = \frac{M + Q \times P_{entry}}{Q \times (1 + MMR)}}$

Substituting $M = Q \times P_{entry} / L$:

$P_{liq}^{short} = P_{entry} \times \frac{1 + \frac{1}{L}}{1 + MMR}$

---

## Worked examples

### Single leverage – basic example

Suppose you open a **long** on an asset at \$100 with 10x leverage and MMR = 2.5%, with a liquidation price of \$92.31.

If the mark price drops to \$92.31, your position is liquidated.

For the equivalent **short** at the same parameters, if the mark price rises to \$107.32, the short is liquidated.

---

## The liquidation process

When the mark price reaches your liquidation price:

1. **Liquidation is triggered.** The liquidation engine takes over your position immediately.
2. **Position is closed at best available price.** The engine attempts to close your position at or near the current market price. Because TrueCurrent uses a competitive liquidity network, liquidations typically execute close to the liquidation price.
3. **Remaining margin is forfeited (if any).** When the position is liquidated, the surplus is split between the liquidator and insurance fund.
4. **Insurance fund absorbs shortfalls.** If the position closes at a worse price than the liquidation price (negative equity), the insurance fund covers the difference. You do not owe beyond your deposited margin.

---

## Insurance fund

<Note>
  TrueCurrent maintains an insurance fund for each market to protect the system against insolvent liquidations. You never owe beyond your deposited margin – the fund covers any shortfall.
</Note>

**Purpose:** The insurance fund ensures that winning traders are always paid in full, even when a counterparty is liquidated at a loss beyond their margin.

**Accrual:** The fund grows whenever a liquidation closes between $P_{liq}$ and $P_{bankrupt}$. The margin buffer captured in that price range (the difference between the position's equity at $P_{liq}$ and zero) is transferred to the insurance fund, not returned to the trader.

**Depletion:** If the insurance fund is exhausted – meaning a catastrophic liquidation event exceeds its reserves – the system activates Auto-Deleveraging (ADL) as a last resort. See the [ADL section](#auto-deleveraging-adl) below.

The onchain insurance fund address can be queried on the Injective explorer for real-time balance transparency. Refer to the [Injective liquidation documentation](https://docs.injective.network/defi/trading/margin-liquidation) for the per-market insurance fund details.

---

## Auto-deleveraging (ADL)

If a liquidation closes at a price worse than the bankruptcy price _and_ the insurance fund cannot cover the shortfall, TrueCurrent activates auto-deleveraging as a last resort. ADL closes a portion of the most profitable opposite-side positions at the insolvent counterparty's bankruptcy price to absorb the deficit.

ADL is rare – it only fires when the insurance fund has been depleted by an extreme event. But if you carry leveraged exposure during a tail-risk move, it is the final backstop you can be touched by even though your own margin is healthy.

For when ADL activates, how the queue is ranked, the exit price you receive, and how to reduce your exposure, see [Auto-deleveraging (ADL)](/trading/adl).

---

## Avoiding liquidation

**Monitor your margin ratio.** The Positions panel shows your margin ratio and liquidation price in real time. The closer your mark price is to your liquidation price, the more urgent the situation.

**Add margin.** You can deposit additional USDC to an open position at any time to increase equity and push your liquidation price further away.

**Reduce leverage.** Partially closing a position reduces your notional exposure while keeping the same margin, effectively deleveraging.

**Use lower leverage.** The relationship is direct: at 20× leverage, a 5% adverse move approaches liquidation. At 2× leverage, you'd need a 50% move. Sizing leverage to your risk tolerance is the most effective protection.

**Watch funding rates.** Persistent funding payments drain margin over time. A position that's safe today may be closer to liquidation tomorrow if funding is running against you. See [Funding rates](/trading/funding-rates).