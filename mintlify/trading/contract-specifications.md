---
title: "Contract parameters"
description: "How TrueCurrent perpetual-market parameters are set, where to query the live onchain values, and how parameters change over time via Injective governance."
updatedAt: "2026-05-05"
---

This page explains the quantitative parameters that govern trading on TrueCurrent. These are inputs for quoting models, risk systems, and position-sizing calculations.

<Warning>
Use the onchain `DerivativeMarket` response as the source of truth for production systems. Market parameters are set per-market and can change through Injective governance.
</Warning>

---

## What governs a market

Every TrueCurrent perpetual market is an Injective derivative market. Its parameters live onchain and include:

| Parameter | What it controls |
|---|---|
| **Tick size** | Smallest price increment a quote can use |
| **Lot size** | Smallest quantity increment per order |
| **Min notional order size** | Smallest dollar-equivalent order the market will accept |
| **Initial margin rate (IMR)** | Minimum margin to *open* a position; equal to `1 / max_leverage` |
| **Maintenance margin rate (MMR)** | Minimum margin to *keep* a position open; below this, liquidation triggers |
| **Funding rate cap** | Per-market clamp on the hourly funding rate |
| **Max open interest** | Cap on aggregate notional in the market |
| **Settlement currency** | The quote-side asset used for margin and settlement (USDC for current markets) |
| **Oracle source** | The Injective oracle feed used to derive mark price |

To get the live values for a market, query Injective's exchange module `DerivativeMarket` endpoint. Do not hard-code values in long-lived clients — re-read at startup, and refresh on governance events that touch your market.

---

## How parameters are set

Contract parameters for an Injective derivative market are set when the market is first created via an Injective governance proposal. Subsequent changes also require a governance vote.

The process:

1. A governance proposal is submitted on the Injective chain specifying the parameter values
2. INJ stakers vote to approve or reject the proposal
3. If approved, the parameter update takes effect at the governance execution block

This means contract parameters can change over time. Always verify current values from the onchain source if you are building a quoting or risk system that depends on them.

---

## Related pages

- [Available markets](/trading/markets) — listing process and oracle source
- [Margin trading](/trading/margin-trading) — how IMR and MMR are used in margin calculations
- [Liquidation](/trading/liquidation) — how the liquidation price is calculated from MMR
- [Auto-deleveraging (ADL)](/trading/adl) — what happens when the insurance fund cannot cover a liquidation
