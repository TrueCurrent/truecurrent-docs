---
title: "Available markets"
description: "How perpetual futures markets are listed on TrueCurrent and how to find the live list of supported markets and their parameters."
updatedAt: "2026-05-06"
---

TrueCurrent offers perpetual futures markets on Injective. Each market is collateralized in USDC and has an onchain mark price used for P&L, liquidations, funding, and trigger-order evaluation.

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

## Mark price

Each market exposes an onchain `mark_price` through Injective. TrueCurrent uses that mark price for unrealized P&L, liquidations, funding-rate calculations, TP/SL trigger evaluation, and quote validation. It is public and verifiable onchain.

For the full mark and quoted-price model, see [Mark and quoted prices](/trading/prices).
