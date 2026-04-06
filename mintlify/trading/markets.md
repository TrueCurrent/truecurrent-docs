---
title: "Available markets"
description: "--"
updatedAt: "2026-04-06"
---

TrueCurrent currently offers perpetual futures markets on Injective. New markets are added based on demand and liquidity availability.

---

## Active markets

| Market | Collateral | Max leverage | Min order size | Funding interval |
|--------|-----------|-------------|----------------|-----------------|
| INJ/USDC PERP | USDC | 20x | TBD | Hourly |

*Additional markets coming soon – check the app for the latest listing.*

---

## Market details

### INJ/USDC PERP

The Injective native token perpetual is TrueCurrent's flagship market. INJ is the utility and governance token of the Injective network, used for gas, staking, and governance.

- **Oracle:** Injective onchain oracle (aggregated from multiple sources)
- **Settlement:** USDC
- **Funding:** Hourly

---

## How new markets are listed

New markets on TrueCurrent require at least one registered market maker willing to provide liquidity for that market. The listing process is:

1. Community or team identifies demand for a new market
2. At least one market maker commits to providing RFQ liquidity
3. A new market is deployed to the TrueCurrent contract
4. The market is made available in the UI

If you're a market maker interested in supporting a new market, see [Getting whitelisted](/market-makers/getting-whitelisted).

---

## Mark price and oracle

Each market's mark price is derived from Injective's onchain oracle. The oracle is designed to be tamper-resistant and aggregates prices from multiple credible sources. This mark price is used for:

- Unrealized P&L calculations
- Liquidation checks
- Funding rate calculations

You can view Injective oracle prices directly onchain for full transparency.
