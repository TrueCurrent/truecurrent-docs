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

## Margin tiers

Perpetual markets use a tiered margin structure: as your notional position size increases, the required margin rate rises. This protects the insurance fund by ensuring larger positions carry proportionally more equity as a buffer.

The exact live tiers must be read from the onchain market/config response. The table below is an example of how tiered parameters are interpreted; do not treat it as a launch schedule unless the same values are present onchain.

| Notional (USDC) | Initial margin rate | Maintenance margin rate | Max leverage |
|---|---|---|---|
| 0 – 100,000 | 5.0% | 2.5% | 20× |
| 100,001 – 500,000 | 10.0% | 5.0% | 10× |
| 500,001 – 2,000,000 | 20.0% | 10.0% | 5× |
| 2,000,001+ | 50.0% | 25.0% | 2× |

<Note>
The onchain source of truth is the Injective exchange module's `DerivativeMarket` response for this market. Confirm current values before opening a position or deploying an automated quoting system.
</Note>

---

## Worked example — large position increases IMR

Margin tiers are not always intuitive. Here is a concrete example, using the illustrative tiers above, showing how scaling into a larger position increases the initial margin requirement.

**Scenario:** You want to open a long position on INJ/USDC PERP when INJ is priced at $10.

**Trade A — $50,000 notional (within Tier 1):**

$$Q = 5{,}000\ \text{INJ},\quad P_{entry} = \$10,\quad \text{IMR} = 5\%$$

$$IM = Q \times P_{entry} \times IMR = 5{,}000 \times 10 \times 0.05 = \$2{,}500$$

**Trade B — $600,000 notional (crosses into Tier 3):**

$$Q = 60{,}000\ \text{INJ},\quad P_{entry} = \$10,\quad \text{IMR} = 20\%$$

$$IM = 60{,}000 \times 10 \times 0.20 = \$120{,}000$$

Effective leverage on Trade B is now 5× maximum, not 20×. If you try to open this position with only $30,000 in margin (the 10× rate you might expect from a smaller trade), the position will be rejected.

**Key rule:** the IMR that applies is determined by the total notional of the position you are opening, including any existing notional in the same market on the same subaccount.

---

## Why margin tiers exist

The insurance fund is the buffer between a liquidated position and ADL. Larger positions create larger potential shortfalls if they are liquidated at an adverse price. Requiring more equity per dollar of notional exposure at higher sizes:

1. Reduces the probability that a large liquidation results in a shortfall the fund cannot cover
2. Keeps the fund solvent for more extreme but smaller events
3. Limits the aggregate ADL exposure for all other traders

Tiers also naturally limit very high leverage to smaller notional sizes where the risk is more easily absorbed.

See [Auto-deleveraging (ADL)](/trading/adl) for how the insurance fund and ADL interact.

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
