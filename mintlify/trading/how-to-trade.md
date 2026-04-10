---
title: "How to trade"
description: "Complete trading guide for TrueCurrent perpetuals covering position opening, long and short strategies, leverage selection, margin management, P&L calculation, and liquidation avoidance techniques."
updatedAt: "2026-04-06"
---

A complete guide to opening, managing, and closing positions on TrueCurrent.

---

## Opening a position

### 1. Select a market

Choose the perpetual market you want to trade from the markets list. You'll see the current price, 24h volume, open interest, and funding rate for each market.

<div class="image-placeholder">
  <img src="/img/markets-list.png" alt="Markets list" />
  <p><em>Available markets on TrueCurrent</em></p>
</div>

### 2. Choose long or short

**Long** – you profit when the price rises above your entry.

**Short** – you profit when the price falls below your entry.

### 3. Set your margin and leverage

**Margin** is the USDC collateral you're putting up. **Leverage** multiplies your exposure.

```
Position size = Margin × Leverage
```

A $100 margin at 5x leverage = $500 position. Every 1% price move = 5% change in your margin.

**Leverage guidelines:**
- 1x–3x – Conservative, good for learning
- 5x–10x – Moderate, for experienced traders
- 10x–20x – High risk, requires active monitoring

### 4. Set your price tolerance

Your **price tolerance** determines how far from the current price you're willing to fill. A wider tolerance means a higher chance of filling; a tighter tolerance gives you more control.

For most market conditions, 0.5–1% works well. In volatile markets, set it slightly wider.

### 5. Confirm

Click **Trade**. TrueCurrent automatically collects prices from institutional liquidity providers, picks the best one, and executes – all in under a second. You'll see an estimated price and can review your worst-case price before confirming; the trade will never execute at a less favorable price than your limit.

<div class="image-placeholder">
  <img src="/img/trade-confirm.png" alt="Confirming a trade" />
  <p><em>Confirming your trade</em></p>
</div>

Your position opens in under one second.

---

## Managing open positions

Your positions appear in the **Positions** panel. For each position you can see:

- **Entry price** – the price you traded at
- **Mark price** – the current fair value price (used for P&L and liquidation)
- **Unrealized P&L** – your current profit or loss
- **Margin** and **available margin**
- **Liquidation price** – the price at which your position is automatically closed

<div class="image-placeholder">
  <img src="/img/position-details.png" alt="Position management panel" />
  <p><em>The positions panel with key metrics</em></p>
</div>

### Adding margin

If your position is approaching the liquidation price, you can add margin to push it further away. Go to your position details and click **Add Margin**.

### Partial close

You can reduce your position size without closing entirely. Click **Partial Close**, enter the quantity to close, and confirm. The corresponding margin is released.

---

## Closing a position

Click **Close** on any open position. Closing works the same as opening – TrueCurrent finds the best available price and executes automatically. You can close fully or partially.

Closed positions settle instantly and released margin returns to your available balance.

---

## Understanding your P&L

Your unrealized P&L is calculated as:

**For longs:**
```
P&L = (Mark price − Entry price) × Position size
```

**For shorts:**
```
P&L = (Entry price − Mark price) × Position size
```

When you close, P&L is realized and added to (or subtracted from) your subaccount balance.

Note: funding payments are also applied to your margin over time. See [Funding rates](/trading/funding-rates).

---

## Liquidation

If the mark price reaches your **liquidation price**, your position is automatically closed by the liquidation system:

- Your remaining margin is used to settle the position
- If there's a shortfall, the insurance fund covers it
- Any margin above the liquidation cost is returned to your account

To avoid liquidation: use lower leverage, add margin, or close the position before it becomes critical.

See [Perpetual markets](/trading/perpetual-markets) for details on how margin and liquidation work.
