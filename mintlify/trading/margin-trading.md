---
title: "Margin trading"
description: "Master margin trading on TrueCurrent with cross-margin mechanics, initial and maintenance margin ratios, margin ratio calculations, available margin, and liquidation threshold management."
updatedAt: "2026-04-30"
---

Margin trading lets you control a position larger than your deposited collateral by using leverage. This page explains how margin is calculated, how your account equity is tracked, and how to read the key metrics in your positions panel.

---

<Warning>
**TrueCurrent uses cross-margin only.** There is no isolated margin mode. All open positions in your subaccount share a single equity pool. Profits on one position directly offset losses on another — and a large losing position can draw on the margin of your other positions.
</Warning>

**Only USDC is accepted as collateral.** There is no multi-asset margin or cross-collateral support currently. You must hold USDC in your subaccount to open and maintain positions.

---

## Cross-margin vs. isolated margin

Traders arriving from other platforms that offer isolated margin modes will notice TrueCurrent works differently.

| Feature | Cross-margin (TrueCurrent) | Isolated margin (other platforms) |
|---------|---------------------------|----------------------------------|
| Collateral pool | Shared across all positions in subaccount | Separate per position |
| P&L netting | Profits on position A offset losses on position B | No netting; each position is independent |
| Liquidation risk | One large loser can liquidate other positions by draining shared equity | Loss limited to the margin allocated to that specific position |
| Capital efficiency | Higher — idle margin in profitable positions is available for other trades | Lower — each position requires its own dedicated margin |
| Max loss per position | Can exceed initial margin of that position (via equity from other positions) | Capped at the isolated margin allocated |
| Recommended for | Hedged portfolios; traders managing multiple correlated positions | Speculative positions where you want strict risk isolation |

**TrueCurrent's cross-margin model is more capital-efficient** — you don't need to pre-allocate separate margin buckets for each position. However, it requires more active overall portfolio monitoring, since a single bad position affects your entire subaccount's health.

If you want isolated-margin-like behavior, you can achieve it by using **separate subaccounts** — each subaccount has its own independent equity pool.

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

### Margin tier table

Margin rates may increase for very large notional positions as a risk management measure. The table below shows indicative tiers; exact values are set per market and may be updated via governance.

| Notional position size | Max leverage | IMR | MMR |
|------------------------|-------------|-----|-----|
| Up to $50,000 | 20× | 5.0% | 2.5% |
| $50,001 – $250,000 | 10× | 10.0% | 5.0% |
| $250,001 – $1,000,000 | 5× | 20.0% | 10.0% |
| Above $1,000,000 | 2× | 50.0% | 25.0% |

<Note>
These are illustrative tiers. Confirm actual parameters for each market via the market specifications or the Injective exchange module before sizing large positions.
</Note>

If your position grows into a higher notional tier (e.g. due to mark price appreciation), you may be required to post additional initial margin to open further positions in that market.

---

## Worked example: position lifecycle with cross-margin

This example traces equity, margin ratio, and available margin through four steps for a single-position account. Assumptions: 10× leverage, $P_{entry} = \$100$, MMR = 2.5%, position size Q = 10 units.

**Step 1 — Open position**

- Initial margin deposited: $100 (= 10 × $100 / 10)
- Notional: $1,000
- uPnL: $0
- Account equity: $100
- Margin ratio: $100 / $1,000 = **10.0%**
- Available margin: $100 − $1,000 × 2.5% = $100 − $25 = **$75**

**Step 2 — Price moves against position (mark price = $95)**

- uPnL: (95 − 100) × 10 = −$50
- Account equity: $100 − $50 = **$50**
- Margin ratio: $50 / (10 × $95) = $50 / $950 ≈ **5.26%** (above MMR 2.5%, safe)
- Available margin: $50 − $950 × 2.5% = $50 − $23.75 = **$26.25**

**Step 3 — Add $30 margin**

- Account equity: $50 + $30 = **$80**
- Margin ratio: $80 / $950 ≈ **8.42%** (comfortable buffer)
- Available margin: $80 − $23.75 = **$56.25**

**Step 4 — Close half the position (sell 5 units at $95)**

- Realised P&L on 5 units: (95 − 100) × 5 = −$25 realised
- Remaining position: 5 units
- Notional: 5 × $95 = $475
- Remaining equity: $80 − $25 = **$55**
- Margin ratio: $55 / $475 ≈ **11.6%** (strong; leverage effectively reduced to ~4.3×)

---

### Cross-margin with two simultaneous positions

This example shows how profits and losses net across positions in a shared equity pool.

Setup: Account equity = $200, two positions open simultaneously.

- **Position A:** 10 INJ long at $25, 10× leverage. Notional = $250, margin used = $25.
- **Position B:** 5 ETH long at $3,000, 10× leverage. Notional = $15,000, margin used = $1,500.

*(Note: the above is illustrative; your actual available margin must cover total required margin across all open positions.)*

For a simpler cross-margin example: account has $200 USDC, two positions each requiring $100 margin.

- **Position A:** 4 units long at $25 (10×). Required margin: $100.
- **Position B:** 4 units short at $25 (10×) on a different market. Required margin: $100.

**After price moves to $30 on Position A's market, $20 on Position B's market:**

- Position A uPnL: (30 − 25) × 4 = **+$20**
- Position B uPnL: (25 − 20) × 4 = **+$20** (short profits when price falls)
- Total uPnL: +$40
- Account equity: $200 + $40 = **$240**

Both positions benefit. Shared equity grows.

**After price moves to $30 on A's market, $32 on B's market (adverse for short):**

- Position A uPnL: +$20
- Position B uPnL: (25 − 32) × 4 = **−$28**
- Total uPnL: −$8
- Account equity: $200 − $8 = **$192**

Position A's profit ($20) partially offsets Position B's loss ($28). Equity stays healthy despite one losing position. This is the key benefit of cross-margin: **profits in one position directly protect the rest of the portfolio.**

---

## Managing your margin

**Add margin** to a position at any time to increase your equity and push your liquidation price further away. Go to your position in the Positions panel and click **Add Margin**.

**Reduce leverage** by partially closing a position, which releases margin and reduces your notional exposure.

**Monitor the margin ratio** – if it approaches MMR, act proactively rather than waiting for liquidation.

**Use separate subaccounts** if you want isolated-margin-like behavior — each subaccount maintains its own independent equity pool.

See [Liquidation](/trading/liquidation) for exactly how the liquidation price is calculated.
