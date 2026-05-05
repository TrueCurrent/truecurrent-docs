---
title: "Available markets"
description: "How perpetual futures markets are listed on TrueCurrent and how to find the live list of supported markets and their parameters."
updatedAt: "2026-05-05"
---

TrueCurrent offers perpetual futures markets on Injective. Each market is collateralized in USDC and uses Injective's onchain oracle for mark and index prices.

The live list of supported markets, current parameters (tick size, max leverage, funding cadence), and 24h volume is shown in the trading interface. For programmatic access, query the Injective exchange module's `DerivativeMarket` endpoint — that is the onchain source of truth.

---

## How new markets are listed

New markets on TrueCurrent require at least one registered market maker willing to provide liquidity. The listing process is:

1. The community or the team identifies demand for a new market
2. At least one market maker commits to providing RFQ liquidity
3. The market is deployed via Injective governance (this sets tick size, lot size, IMR, MMR, funding cap, and max open interest)
4. The market is enabled in the TrueCurrent UI

If you are a market maker interested in supporting a new market, see [Getting whitelisted](/market-makers/getting-whitelisted).

---

## Mark price and oracle

Each market's mark price is derived from Injective's onchain oracle, which aggregates prices from multiple credible sources. The mark price is used for unrealized P&L, liquidations, funding-rate calculations, and TP/SL trigger evaluation. Oracle prices are public and verifiable onchain.

For the full mark/index/quoted-price model, see [Index, mark, and quoted prices](/trading/prices).
