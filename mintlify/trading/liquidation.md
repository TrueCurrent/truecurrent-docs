---
title: "Liquidation"
updatedAt: "2026-04-08"
---

When a position's margin ratio falls to or below the maintenance margin rate (MMR), TrueCurrent's liquidation engine automatically closes it. This page explains exactly when and how that happens, how to calculate your liquidation price, and how to protect yourself.

---

## What triggers liquidation

Liquidation is triggered when your margin ratio falls at or below the maintenance margin rate:

```
MR = (M + uPnL) / (Q × P_mark) ≤ MMR
```

As the mark price moves against your position, `uPnL` decreases, which decreases `MR`. When `MR` hits `MMR`, your position is liquidated.

---

## Liquidation price

The liquidation price is the mark price at which your margin ratio exactly equals MMR. To find it, set `MR = MMR` and solve for `P_mark`.

### Long position

For a long, `uPnL = (P_mark − P_entry) × Q`. Setting `MR = MMR` and solving for `P_liq`:

```
P_liq_long = (Q × P_entry − M) / (Q × (1 − MMR))
```

Since initial margin `M = Q × P_entry / L`, this simplifies to:

```
P_liq_long = P_entry × (1 − 1/L) / (1 − MMR)
```

### Short position

For a short, `uPnL = (P_entry − P_mark) × Q`. Setting `MR = MMR` and solving for `P_liq`:

```
P_liq_short = (M + Q × P_entry) / (Q × (1 + MMR))
```

Substituting `M = Q × P_entry / L`:

```
P_liq_short = P_entry × (1 + 1/L) / (1 + MMR)
```

---

## Worked example

Suppose you open a **long** at `P_entry = $100` with `L = 10×` leverage and `MMR = 2.5%`:

```
P_liq = 100 × (1 − 1/10) / (1 − 0.025)
      = 100 × 0.9 / 0.975
      ≈ $92.31
```

If the mark price drops to $92.31, your position is liquidated.

For the equivalent **short** at the same parameters:

```
P_liq = 100 × (1 + 1/10) / (1 + 0.025)
      = 100 × 1.1 / 1.025
      ≈ $107.32
```

If the mark price rises to $107.32, the short is liquidated.

---

## Bankruptcy price

The bankruptcy price is where unrealized losses fully consume your margin – equity reaches zero. This is worse than the liquidation price and represents the point at which the insurance fund may need to intervene.

**Long:**

```
P_bankrupt_long = P_entry × (1 − 1/L)
```

**Short:**

```
P_bankrupt_short = P_entry × (1 + 1/L)
```

The gap between `P_liq` and `P_bankrupt` is where the liquidation engine operates. Positions are closed at `P_liq`, and any remaining margin above zero is partially returned. If the position cannot be closed before the bankruptcy price is reached (e.g., during extreme volatility), the insurance fund covers the shortfall.

---

## The liquidation process

When the mark price reaches your liquidation price:

1. **Liquidation is triggered.** The liquidation engine takes over your position immediately.
2. **Position is closed at best available price.** The engine attempts to close your position at or near the current market price. Because TrueCurrent uses a competitive liquidity network, liquidations typically execute close to the liquidation price.
3. **Remaining margin is returned (if any).** If the position closes at a better price than the bankruptcy price, the surplus is returned to your account.
4. **Insurance fund absorbs shortfalls.** If the position closes at a worse price than the bankruptcy price (negative equity), the insurance fund covers the difference. You do not owe beyond your deposited margin.

---

## Insurance fund

TrueCurrent maintains an insurance fund to cover situations where liquidated positions cannot be closed before reaching the bankruptcy price. This ensures that winning traders are always paid in full, even when counterparties are liquidated at a loss.

The insurance fund grows from a portion of liquidation proceeds when positions are closed between `P_liq` and `P_bankrupt`.

---

## Avoiding liquidation

**Monitor your margin ratio.** The Positions panel shows your margin ratio and liquidation price in real time. The closer your mark price is to your liquidation price, the more urgent the situation.

**Add margin.** You can deposit additional USDC to an open position at any time to increase equity and push your liquidation price further away.

**Reduce leverage.** Partially closing a position reduces your notional exposure while keeping the same margin, effectively deleveraging.

**Use lower leverage.** The relationship is direct: at 50× leverage, a 2% adverse move approaches liquidation. At 2× leverage, you'd need a 50% move. Sizing leverage to your risk tolerance is the most effective protection.

**Watch funding rates.** Persistent funding payments drain margin over time. A position that's safe today may be closer to liquidation tomorrow if funding is running against you. See [Funding rates](/trading/funding-rates).
