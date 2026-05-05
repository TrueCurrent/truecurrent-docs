---
title: "Contract parameters"
description: "Reference for TrueCurrent perpetual-market parameters: where to query the live onchain values, which public testnet values are currently confirmed, and how margin parameters affect position sizing."
updatedAt: "2026-05-01"
---

This page explains the quantitative parameters that govern trading on TrueCurrent. These are inputs for quoting models, risk systems, and position-sizing calculations.

<Warning>
Use the onchain `DerivativeMarket` response as the source of truth for production systems. This page can explain the fields and list confirmed public values, but market parameters can change through governance and deployment updates.
</Warning>

---

## INJ/USDC PERP — market parameters

| Parameter | Value |
|---|---|
| **Settlement currency** | USDC |
| **Max leverage** | 20× |
| **Maintenance margin rate (MMR)** | 2.5% |
| **Initial margin rate (IMR) at max leverage** | 5.0% |
| **Funding interval** | Hourly |
| **Oracle source** | Injective onchain oracle |
| **Tick size** | `0.01` on the current INJ/USDC PERP testnet market |
| **Lot size (min quantity per order)** | TBD |
| **Min notional order size** | TBD |
| **Max open interest** | TBD |
| **Funding rate cap** | TBD |

<Note>
TBD values will be confirmed and published here once finalized. In the meantime, refer to the app UI for current minimum order sizes. All values can also be queried from the Injective exchange module via the `DerivativeMarket` endpoint.
</Note>

---

## How parameters are set

Contract parameters for Injective derivative markets — including tick size, lot size, initial margin rate, maintenance margin rate, funding cap, and max open interest — are set when the market is first created via an Injective governance proposal. Any subsequent changes require a new governance vote.

The process:

1. A governance proposal is submitted on the Injective chain specifying the updated parameter values
2. INJ stakers vote to approve or reject the proposal
3. If approved, the parameter update takes effect at the governance execution block

This means contract parameters can change over time. Always verify the current values from the onchain source (`DerivativeMarket` query on the Injective exchange module) if you are building a quoting or risk system that depends on them.

---

## Related pages

- [Available markets](/trading/markets) — market listing overview and oracle source
- [Margin trading](/trading/margin-trading) — how IMR and MMR are used in margin calculations, cross-margin mechanics
- [Liquidation](/trading/liquidation) — how the liquidation price is calculated from MMR, and the role of the insurance fund
- [Auto-deleveraging (ADL)](/trading/adl) — what happens when the insurance fund is insufficient to cover a liquidation
